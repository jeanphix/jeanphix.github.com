---
layout: post
lang: en
title: "Application time series with postgres materialized views"
tags:
- data
- postgresql
- sql
- bi
---

When building an application that manages a huge amount of data, it's often a requirement to provide to the users a way to visualize how data grow over the time.

Let's consider we are making a payment processing app for merchants with the following database schema:

{% highlight sql %}
create table merchant (
    id serial primary key,
    name char(50) not null unique
);


create table transaction (
    id bigserial primary key,
    amount decimal(10,2) not null,
    created_at timestamp with time zone default now() not null,
    merchant_id integer references merchant(id) not null
);
{% endhighlight %}

With few test data:

{% highlight sql %}
/* Make 1000 merchants */
insert into merchant(name)
select concat('merchant ', generate_series(1, 1000));

/* Make 100,000,000 transactions bound to the merchants */
insert into transaction(amount, created_at, merchant_id)
select amount, created_at, merchant_id from (
    select
        generate_series(1, 100 * 1000 * 1000),
        (random() * 100)::decimal(6, 2) as amount,
        now() - concat((random() * 500)::int, ' days')::interval as created_at,
        trunc(random() * 1000 + 1) as merchant_id
) as data;
{% endhighlight %}

On the merchant dashboard, we want to display a wonderful chart showing them the total amount of transaction per month for the whole year.

The natural way to extract that kind of data is to consider aggregating over the normalized application schema:

{% highlight sql %}
select
    date_trunc('month', created_at::date)  as month,
    sum(amount) as amount
from transaction
where created_at
    between
        date_trunc('month', now()::date) - '1 years'::interval
    and
        date_trunc('month', now()::date) + '1 months'::interval
and
    merchant_id = 256
group by month
order by month asc;
{% endhighlight %}

That results in something like:

{% highlight sql %}
         month          |  amount
------------------------+----------
 2014-07-01 00:00:00+00 | 32964.08
 2014-08-01 00:00:00+00 | 33424.42
 2014-09-01 00:00:00+00 | 30665.18
 2014-10-01 00:00:00+00 | 31402.36
 2014-11-01 00:00:00+00 | 27657.13
 2014-12-01 00:00:00+00 | 32035.49
 2015-01-01 00:00:00+00 | 31310.66
 2015-02-01 00:00:00+00 | 27283.83
 2015-03-01 00:00:00+00 | 32315.47
 2015-04-01 00:00:00+00 | 32318.75
 2015-05-01 00:00:00+00 | 32122.84
 2015-06-01 00:00:00+00 | 27699.18
 2015-07-01 00:00:00+00 | 25544.77
{% endhighlight %}

But the query took *1068.265* ms to execute, which is really too slow for a web app...

## Store a time serie within a materialized view:

As in our dashboard we also want to allow the user to zoom in a specific month and a specific day, we will then consider that the unit serie will be: total amount / merchant / day.

So the idea here is to create a (materialized) view to store our time serie:

{% highlight bash %}
create materialized view daily_merchant_transaction_amount as select
    merchant_period.merchant_id as merchant_id,
    merchant_period.day::date as day,
    sum(transaction.amount) as amount

from (
    /* Make merchant, day pairs */
    select u.id as merchant_id, period.day as day
    from (select id from merchant) u, (
        /* List all days */
        select distinct(
            generate_series(
                date_trunc('month', min(created_at)::date),
                date_trunc('month', max(created_at)::date)
                    + '1 MONTH'::interval - '1 day'::interval,
                '1 days'
            )
        ) as day
        from transaction
    ) period
    where day <= now()
) merchant_period
left join transaction
    on transaction.created_at::date = merchant_period.day
    and transaction.merchant_id = merchant_period.merchant_id
group by merchant_period.day, merchant_period.merchant_id
order by merchant_period.day desc;
{% endhighlight %}

And a unique index on it:

{% highlight sql %}
create
    unique index ix_daily_merchant_transaction_amount__merchant_id__day
    on daily_merchant_transaction_amount(merchant_id, day desc);
{% endhighlight %}


Now, retrieving our per month data for a whole year (with total) is as simple as:

{% highlight sql %}
with data as (
    select date_trunc('month', day) as month,
    sum(amount) as amount
    from daily_merchant_transaction_amount
    where
        merchant_id = 256
    and
        day >= date_trunc('month', now()::date) - '1 year'::interval
    group by month
    order by month asc
)
select * from data
union all
select null, sum(amount) from data;
{% endhighlight %}

That results in:

{% highlight sql %}
         month          |  amount
------------------------+-----------
 2014-07-01 00:00:00+00 |  32964.08
 2014-08-01 00:00:00+00 |  33424.42
 2014-09-01 00:00:00+00 |  30665.18
 2014-10-01 00:00:00+00 |  31402.36
 2014-11-01 00:00:00+00 |  27657.13
 2014-12-01 00:00:00+00 |  32035.49
 2015-01-01 00:00:00+00 |  31310.66
 2015-02-01 00:00:00+00 |  27283.83
 2015-03-01 00:00:00+00 |  32315.47
 2015-04-01 00:00:00+00 |  32318.75
 2015-05-01 00:00:00+00 |  32122.84
 2015-06-01 00:00:00+00 |  27699.18
 2015-07-01 00:00:00+00 |  25544.77
                        | 396744.16
(14 rows)
{% endhighlight %}

within *1.906 ms*!

Or, for example, to query transaction amount per day for last month with total:

{% highlight sql %}
with data as (
    select day, amount
    from daily_merchant_transaction_amount
    where merchant_id = 256 and day > now()::date - '1 month'::interval
    order by day asc
)
select * from data
union all
select null, sum(amount) from data;
{% endhighlight %}

That returns:

{% highlight sql %}
    day     |  amount
------------+----------
 2015-06-27 |  1101.01
 2015-06-28 |  1614.27
 2015-06-29 |   966.23
 2015-06-30 |  1202.67
 2015-07-01 |   848.47
 2015-07-02 |  1074.99
 2015-07-03 |   776.63
 2015-07-04 |   878.01
 2015-07-05 |  1055.64
 2015-07-06 |  1253.17
 2015-07-07 |  1003.11
 2015-07-08 |   905.04
 2015-07-09 |   901.39
 2015-07-10 |  1126.45
 2015-07-11 |  1398.06
 2015-07-12 |  1045.39
 2015-07-13 |   750.14
 2015-07-14 |  1069.99
 2015-07-15 |   941.57
 2015-07-16 |  1189.76
 2015-07-17 |  1055.53
 2015-07-18 |  1185.39
 2015-07-19 |  1017.86
 2015-07-20 |   942.38
 2015-07-21 |   628.57
 2015-07-22 |   945.19
 2015-07-23 |   457.89
 2015-07-24 |  1152.21
 2015-07-25 |  1113.52
 2015-07-26 |   828.42
            | 30428.95
(31 rows)
{% endhighlight %}

within *0.113 ms*.

## Refresh strategy

Unfortunately, postgresql still doesn't provide a way to partially refresh a view, but 9.4 introduced a *concurrently* parameter that allows to run a non blocking *refesh*:

{% highlight sql %}
refresh materialized view concurrently daily_merchant_transaction_amount;
{% endhighlight %}

## Scale up considerations

* put the views in physical volumes separated from the application tables.
* query the views from dedicated slave nodes.
* make per time granularity views, e.g.: per hour, per day, per month.
* sharding, e.g.: per merchant.

Cheers,
