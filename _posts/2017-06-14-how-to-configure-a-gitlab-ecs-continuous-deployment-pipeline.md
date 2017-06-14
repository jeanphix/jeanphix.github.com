---
layout: post
lang: en
title: How to configure a Gitlab / ECS continuous deployment pipeline
description: How to configure a Gitlab / ECS continuous deployment pipeline
tags:
- AWS
- Gitlab
- Continuous Deployment
- Docker
- ECS
- CloudFormation
---

Here will see how to build a _Gitlab CI/CD pipeline_ to deploy new codebase
to _ECS_.

The _pipeline_ will be split in three steps (stages):

* build the new docker image
* run the test suite within the new image
* deploy the new image to ECS


Setting up the AWS deployment key pair
--------------------------------------

In order to let _Gitlab CI_ to deploy new docker images to _ECR repository_
then update the _CloudFormation stack_ to use the image we need to
prepare an _IAM key pair_ with appropriate credentials to update the
_CloudFormation stack_ and resources.

It looks like:

{% highlight python %}
from troposphere import (
    AWS_ACCOUNT_ID,
    AWS_REGION,
    AWS_STACK_NAME,
    GetAtt,
    Join,
    Output,
    Ref,
)

from troposphere.iam import (
    AccessKey,
    PolicyType,
    User,
)

from .cluster import template


user = Ref(User(
    "DeploymentUser",
    template=template,
))


key = AccessKey(
    "DeploymentKey",
    template=template,
    UserName=user,
)


PolicyType(
    "DeploymentStackPolicy",
    template=template,
    PolicyName="DeploymentStackPolicy",
    Users=[user],
    PolicyDocument=dict(
        Statement=[dict(
            Effect="Allow",
            Action=[
                "cloudformation:UpdateStack",
                "cloudformation:DescribeStackEvents",
                "cloudformation:DescribeStacks",
            ],
            Resource=Join("", [
                "arn:aws:cloudformation:",
                Ref(AWS_REGION),
                ":",
                Ref(AWS_ACCOUNT_ID),
                ":stack/",
                Ref(AWS_STACK_NAME),
                "/*",
            ]),
        )],
    )
)


PolicyType(
    "DeploymentECRTokenPolicy",
    template=template,
    PolicyName="DeploymentECRTokenPolicy",
    Users=[user],
    PolicyDocument=dict(
        Statement=[dict(
            Effect="Allow",
            Action=[
                "ecr:GetAuthorizationToken",
            ],
            Resource="*",
        )],
    )
)


PolicyType(
    "DeploymentUpgradePolicy",
    template=template,
    PolicyName="DeploymentUpgradePolicy",
    Users=[user],
    PolicyDocument=dict(
        Statement=[dict(
            Effect="Allow",
            Action=[
                "ec2:Describe*",
                "ecs:*",
                "iam:PassRole",
                "rds:Describe*",
                "cloudfront:GetDistribution",
                "elasticloadbalancing:Describe*",
                "logs:DescribeLogGroups",
            ],
            Resource="*",
        )],
    )
)


template.add_output(Output(
    "DeploymentAccessKeyID",
    Description="The deployment access key",
    Value=Ref(key),
))


template.add_output(Output(
    "DeploymentSecretKey",
    Description="The deployment secret key",
    Value=GetAtt(key, "SecretAccessKey"),
))
{% endhighlight %}


The ECR repository also need to allow the new deployment user to
manage images:

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
from .deployment import user


# Create an `ECR` docker repository
repository = Repository(
    "ApplicationRepository",
    template=template,
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
                        ":user/",
                        user,
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


Setting up the Gitlab CD/CI pipeline
------------------------------------

First of all, as we don't want to share it in our codebase, we need
to setup target environment configs within Gitlab to have them
exposed as env vars.

We simply go to _settings_ > _CI/CD Pipelines_ within the project web
UI and then set them to something like:

![Gitlab configs]({{ site.url }}/static/images/gitlab-configs.png)

Then we need to describe our continuous deployment pipeline within a
_.gitlab-ci.yml_ file:

{% highlight yaml %}
image: docker:dind

stages:
    - build
    - test
    - staging

variables:
    DOCKER_DRIVER: overlay
    DOCKER_HOST: tcp://docker:2375

    # The commit image to push to Gitlab repository
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

    # The latest image in Gitlab repository
    LATEST_IMAGE: $CI_REGISTRY_IMAGE:latest

    # Postgres configuration
    POSTGRES_DB: app_test
    POSTGRES_USER: runner
    POSTGRES_PASSWORD: ""
    DATABASE_URL: postgresql+psycopg2://$POSTGRES_USER:$POSTGRES_PASSWORD@postgres:5432/app

    # The stack template
    STACK_TEMPLATE: stack.services.application.template

    # The AWS region
    STAGING_AWS_REGION: $STAGING_AWS_REGION

    # The AWS cloudformation stack name
    STAGING_AWS_STACKNAME: $STAGING_AWS_STACKNAME

    # The AWS ECR repository
    STAGING_ECR_REPOSITORY: $STAGING_ECR_REPOSITORY

    # The AWS deployement user access key
    STAGING_AWS_ACCESS_KEY_ID: $STAGING_AWS_ACCESS_KEY_ID
    STAGING_AWS_SECRET_ACCESS_KEY: $STAGING_AWS_SECRET_ACCESS_KEY

services:
    - docker:dind

build:
    stage: build
    script:
        - docker info

        # Login to Gitlab registry
        - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN registry.gitlab.com

        # Pull latest image from Gitlab registry (if exists)
        # to re-use existing layers from previous build
        - set +e && docker pull $LATEST_IMAGE && set -e

        # Build commit image
        - docker build --cache-from $LATEST_IMAGE -t $IMAGE_TAG .

        # Push commit image
        - docker push $IMAGE_TAG

        # Push commit image as latest
        - docker tag $IMAGE_TAG $LATEST_IMAGE
        - docker push $LATEST_IMAGE

test:
    stage: test
    image: $IMAGE_TAG

    services:
        - docker:dind
        - postgres:latest

    script:
        # Run the test suite within commit image
        - entries/test.sh

staging:
    stage: staging
    type: deploy

    script:
        # Login to Gitlab registry
        - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN registry.gitlab.com

        # Pull commit image
        - docker pull $IMAGE_TAG

        # Install python requirements
        - apk update
        - apk upgrade
        - apk add python python-dev py-pip

        # AWS configs
        - export AWS_REGION=$STAGING_AWS_REGION
        - export AWS_ACCESS_KEY_ID=$STAGING_AWS_ACCESS_KEY_ID
        - export AWS_SECRET_ACCESS_KEY=$STAGING_AWS_SECRET_ACCESS_KEY

        # The ECR image tag
        - export ECR_IMAGE_TAG=$STAGING_ECR_REPOSITORY:$CI_COMMIT_SHA

        # Push the commit image to ECR repository
        - pip install awscli
        - "`aws ecr get-login --region $AWS_REGION`"
        - docker tag $IMAGE_TAG $ECR_IMAGE_TAG
        - docker push $ECR_IMAGE_TAG

        # Update the CloudFormation stack to user new image
        - pip install troposphere-cli awacs
        - trop update $STAGING_AWS_STACKNAME -p WebAppRevision $CI_COMMIT_SHA --iam --tail

    only:
        # Only deploy staging branch
        - staging

    environment:
        name: staging
{% endhighlight %}

From there, each commit on staging branch will trigger a deployment
pipeline to our _ECS tasks_:

![Gitlab pipeline]({{ site.url }}/static/images/gitlab-pipeline.png)

Cheers,
