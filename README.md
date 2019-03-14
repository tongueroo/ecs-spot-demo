# ECS Spot Fleet Demo

This is a sample template that demonstrates how to use ECS with Spot Fleets.

## Usage

1. Launch Stack with Spot Fleets
2. Deploy Demo App
3. Cleanup

## Launch Stack

To launch the stack without a KeyName:

    aws cloudformation create-stack --stack-name ecs-spot-demo --template-body file://ecs-spot-demo.yml --capabilities CAPABILITY_IAM


If you would like to launch the stack with an ssh key. Here are the commands:


    # Quick way to grab first keypair on the AWS account.
    KEY_NAME=$(aws ec2 describe-key-pairs | jq -r '.KeyPairs[0].KeyName')

    # Create a parameters file for the one required parameter in this template
    cat <<EOL > parameters.json
    [
      {
        "ParameterKey": "KeyName",
        "ParameterValue": "$KEY_NAME"
      }
    ]
    EOL

    # Launch the stack
    aws cloudformation create-stack --stack-name ecs-spot-demo --template-body file://ecs-spot-demo.yml --parameters file://parameters.json --capabilities CAPABILITY_IAM

## Deploy Demo App

First let's create the ECR repo and set some variables we'll need:

    ECR_REPO=$(aws ecr create-repository --repository-name demo/sinatra | jq -r '.repository.repositoryUri')
    VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values="demo vpc" | jq -r '.Vpcs[].VpcId')

Now we're ready to clone the demo repo and deploy an sample app to ECS with ufo. Note, you'll need Docker installed.

    git clone https://github.com/tongueroo/demo-ufo.git demo
    cd demo
    ufo init --image $ECR_REPO --vpc-id $VPC_ID
    ufo current --service demo-web
    ufo ship # deploys to ECS on the Spot Fleet cluster


## Cleanup

To clean up. First, delete the ECS service.

    cd demo # demo project
    ufo destroy

Delete the ECR repo:

    IMAGES=$(aws ecr list-images --repository-name demo/sinatra | jq -r '.imageIds[].imageDigest')
    for i in $IMAGES; do aws ecr batch-delete-image --repository-name demo/sinatra --image-ids imageDigest=$i ; done
    aws ecr delete-repository --repository-name demo/sinatra

Delete the infrastructure CloudFormation stack:

    aws cloudformation delete-stack --stack-name ecs-spot-demo
