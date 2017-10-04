---
layout: post
lang: en
title: How to configure ECS scheduled (cron) task using cloudformation
description: How to configure ECS scheduled (cron) task using cloudformation
tags:
- AWS
- ECS
- CloudFormation
- Scheduled task
- cron
---

*AWS CloudFormation* now allows to define scheduled tasks to be run within
*ECS* clusters. It's pretty straight forward to setup and ensure the task is
properly placed (probably run just once in most cases) within a cluster.

First of all, we need to define the task
([AWS::ECS::TaskDefinition](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html)):

{% highlight python %}
scheduled_worker_task_definition = TaskDefinition(
    "ScheduledWorkerTask",
    template=template,
    ContainerDefinitions=[
        ContainerDefinition(
            Name="ScheduledWorker",
            Cpu=200,
            Memory=300,
            Essential=True,
            Image="<image>",
            EntryPoint=['<entry_point>']
        ),
    ],
)
{% endhighlight %}

Then we need to create a role
([AWS::IAM::Role](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html))
with proper policies to run a task within our cluster:

{% highlight python %}
run_task_role = iam.Role(
    "RunTaskRole",
    template=template,
    AssumeRolePolicyDocument=dict(Statement=[dict(
        Effect="Allow",
        Principal=dict(Service=["events.amazonaws.com"]),
        Action=["sts:AssumeRole"],
    )]),
    Path="/",
    Policies=[
        iam.Policy(
            PolicyName="RunTaskPolicy",
            PolicyDocument=dict(
                Statement=[dict(
                    Effect="Allow",
                    Action=[
                        "ecs:RunTask",
                    ],
                    Resource=["*"],
                    Condition=dict(
                        ArnEquals={
                            "ecs:cluster": GetAtt(cluster, "Arn"),
                        }
                    )
                )],
            ),
        ),
    ],
)
{% endhighlight %}

And finally an event rule
([AWS::Events::Rule](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-events-rule.html))
that is responsible of the scheduling:

{% highlight python %}
Rule(
    "SchedulingRule",
    template=template,
    Description="My schedule task rule",
    State="ENABLED",
    ScheduleExpression="rate(30 minutes)",
    Targets=[
        Target(
            Id="ScheduledWorkerTaskDefinitionTarget",
            RoleArn=GetAtt(run_task_role, "Arn"),
            Arn=GetAtt(cluster, "Arn"),
            EcsParameters=EcsParameters(
                TaskCount=1,
                TaskDefinitionArn=Ref(scheduled_worker_task_definition),
            ),
        ),
    ]
)
{% endhighlight %}

That's it, the task will be placed and ran within a container every 30 minutes.

Here is the *JSON* template:


{% highlight json %}
        ...
        "ScheduledWorkerTask": {
            "Condition": "Deploy",
            "Properties": {
                "ContainerDefinitions": [
                    {
                        "Cpu": 200,
                        "EntryPoint": [
                            "<entry_point>"
                        ],
                        "Essential": "true",
                        "Image": {
                            "Fn::Join": [
                                "",
                                [
                                    {
                                        "Ref": "AWS::AccountId"
                                    },
                                    ".dkr.ecr.",
                                    {
                                        "Ref": "AWS::Region"
                                    },
                                    ".amazonaws.com/",
                                    {
                                        "Ref": "ApplicationRepository"
                                    },
                                    ":",
                                    "<image>"
                                ]
                            ]
                        },
                        "Memory": 300,
                        "Name": "ScheduledWorker"
                    }
                ]
            },
            "Type": "AWS::ECS::TaskDefinition"
        },

        "SchedulingRule": {
            "Properties": {
                "Description": "My schedule task rule",
                "ScheduleExpression": "rate(30 minutes)",
                "State": "ENABLED",
                "Targets": [
                    {
                        "Arn": {
                            "Fn::GetAtt": [
                                "Cluster",
                                "Arn"
                            ]
                        },
                        "EcsParameters": {
                            "TaskCount": 1,
                            "TaskDefinitionArn": {
                                "Ref": "ScheduledWorkerTask"
                            }
                        },
                        "Id": "ScheduledWorkerTaskDefinitionTarget",
                        "RoleArn": {
                            "Fn::GetAtt": [
                                "RunTaskRole",
                                "Arn"
                            ]
                        }
                    }
                ]
            },
            "Type": "AWS::Events::Rule"
        },

        "RunTaskRole": {
            "Properties": {
                "AssumeRolePolicyDocument": {
                    "Statement": [
                        {
                            "Action": [
                                "sts:AssumeRole"
                            ],
                            "Effect": "Allow",
                            "Principal": {
                                "Service": [
                                    "events.amazonaws.com"
                                ]
                            }
                        }
                    ]
                },
                "Path": "/",
                "Policies": [
                    {
                        "PolicyDocument": {
                            "Statement": [
                                {
                                    "Action": [
                                        "ecs:RunTask"
                                    ],
                                    "Condition": {
                                        "ArnEquals": {
                                            "ecs:cluster": {
                                                "Fn::GetAtt": [
                                                    "Cluster",
                                                    "Arn"
                                                ]
                                            }
                                        }
                                    },
                                    "Effect": "Allow",
                                    "Resource": [
                                        "*"
                                    ]
                                }
                            ]
                        },
                        "PolicyName": "RunTaskPolicy"
                    }
                ]
            },
            "Type": "AWS::IAM::Role"
        },
        ...
{% endhighlight %}
