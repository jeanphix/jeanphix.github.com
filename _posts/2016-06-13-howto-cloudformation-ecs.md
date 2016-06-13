---
layout: post
lang: en
title: How to build a scalable AWS web app stack using ECS and CloudFormation
description: How to manage an ECS web app stack using CloudFormation
tags:
- AWS
- CloudFormation
- ECS
- Docker
- Architecture
---

In this small tutorial, I'll try to show you how to deploy a web app
onto a scalable modern [AWS](https://aws.amazon.com/) stack.

As [AWS](https://aws.amazon.com/) rencently introduced new services such as
[ECS](https://aws.amazon.com/ecs/), we can now build simple infrastructures
that handle all usual web app requirements:

* [Docker](https://www.docker.com/) containers orchestration (backends)
* Assets publishing (storage)
* Assets delivery (CDN)
* Databases
...


CloudFormation
--------------

[AWS](https://aws.amazon.com/) offers a service called
[CloudFormation](https://aws.amazon.com/cloudformation/)
which allows us to declaratively orchestrate many of their services by
maintaining a JSON template.

For example, we can create a CloudFormation stack that manages an S3 bucket
by writing up a simple template like this one :

{% highlight json %}
{
    "Resources": {
        "AssetsBucket": {
            "Properties": {
                "AccessControl": "PublicRead",
            },
            "Type": "AWS::S3::Bucket"
        },
    },
    "Outputs": {
        "AssetsBucketDomainName": {
            "Description": "Assets bucket domain name",
            "Value": {
                "Fn::GetAtt": [
                    "AssetsBucket",
                    "DomainName"
                ]
            }
        },
    }
}
{% endhighlight %}

Then, when submitted to CloudFormation, the S3 bucket will be
created and we'll get back its url.

Later, we'll probably want to add a
[CloudFront](https://aws.amazon.com/cloudfront/)
CDN in front of our bucket, we'll then edit the template to
something like :

{% highlight json %}
{
    "Resources": {
        "AssetsBucket": {
            "DeletionPolicy": "Retain",
            "Properties": {
                "AccessControl": "PublicRead",
            },
            "Type": "AWS::S3::Bucket"
        },
        "AssetsDistribution": {
            "Properties": {
                "DistributionConfig": {
                    "DefaultCacheBehavior": {
                        "ForwardedValues": {
                            "QueryString": "false"
                        },
                        "TargetOriginId": "Assets",
                        "ViewerProtocolPolicy": "allow-all"
                    },
                    "Enabled": "true",
                    "Origins": [
                        {
                            "DomainName": {
                                "Fn::GetAtt": [
                                    "AssetsBucket",
                                    "DomainName"
                                ]
                            },
                            "Id": "Assets",
                            "S3OriginConfig": {
                                "OriginAccessIdentity": ""
                            }
                        }
                    ]
                }
            },
            "Type": "AWS::CloudFront::Distribution"
        }
    },
    "Outputs": {
        "AssetsBucketDomainName": {
            "Description": "Assets bucket domain name",
            "Value": {
                "Fn::GetAtt": [
                    "AssetsBucket",
                    "DomainName"
                ]
            }
        },
        "AssetsDistributionDomainName": {
            "Description": "The assest CDN domain name",
            "Value": {
                "Fn::GetAtt": [
                    "AssetsDistribution",
                    "DomainName"
                ]
            }
        }
    }
}
{% endhighlight %}

When submitted as an update, the CloudFront distribution will be added to our
stack. _Note that CloudFormation updates are transactionals, means that if
a resource failed to create or upgrade, the stack is rolled back to previous
state._

As you may notice, the JSON format is not really human friendly and leads
to a very verbose template.

That's why I choosed to use and contribute to a python library called
[troposphere](https://github.com/cloudtools/troposphere) that permits
to abstract CloudFormation templates declaratively.

The same template could be written using troposphere as :

{% highlight python %}
from troposphere import (
    Output,
    GetAtt,
    Template,
)

from troposphere.s3 import (
    Bucket,
    PublicRead,
)

from troposphere.cloudfront import (
    DefaultCacheBehavior,
    Distribution,
    DistributionConfig,
    ForwardedValues,
    Origin,
    S3Origin,
)

# The CloudFormation template
template = Template()

# Create an S3 bucket
assets_bucket = template.add_resource(
    Bucket(
        "AssetsBucket",
        AccessControl=PublicRead,
    )
)

# Output S3 asset bucket domain name
template.add_output(Output(
    "AssetsBucketDomainName",
    Description="Assets bucket domain name",
    Value=GetAtt(assets_bucket, "DomainName")
))

# Create a CloudFront CDN distribution
distribution = template.add_resource(
    Distribution(
        'AssetsDistribution',
        DistributionConfig=DistributionConfig(
            Origins=[Origin(
                Id="Assets",
                DomainName=GetAtt(assets_bucket, "DomainName"),
                S3OriginConfig=S3Origin(
                    OriginAccessIdentity="",
                ),
            )],
            DefaultCacheBehavior=DefaultCacheBehavior(
                TargetOriginId="Assets",
                ForwardedValues=ForwardedValues(
                    QueryString=False
                ),
                ViewerProtocolPolicy="allow-all",
            ),
            Enabled=True
        ),
    )
)

# Output CloudFront url
template.add_output(Output(
    "AssetsDistributionDomainName",
    Description="The assest CDN domain name",
    Value=GetAtt(distribution, "DomainName")
))
{% endhighlight %}

The application
---------------

To illustrate the use case, we need a small web app to deploy within our
stack. For this purpose, I created a tiny
[Django application](https://github.com/jeanphix/hello-django-ecs) that
fits our needs :

{% highlight bash %}
git clone https://github.com/jeanphix/hello-django-ecs.git hello
cd hello/
docker build -t application:01 .
{% endhighlight %}

Basically, the app provides two endpoints :

* */* a basic styled html homepage
* */health-check* health status

The application only requires four persistent stack nodes :

* A Docker repository
* A database
* A storage for static assets
* A storage for logs

The configs are picked up from environment vars (see
[12factors](http://12factor.net/)).

So now, we've got tooling and required assets, let's start to build our awesome
stack!

The Docker repository ([repository.py](https://github.com/jeanphix/ecs/blob/master/stack/repository.py))
=====================

As our first app revision is ready to be deployed, we have to create an
[ECR](https://aws.amazon.com/ecr/) repository that will be responsible
to host our Docker images.

To starts with, we allow all users from our AWS account to manage
the images :

{% highlight python %}
from troposphere import (
    AWS_ACCOUNT_ID,
    AWS_REGION,
    Join,
    Ref,
    Output,
)
from troposphere.ecr import Repository
from awacs.aws import (
    Allow,
    Policy,
    AWSPrincipal,
    Statement,
)
import awacs.ecr as ecr

from .template import template


# Create an `ECR` docker repository
repository = Repository(
    "ApplicationRepository",
    template=template,
    RepositoryName="application",
    # Allow all account users to manage images.
    RepositoryPolicyText=Policy(
        Version="2008-10-17",
        Statement=[
            Statement(
                Sid="AllowPushPull",
                Effect=Allow,
                Principal=AWSPrincipal([
                    Join("", [
                        "arn:aws:iam::",
                        Ref(AWS_ACCOUNT_ID),
                        ":root",
                    ]),
                ]),
                Action=[
                    ecr.GetDownloadUrlForLayer,
                    ecr.BatchGetImage,
                    ecr.BatchCheckLayerAvailability,
                    ecr.PutImage,
                    ecr.InitiateLayerUpload,
                    ecr.UploadLayerPart,
                    ecr.CompleteLayerUpload,
                ],
            ),
        ]
    ),
)


# Output ECR repository URL
template.add_output(Output(
    "RepositoryURL",
    Description="The docker repository URL",
    Value=Join("", [
        Ref(AWS_ACCOUNT_ID),
        ".dkr.ecr.",
        Ref(AWS_REGION),
        ".amazonaws.com/",
        Ref(repository),
    ]),
))
{% endhighlight %}

At this point, when we submit the template to CloudFormation we get back
the _application_ repository url.

We are now ready to push the image :

```bash
# Install AWS CLI
pip install awscli

# Login to ECR repository
`aws ecr get-login --region eu-west-1`

# Push the image
docker tag application:0.1 <accountid>.dkr.ecr.<region>.amazonaws.com/application:0.1
docker push <accountid>.dkr.ecr.<region>.amazonaws.com/application:0.1
```

Our image is now available within the ECR repository.

The VPC [vpc.py](https://github.com/jeanphix/ecs/blob/master/stack/vpc.py)
=======

The [VPC](https://aws.amazon.com/vpc/) network is splitted within five
subnets where we disptach our resources :

* 10.0.1.0/24 *PublicSubnet* that holds the public instances (like NAT).
* 10.0.2.0/24 *LoadbalancerASubnet* within availability zone A
    that holds an interface of our future loadbalancer.
* 10.0.3.0/24 *LoadbalancerBSubnet* within availability zone B
    that holds another interface of our future loadbalancer.
* 10.0.10.0/24 *ContainerASubnet* within avalability zone A that holds
	few of ours future backend instances.
* 10.0.11.0/24 *ContainerBSubnet* within avalability zone B that holds
	the other future backend instances.

{% highlight python %}
from troposphere import (
    AWS_REGION,
    GetAtt,
    Join,
    Ref,
)

from troposphere.ec2 import (
    EIP,
    InternetGateway,
    NatGateway,
    Route,
    RouteTable,
    Subnet,
    SubnetRouteTableAssociation,
    VPC,
    VPCGatewayAttachment,
)

from .template import template


vpc = VPC(
    "Vpc",
    template=template,
    CidrBlock="10.0.0.0/16",
)


# Allow outgoing to outside VPC
internet_gateway = InternetGateway(
    "InternetGateway",
    template=template,
)


# Attach Gateway to VPC
VPCGatewayAttachment(
    "GatewayAttachement",
    template=template,
    VpcId=Ref(vpc),
    InternetGatewayId=Ref(internet_gateway),
)


# Public route table
public_route_table = RouteTable(
    "PublicRouteTable",
    template=template,
    VpcId=Ref(vpc),
)


public_route = Route(
    "PublicRoute",
    template=template,
    GatewayId=Ref(internet_gateway),
    DestinationCidrBlock="0.0.0.0/0",
    RouteTableId=Ref(public_route_table),
)


# Holds public instances
public_subnet_cidr = "10.0.1.0/24"

public_subnet = Subnet(
    "PublicSubnet",
    template=template,
    VpcId=Ref(vpc),
    CidrBlock=public_subnet_cidr,
)


SubnetRouteTableAssociation(
    "PublicSubnetRouteTableAssociation",
    template=template,
    RouteTableId=Ref(public_route_table),
    SubnetId=Ref(public_subnet),
)


# NAT
nat_ip = EIP(
    "NatIp",
    template=template,
    Domain="vpc",
)


nat_gateway = NatGateway(
    "NatGateway",
    template=template,
    AllocationId=GetAtt(nat_ip, "AllocationId"),
    SubnetId=Ref(public_subnet),
)


# Holds load balancer
loadbalancer_a_subnet_cidr = "10.0.2.0/24"
loadbalancer_a_subnet = Subnet(
    "LoadbalancerASubnet",
    template=template,
    VpcId=Ref(vpc),
    CidrBlock=loadbalancer_a_subnet_cidr,
    AvailabilityZone=Join("", [Ref(AWS_REGION), "a"]),
)


SubnetRouteTableAssociation(
    "LoadbalancerASubnetRouteTableAssociation",
    template=template,
    RouteTableId=Ref(public_route_table),
    SubnetId=Ref(loadbalancer_a_subnet),
)


loadbalancer_b_subnet_cidr = "10.0.3.0/24"
loadbalancer_b_subnet = Subnet(
    "LoadbalancerBSubnet",
    template=template,
    VpcId=Ref(vpc),
    CidrBlock=loadbalancer_b_subnet_cidr,
    AvailabilityZone=Join("", [Ref(AWS_REGION), "b"]),
)


SubnetRouteTableAssociation(
    "LoadbalancerBSubnetRouteTableAssociation",
    template=template,
    RouteTableId=Ref(public_route_table),
    SubnetId=Ref(loadbalancer_b_subnet),
)


# Private route table
private_route_table = RouteTable(
    "PrivateRouteTable",
    template=template,
    VpcId=Ref(vpc),
)


private_nat_route = Route(
    "PrivateNatRoute",
    template=template,
    RouteTableId=Ref(private_route_table),
    DestinationCidrBlock="0.0.0.0/0",
    NatGatewayId=Ref(nat_gateway),
)


# Holds containers instances
container_a_subnet_cidr = "10.0.10.0/24"
container_a_subnet = Subnet(
    "ContainerASubnet",
    template=template,
    VpcId=Ref(vpc),
    CidrBlock=container_a_subnet_cidr,
    AvailabilityZone=Join("", [Ref(AWS_REGION), "a"]),
)


SubnetRouteTableAssociation(
    "ContainerARouteTableAssociation",
    template=template,
    SubnetId=Ref(container_a_subnet),
    RouteTableId=Ref(private_route_table),
)


container_b_subnet_cidr = "10.0.11.0/24"
container_b_subnet = Subnet(
    "ContainerBSubnet",
    template=template,
    VpcId=Ref(vpc),
    CidrBlock=container_b_subnet_cidr,
    AvailabilityZone=Join("", [Ref(AWS_REGION), "b"]),
)


SubnetRouteTableAssociation(
    "ContainerBRouteTableAssociation",
    template=template,
    SubnetId=Ref(container_b_subnet),
    RouteTableId=Ref(private_route_table),
)
{% endhighlight %}

I won't detail the routing configuration here, but feel free to shoot me
questions if it's too obscure for you.

The database [database.py](https://github.com/jeanphix/ecs/blob/master/stack/database.py)
============

AWS offers a service called RDS which provides managed database servers
such as [PostgreSQL](https://www.postgresql.org/).

We create a [RDS::DBInstance](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html)
that has interfaces within each of our container subnets.

A [RDS::DBSecurityGroup](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-security-group.html)
ensures that incoming tcp connections are allowed from those subnets :

{% highlight python %}
from troposphere import (
    ec2,
    Parameter,
    rds,
    Ref,
    AWS_STACK_NAME,
)

from .template import template
from .vpc import (
    vpc,
    container_a_subnet,
    container_a_subnet_cidr,
    container_b_subnet,
    container_b_subnet_cidr,
)


db_name = template.add_parameter(Parameter(
    "DatabaseName",
    Default="app",
    Description="The database name",
    Type="String",
    MinLength="1",
    MaxLength="64",
    AllowedPattern="[a-zA-Z][a-zA-Z0-9]*",
    ConstraintDescription=(
        "must begin with a letter and contain only"
        " alphanumeric characters."
    )
))


db_user = template.add_parameter(Parameter(
    "DatabaseUser",
    Default="app",
    Description="The database admin account username",
    Type="String",
    MinLength="1",
    MaxLength="16",
    AllowedPattern="[a-zA-Z][a-zA-Z0-9]*",
    ConstraintDescription=(
        "must begin with a letter and contain only"
        " alphanumeric characters."
    )
))


db_password = template.add_parameter(Parameter(
    "DatabasePassword",
    NoEcho=True,
    Description="The database admin account password",
    Type="String",
    MinLength="10",
    MaxLength="41",
    AllowedPattern="[a-zA-Z0-9]*",
    ConstraintDescription="must contain only alphanumeric characters."
))

db_class = template.add_parameter(Parameter(
    "DatabaseClass",
    Default="db.t2.small",
    Description="Database instance class",
    Type="String",
    AllowedValues=['db.t2.small', 'db.t2.medium'],
    ConstraintDescription="must select a valid database instance type.",
))


db_allocated_storage = template.add_parameter(Parameter(
    "DatabaseAllocatedStorage",
    Default="5",
    Description="The size of the database (Gb)",
    Type="Number",
    MinValue="5",
    MaxValue="1024",
    ConstraintDescription="must be between 5 and 1024Gb.",
))


db_security_group = ec2.SecurityGroup(
    'DatabaseSecurityGroup',
    template=template,
    GroupDescription="Database security group.",
    VpcId=Ref(vpc),
    SecurityGroupIngress=[
        # Postgres in from web clusters
        ec2.SecurityGroupRule(
            IpProtocol="tcp",
            FromPort="5432",
            ToPort="5432",
            CidrIp=container_a_subnet_cidr,
        ),
        ec2.SecurityGroupRule(
            IpProtocol="tcp",
            FromPort="5432",
            ToPort="5432",
            CidrIp=container_b_subnet_cidr,
        ),
    ],
)


db_subnet_group = rds.DBSubnetGroup(
    "DatabaseSubnetGroup",
    template=template,
    DBSubnetGroupDescription="Subnets available for the RDS DB Instance",
    SubnetIds=[Ref(container_a_subnet), Ref(container_b_subnet)],
)


db_instance = rds.DBInstance(
    "PostgreSQL",
    template=template,
    DBName=Ref(db_name),
    AllocatedStorage=Ref(db_allocated_storage),
    DBInstanceClass=Ref(db_class),
    DBInstanceIdentifier=Ref(AWS_STACK_NAME),
    Engine="postgres",
    EngineVersion="9.4.5",
    MultiAZ=True,
    StorageType="gp2",
    MasterUsername=Ref(db_user),
    MasterUserPassword=Ref(db_password),
    DBSubnetGroupName=Ref(db_subnet_group),
    VPCSecurityGroups=[Ref(db_security_group)],
    BackupRetentionPeriod="7",
    DeletionPolicy="Snapshot",
)
{% endhighlight %}


Static assets [assets.py](https://github.com/jeanphix/ecs/blob/master/stack/assets.py)
=============

To manage our static assets, we use two services :

* [S3](https://aws.amazon.com/fr/s3/) to store them
* [CloudFront](https://aws.amazon.com/cloudfront/) to deliver them

We first create a [S3::Bucket](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html)
for which we allow CORS upload from our web app domain and then put a
[CloudFront::Distribution](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cloudfront-distribution.html)
in front of it :

{% highlight python %}
from troposphere import (
    Join,
    Output,
    GetAtt,
)

from troposphere.s3 import (
    Bucket,
    CorsConfiguration,
    CorsRules,
    PublicRead,
    VersioningConfiguration,
)

from troposphere.cloudfront import (
    DefaultCacheBehavior,
    Distribution,
    DistributionConfig,
    ForwardedValues,
    Origin,
    S3Origin,
)

from .template import template
from .domain import domain_name


# Create an S3 bucket that holds statics and media
assets_bucket = template.add_resource(
    Bucket(
        "AssetsBucket",
        AccessControl=PublicRead,
        VersioningConfiguration=VersioningConfiguration(
            Status="Enabled"
        ),
        DeletionPolicy="Retain",
        CorsConfiguration=CorsConfiguration(
            CorsRules=[CorsRules(
                AllowedOrigins=[Join("", [
                    "https://*.",
                    domain_name,
                ])],
                AllowedMethods=["POST", "PUT", "HEAD", "GET", ],
                AllowedHeaders=[
                    "*",
                ]
            )]
        ),
    )
)


# Output S3 asset bucket name
template.add_output(Output(
    "AssetsBucketDomainName",
    Description="Assets bucket domain name",
    Value=GetAtt(assets_bucket, "DomainName")
))


# Create a CloudFront CDN distribution
distribution = template.add_resource(
    Distribution(
        'AssetsDistribution',
        DistributionConfig=DistributionConfig(
            Origins=[Origin(
                Id="Assets",
                DomainName=GetAtt(assets_bucket, "DomainName"),
                S3OriginConfig=S3Origin(
                    OriginAccessIdentity="",
                ),
            )],
            DefaultCacheBehavior=DefaultCacheBehavior(
                TargetOriginId="Assets",
                ForwardedValues=ForwardedValues(
                    QueryString=False
                ),
                ViewerProtocolPolicy="allow-all",
            ),
            Enabled=True
        ),
    )
)


# Output CloudFront url
template.add_output(Output(
    "AssetsDistributionDomainName",
    Description="The assest CDN domain name",
    Value=GetAtt(distribution, "DomainName")
))
{% endhighlight %}

The cluster: ([cluster.py](https://github.com/jeanphix/ecs/blob/master/stack/cluster.py))
============

An [ECS::Cluster](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_GetStarted.html)
is responsible of the container services orchestration :

{% highlight python %}
from troposphere.ecs import (
    Cluster,
)

from .template import template


# ECS cluster
cluster = Cluster(
    "Cluster",
    template=template,
)
{% endhighlight %}

The loadbalancer
----------------

An [ElasticLoadBalancing::LoadBalancer](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-elb.html)
takes care about forwarding HTTP requests the container instances that
run our application.

An SSL certificate is passed by its arn as a stack paramater :

{% highlight python %}
from troposphere import (
    elasticloadbalancing as elb,
    GetAtt,
    Join,
    Output,
    Parameter,
    Ref,
)

from troposphere.ec2 import (
    SecurityGroup,
    SecurityGroupRule,
)

from .template import template
from .vpc import (
    vpc,
    loadbalancer_a_subnet,
    loadbalancer_b_subnet,
)


certificate_id = Ref(template.add_parameter(Parameter(
    "CertId",
    Description="Web SSL certificate id",
    Type="String",
)))


web_worker_port = Ref(template.add_parameter(Parameter(
    "WebWorkerPort",
    Description="Web worker container exposed port",
    Type="Number",
    Default="8000",
)))


# Web load balancer
load_balancer_security_group = SecurityGroup(
    "LoadBalancerSecurityGroup",
    template=template,
    GroupDescription="Web load balancer security group.",
    VpcId=Ref(vpc),
    SecurityGroupIngress=[
        SecurityGroupRule(
            IpProtocol="tcp",
            FromPort="443",
            ToPort="443",
            CidrIp='0.0.0.0/0',
        ),
    ],
)

load_balancer = elb.LoadBalancer(
    'LoadBalancer',
    template=template,
    Subnets=[
        Ref(loadbalancer_a_subnet),
        Ref(loadbalancer_b_subnet),
    ],
    SecurityGroups=[Ref(load_balancer_security_group)],
    Listeners=[elb.Listener(
        LoadBalancerPort=443,
        InstanceProtocol='HTTP',
        InstancePort=web_worker_port,
        Protocol='HTTPS',
        SSLCertificateId=certificate_id,
    )],
    HealthCheck=elb.HealthCheck(
        Target=Join("", ["HTTP:", web_worker_port, "/health-check"]),
        HealthyThreshold="2",
        UnhealthyThreshold="2",
        Interval="100",
        Timeout="10",
    ),
    CrossZone=True,
)

template.add_output(Output(
    "LoadBalancerDNSName",
    Description="Loadbalancer DNS",
    Value=GetAtt(load_balancer, "DNSName")
))
{% endhighlight %}


The container instances
-----------------------

The cluster is composed of several EC2 instances, let's call them the
*container instances* that host the Docker containers required by
the application.

Each *container instance* registers by itself to the cluster. To prepare
those instances, we create an
[IAM::InstanceProfile](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)
bound to an [IAM::Role](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html),
with proper credentials to manage the
[ECS::Cluster](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-cluster.html) :

{% highlight python %}
from troposphere import (
    iam,
    Ref,
)

from .template import template


# ECS container role
container_instance_role = iam.Role(
    "ContainerInstanceRole",
    template=template,
    AssumeRolePolicyDocument=dict(Statement=[dict(
        Effect="Allow",
        Principal=dict(Service=["ec2.amazonaws.com"]),
        Action=["sts:AssumeRole"],
    )]),
    Path="/",
    Policies=[
        iam.Policy(
            PolicyName="ECSManagementPolicy",
            PolicyDocument=dict(
                Statement=[dict(
                    Effect="Allow",
                    Action=[
                        "ecs:*",
                        "elasticloadbalancing:*",
                    ],
                    Resource="*",
                )],
            ),
        ),
    ]
)


# ECS container instance profile
container_instance_profile = iam.InstanceProfile(
    "ContainerInstanceProfile",
    template=template,
    Path="/",
    Roles=[Ref(container_instance_role)],
)
{% endhighlight %}

In order to define the container instance boostrap, we use an
[AutoScaling::LaunchConfiguration](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-as-launchconfig.html)
that configures cluster management requirements :

{% highlight python %}
from troposphere import (
    AWS_REGION,
    AWS_STACK_ID,
    AWS_STACK_NAME,
    autoscaling,
    Base64,
    cloudformation,
    FindInMap,
    Join,
    Parameter,
    Ref,
)

from troposphere.ec2 import (
    SecurityGroup,
    SecurityGroupRule,
)

from .template import template
from .vpc import (
    vpc,
    loadbalancer_a_subnet_cidr,
    loadbalancer_b_subnet_cidr,
)


container_instance_type = Ref(template.add_parameter(Parameter(
    "ContainerInstanceType",
    Description="The container instance type",
    Type="String",
    Default="t2.micro",
    AllowedValues=["t2.micro", "t2.small", "t2.medium"]
)))


web_worker_port = Ref(template.add_parameter(Parameter(
    "WebWorkerPort",
    Description="Web worker container exposed port",
    Type="Number",
    Default="8000",
)))


template.add_mapping("ECSRegionMap", {
    "eu-west-1": {"AMI": "ami-4e6ffe3d"},
    "us-east-1": {"AMI": "ami-8f7687e2"},
    "us-west-2": {"AMI": "ami-84b44de4"},
})


# ...


container_security_group = SecurityGroup(
    'ContainerSecurityGroup',
    template=template,
    GroupDescription="Container security group.",
    VpcId=Ref(vpc),
    SecurityGroupIngress=[
        # HTTP from web public subnets
        SecurityGroupRule(
            IpProtocol="tcp",
            FromPort=web_worker_port,
            ToPort=web_worker_port,
            CidrIp=loadbalancer_a_subnet_cidr,
        ),
        SecurityGroupRule(
            IpProtocol="tcp",
            FromPort=web_worker_port,
            ToPort=web_worker_port,
            CidrIp=loadbalancer_b_subnet_cidr,
        ),
    ],
)


container_instance_configuration_name = "ContainerLaunchConfiguration"


container_instance_configuration = autoscaling.LaunchConfiguration(
    container_instance_configuration_name,
    template=template,
    Metadata=autoscaling.Metadata(
        cloudformation.Init(dict(
            config=cloudformation.InitConfig(
                commands=dict(
                    register_cluster=dict(command=Join("", [
                        "#!/bin/bash\n",
                        # Register the cluster
                        "echo ECS_CLUSTER=",
                        Ref(cluster),
                        " >> /etc/ecs/config\n",
                    ]))
                ),
                files=cloudformation.InitFiles({
                    "/etc/cfn/cfn-hup.conf": cloudformation.InitFile(
                        content=Join("", [
                            "[main]\n",
                            "template=",
                            Ref(AWS_STACK_ID),
                            "\n",
                            "region=",
                            Ref(AWS_REGION),
                            "\n",
                        ]),
                        mode="000400",
                        owner="root",
                        group="root",
                    ),
                    "/etc/cfn/hooks.d/cfn-auto-reload.conf":
                    cloudformation.InitFile(
                        content=Join("", [
                            "[cfn-auto-reloader-hook]\n",
                            "triggers=post.update\n",
                            "path=Resources.%s."
                            % container_instance_configuration_name,
                            "Metadata.AWS::CloudFormation::Init\n",
                            "action=/opt/aws/bin/cfn-init -v ",
                            "         --template ",
                            Ref(AWS_STACK_NAME),
                            "         --resource %s"
                            % container_instance_configuration_name,
                            "         --region ",
                            Ref("AWS::Region"),
                            "\n",
                            "runas=root\n",
                        ])
                    )
                }),
                services=dict(
                    sysvinit=cloudformation.InitServices({
                        'cfn-hup': cloudformation.InitService(
                            enabled=True,
                            ensureRunning=True,
                            files=[
                                "/etc/cfn/cfn-hup.conf",
                                "/etc/cfn/hooks.d/cfn-auto-reloader.conf",
                            ]
                        ),
                    })
                )
            )
        ))
    ),
    SecurityGroups=[Ref(container_security_group)],
    InstanceType=container_instance_type,
    ImageId=FindInMap("ECSRegionMap", Ref(AWS_REGION), "AMI"),
    IamInstanceProfile=Ref(container_instance_profile),
    UserData=Base64(Join('', [
        "#!/bin/bash -xe\n",
        "yum install -y aws-cfn-bootstrap\n",

        "/opt/aws/bin/cfn-init -v ",
        "         --template ", Ref(AWS_STACK_NAME),
        "         --resource %s " % container_instance_configuration_name,
        "         --region ", Ref(AWS_REGION), "\n",
    ])),
)
{% endhighlight %}

The container instances are managed by an
[Autoscaling::AutoScalingGroup](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-as-group.html)
that ensures we have enough running instances within desired subnets and
replaces the non healthy ones :

{% highlight python %}
from troposphere import (
    autoscaling,
    Parameter,
    Ref,
)

from .template import template
from .vpc import (
    container_a_subnet,
    container_b_subnet,
)


container_instance_type = Ref(template.add_parameter(Parameter(
    "ContainerInstanceType",
    Description="The container instance type",
    Type="String",
    Default="t2.micro",
    AllowedValues=["t2.micro", "t2.small", "t2.medium"]
)))


web_worker_port = Ref(template.add_parameter(Parameter(
    "WebWorkerPort",
    Description="Web worker container exposed port",
    Type="Number",
    Default="8000",
)))


max_container_instances = Ref(template.add_parameter(Parameter(
    "MaxScale",
    Description="Maximum container instances count",
    Type="Number",
    Default="3",
)))


desired_container_instances = Ref(template.add_parameter(Parameter(
    "DesiredScale",
    Description="Desired container instances count",
    Type="Number",
    Default="3",
)))


# ...


autoscaling_group_name = "AutoScalingGroup"


autoscaling_group = autoscaling.AutoScalingGroup(
    autoscaling_group_name,
    template=template,
    VPCZoneIdentifier=[Ref(container_a_subnet), Ref(container_b_subnet)],
    MinSize=desired_container_instances,
    MaxSize=max_container_instances,
    DesiredCapacity=desired_container_instances,
    LaunchConfigurationName=Ref(container_instance_configuration),
    LoadBalancerNames=[Ref(load_balancer)],
    # Since one instance within the group is a reserved slot
    # for rolling ECS service upgrade, it's not possible to rely
    # on a "dockerized" `ELB` health-check, else this reserved
    # instance will be flagged as `unhealthy` and won't stop respawning'
    HealthCheckType="EC2",
    HealthCheckGracePeriod=300,
)
{% endhighlight %}


The application service
=======================

Logging
-------

A [Logs::LogGroup](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-logs-loggroup.html)
keeps the Docker logs from our *container instances* :

{% highlight python %}
from troposphere import logs

from .template import template


web_log_group = logs.LogGroup(
    "WebLogs",
    template=template,
    RetentionInDays=365,
    DeletionPolicy="Retain",
)
{% endhighlight %}

We next allow *container instances* to put new logs by adding the
appropriate policy to their instance profile :

{% highlight python %}
# ...
         iam.Policy(
            PolicyName="LoggingPolicy",
            PolicyDocument=dict(
                Statement=[dict(
                    Effect="Allow",
                    Action=[
                        "logs:Create*",
                        "logs:PutLogEvents",
                    ],
                    Resource="arn:aws:logs:*:*:*",
                )],
            ),
        ),
# ...
{% endhighlight %}

Then enable the *awslogs* docker logging driver by updating the
launch configuration :

{% highlight python %}
# Enable CloudWatch docker logging
'echo \'ECS_AVAILABLE_LOGGING_DRIVERS=',
'["json-file","awslogs"]\'',
" >> /etc/ecs/config\n",
{% endhighlight %}


Managing static assets
----------------------

A new policy is added to the *container instance* instance role to allow
the containers to manage the assets bucket :

{% highlight python %}
# ...
        iam.Policy(
            PolicyName="AssetsManagementPolicy",
            PolicyDocument=dict(
                Statement=[dict(
                    Effect="Allow",
                    Action=[
                        "s3:ListBucket",
                    ],
                    Resource=Join("", [
                        "arn:aws:s3:::",
                        Ref(assets_bucket),
                    ]),
                ), dict(
                    Effect="Allow",
                    Action=[
                        "s3:*",
                    ],
                    Resource=Join("", [
                        "arn:aws:s3:::",
                        Ref(assets_bucket),
                        "/*",
                    ]),
                )],
            ),
        ),
# ...
{% endhighlight %}


Pulling Docker images
---------------------

Another policy is added to allow *container instances* to download new
application images :

{% highlight python %}
# ...
        iam.Policy(
            PolicyName='ECRManagementPolicy',
            PolicyDocument=dict(
                Statement=[dict(
                    Effect='Allow',
                    Action=[
                        ecr.GetAuthorizationToken,
                        ecr.GetDownloadUrlForLayer,
                        ecr.BatchGetImage,
                        ecr.BatchCheckLayerAvailability,
                    ],
                    Resource="*",
                )],
            ),
        ),
# ...
{% endhighlight %}


The task definition
-------------------

An [ECS::TaskDefinition](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html)
takes care about defining how Docker containers are ran an exposes the
appropriate env vars to allow the application to know about our stack
resources :

{% highlight python %}
from troposphere import (
    AWS_ACCOUNT_ID,
    AWS_REGION,
    Equals,
    GetAtt,
    Join,
    Not,
    Parameter,
    Ref,
)

from troposphere.ecs import (
    ContainerDefinition,
    Environment,
    LogConfiguration,
    PortMapping,
    TaskDefinition,
)

from .template import template
from .assets import (
    assets_bucket,
    distribution,
)
from .database import (
    db_instance,
    db_name,
    db_user,
    db_password,
)
from .domain import domain_name
from .repository import repository


web_worker_cpu = Ref(template.add_parameter(Parameter(
    "WebWorkerCPU",
    Description="Web worker CPU units",
    Type="Number",
    Default="512",
)))


web_worker_memory = Ref(template.add_parameter(Parameter(
    "WebWorkerMemory",
    Description="Web worker memory",
    Type="Number",
    Default="700",
)))


web_worker_desired_count = Ref(template.add_parameter(Parameter(
    "WebWorkerDesiredCount",
    Description="Web worker task instance count",
    Type="Number",
    Default="2",
)))


app_revision = Ref(template.add_parameter(Parameter(
    "WebAppRevision",
    Description="An optional docker app revision to deploy",
    Type="String",
    Default="",
)))


deploy_condition = "Deploy"
template.add_condition(deploy_condition, Not(Equals(app_revision, "")))


secret_key = Ref(template.add_parameter(Parameter(
    "SecretKey",
    Description="Application secret key",
    Type="String",
)))


# ...


# ECS task
web_task_definition = TaskDefinition(
    "WebTask",
    template=template,
    Condition=deploy_condition,
    ContainerDefinitions=[
        ContainerDefinition(
            Name="WebWorker",
            #  1024 is full CPU
            Cpu=web_worker_cpu,
            Memory=web_worker_memory,
            Essential=True,
            Image=Join("", [
                Ref(AWS_ACCOUNT_ID),
                ".dkr.ecr.",
                Ref(AWS_REGION),
                ".amazonaws.com/",
                Ref(repository),
                ":",
                app_revision,
            ]),
            PortMappings=[PortMapping(
                ContainerPort=web_worker_port,
                HostPort=web_worker_port,
            )],
            LogConfiguration=LogConfiguration(
                LogDriver="awslogs",
                Options={
                    'awslogs-group': Ref(web_log_group),
                    'awslogs-region': Ref(AWS_REGION),
                }
            ),
            Environment=[
                Environment(
                    Name="AWS_STORAGE_BUCKET_NAME",
                    Value=Ref(assets_bucket),
                ),
                Environment(
                    Name="CDN_DOMAIN_NAME",
                    Value=GetAtt(distribution, "DomainName"),
                ),
                Environment(
                    Name="DOMAIN_NAME",
                    Value=domain_name,
                ),
                Environment(
                    Name="PORT",
                    Value=web_worker_port,
                ),
                Environment(
                    Name="SECRET_KEY",
                    Value=secret_key,
                ),
                Environment(
                    Name="DATABASE_URL",
                    Value=Join("", [
                        "postgres://",
                        Ref(db_user),
                        ":",
                        Ref(db_password),
                        "@",
                        GetAtt(db_instance, 'Endpoint.Address'),
                        "/",
                        Ref(db_name),
                    ]),
                ),
            ],
        )
    ],
)
{% endhighlight %}

The service
-----------

An [ECS::Service](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html)
is setted up with proper credentials (as an
[IAM::Role](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html))
to run the task :


{% highlight python %}
from troposphere import (
    iam,
    Parameter,
    Ref,
)

from troposphere.ecs import (
    LoadBalancer,
    Service,
)

from .template import template


web_worker_desired_count = Ref(template.add_parameter(Parameter(
    "WebWorkerDesiredCount",
    Description="Web worker task instance count",
    Type="Number",
    Default="2",
)))


app_service_role = iam.Role(
    "AppServiceRole",
    template=template,
    AssumeRolePolicyDocument=dict(Statement=[dict(
        Effect="Allow",
        Principal=dict(Service=["ecs.amazonaws.com"]),
        Action=["sts:AssumeRole"],
    )]),
    Path="/",
    Policies=[
        iam.Policy(
            PolicyName="WebServicePolicy",
            PolicyDocument=dict(
                Statement=[dict(
                    Effect="Allow",
                    Action=[
                        "elasticloadbalancing:Describe*",
                        "elasticloadbalancing"
                        ":DeregisterInstancesFromLoadBalancer",
                        "elasticloadbalancing"
                        ":RegisterInstancesWithLoadBalancer",
                        "ec2:Describe*",
                        "ec2:AuthorizeSecurityGroupIngress",
                    ],
                    Resource="*",
                )],
            ),
        ),
    ]
)


app_service = Service(
    "AppService",
    template=template,
    Cluster=Ref(cluster),
    Condition=deploy_condition,
    DependsOn=[autoscaling_group_name],
    DesiredCount=web_worker_desired_count,
    LoadBalancers=[LoadBalancer(
        ContainerName="WebWorker",
        ContainerPort=web_worker_port,
        LoadBalancerName=Ref(load_balancer),
    )],
    TaskDefinition=Ref(web_task_definition),
    Role=Ref(app_service_role),
)
{% endhighlight %}


Conclusion
==========

We now have a well designed stack that covers all our application
requirements.

All persitent data (image repository, assets, database, logs) are
handled by AWS managed services so we do not have to maintain
any servers using tools like puppet, ansible, saltstack...

You can browse the whole stuff we built here
[on github](https://github.com/jeanphix/ecs).

Next time I'll show you how to integrate continuous deployment to
the stack.

Cheers,
