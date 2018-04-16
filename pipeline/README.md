# Background
Infrastructure as Code Cloudformation template for a Codebuild Docker Images pipeline.

The pipeline automates the build of Codebuild Docker images whose Dockerfiles are sourced (through fork)
from the official AWS [aws-codebuild-docker-images](https://github.com/aws/aws-codebuild-docker-images) repository.

# Deployment

## Prerequisites

The instructions below assume that an AWS CLI profile ($HOME/.aws/credentials/config) has been created with a name that is exported as APP_ENV. 

Create a file env.sh that sources the following environment variables:
```
# Git Repo path
GIT_DIR=<path>/aws-codebuild-docker-images
## Account, region and other environment configuration
APP_NAME=codebuild-docker-images-pipeline
APP_ENV=<AWS profile name>
APP_ACCOUNT=<AWS Account>
APP_REGION=<AWS region>

```

Create a Cloudformation parameter file $GIT_DIR/pipeline/codebuild-docker-images-pipeline-${APP_ENV}.json that contains the following parameters:
```
[
  {
    "ParameterKey": "ApplicationName",
    "ParameterValue": "codebuild-docker-images"
  },
  {
    "ParameterKey": "GitHubRepo",
    "ParameterValue": "aws-codebuild-docker-images"
  },
  {
    "ParameterKey": "GitHubBranch",
    "ParameterValue": "virt"
  },
  {
    "ParameterKey": "GitHubUser",
    "ParameterValue": "Virtuability"
  },
  {
    "ParameterKey": "GitHubToken",
    "ParameterValue": "<GitHub token>"
  }
]
```

## Codebuild Docker Images Pipeline

The following commands create and update the Codepipeline and S3 Cloudformation staging bucket.
```

# Source environment
cd $GIT_DIR/pipeline
. env.sh

# Cloudformation staging area for packaging and deployment
aws s3 mb s3://cfn-${APP_ACCOUNT}-${APP_REGION} --region ${APP_REGION} --profile ${APP_ENV}

aws cloudformation validate-template --template-body file://$GIT_DIR/pipeline/${APP_NAME}.template --profile ${APP_ENV}

aws cloudformation package --template-file $GIT_DIR/pipeline/${APP_NAME}.template --s3-bucket cfn-${APP_ACCOUNT}-${APP_REGION} --s3-prefix ${APP_NAME}/$APP_ENV --output-template-file $GIT_DIR/pipeline/${APP_NAME}.packaged --profile ${APP_ENV}

aws cloudformation create-stack --stack-name ${APP_NAME} --template-body file://$GIT_DIR/pipeline/${APP_NAME}.packaged --parameters file://$GIT_DIR/pipeline/parameters/${APP_NAME}-${APP_ENV}.json --tags Key=Owner,Value=aws+virtaws2@virtuability.com Key=ApplicationID,Value=${APP_NAME} Key=Environment,Value=$APP_ENV --capabilities CAPABILITY_NAMED_IAM --region ${APP_REGION} --profile ${APP_ENV}

aws cloudformation update-stack --stack-name ${APP_NAME} --template-body file://$GIT_DIR//pipeline/${APP_NAME}.packaged --parameters file://$GIT_DIR/pipeline/parameters/${APP_NAME}-${APP_ENV}.json --tags Key=Owner,Value=aws+virtaws2@virtuability.com Key=ApplicationID,Value=${APP_NAME} Key=Environment,Value=$APP_ENV --capabilities CAPABILITY_NAMED_IAM --region ${APP_REGION} --profile ${APP_ENV}

# Recommendation is to use change-set over update-stack to validate changes made prior to making them
aws cloudformation create-change-set --change-set-name ${APP_NAME} --stack-name ${APP_NAME} --template-body file://$GIT_DIR/pipeline/${APP_NAME}.packaged --parameters file://$GIT_DIR/pipeline/parameters/${APP_NAME}-${APP_ENV}.json --tags Key=Owner,Value=aws+virtaws2@virtuability.com Key=ApplicationID,Value=${APP_NAME} Key=Environment,Value=$APP_ENV --capabilities CAPABILITY_NAMED_IAM --region ${APP_REGION} --profile ${APP_ENV}

aws cloudformation execute-change-set --change-set-name ${APP_NAME} --stack-name ${APP_NAME} --region ${APP_REGION} --profile ${APP_ENV}

```
