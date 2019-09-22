# DevOpsDaysBoston
Sample repository for DevOpsDays Boston Workshop

# Pre-requisites

* AWS Account
* IAM User with git credentials <!-- Create guide-->
* Python 3
* AWS SAM CLI
* Postman (nice to have, but not required)


Running these commands should produce similar results if you have Python and the SAM CLI installed correctly
```bash
$ sam --version
SAM CLI, version 0.18.0
$ python --version
Python 3.7.2
```

# Introduction To The Tools

In this workshop we'll be using the following services from AWS to manage our deployments:

* CodeCommit
* CodeBuild
* CodePipeline
* CodeDeploy
* CloudFormation

Our application will use the following services as part of it's business logic:

* Lambda
* API Gateway
* DynamoDB

# Setting Up Our Environment
First we'll create a folder that will will hold our Cloudformation Templates.
```bash
$ cd MyPipelineFolder
$ touch pipeline.yaml
```

Let's open `pipeline.yaml` in a text editor and add some CloudFormation boiler-plate.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Our Pipeline Stack

Parameters:
  ProjectName:
    Description: Name of the Project
    Type: String
    Default: "MyDevOpsProject"

Resources:
  MyRepo:
    Type: AWS::CodeCommit::Repository
    Properties:
      RepositoryName: !Ref ProjectName
```

*This complete template can be found in /templates/001_pipeline.yaml*

This will create a CodeCommit Repository named after the supplied `ProjectName` parameter. Since we provided a default value, we aren't required to explicitly specify it later when we deploy the template. Let's go ahead and try to deploy this just to make sure our environment is set up correctly.

```
$ aws cloudformation deploy --template-file ./pipeline.yaml --stack-name devopsday-pipeline

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - devopsday-pipeline
```

Let's go look at our console and verify this was created correctly
<!-- Add Image -->
Perfect! Now we'll add our S3 Bucket. Add these to the Resources section, the order doesn't matter so they can be placed anywhere in the `Resources` section of the template. We are creating this bucket so that CodeBuild has a place to put your compiled build artifacts, these are things like JARs, CloudFormation Templates, Executables, etc...

Your template should now look like this. As the template grows I will stop showing the entire template in an effort to save space on your screen.
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Our Pipeline Stack

Parameters:
  ProjectName:
    Description: Name of the Project
    Type: String
    Default: "MyDevOpsProject"

Resources:
  MyRepo:
    Type: AWS::CodeCommit::Repository
    Properties:
      RepositoryName: !Ref ProjectName

  S3Bucket:
    Type: AWS::S3::Bucket
```
*This complete template can be found in /templates/002_pipeline.yaml*

Now we'll add an IAM Role that the CodeBuild Project will use when running a build. Since our project will just dump files into S3 for later use by the pipeline, we don't need too many permissions.

```yaml
  BuildProjectRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${ProjectName}-CodeBuildRole
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - codebuild.amazonaws.com
            Action:
              - sts:AssumeRole
      Path: /

  BuildProjectPolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: !Sub ${ProjectName}-CodeBuildPolicy
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Action:
              - s3:PutObject
              - s3:GetBucketPolicy
              - s3:GetObject
              - s3:ListBucket
            Resource:
             - !Join ['',['arn:aws:s3:::',!Ref S3Bucket, '/*']]
             - !Join ['',['arn:aws:s3:::',!Ref S3Bucket]]
          - Effect: Allow
            Action:
              - logs:CreateLogGroup
              - logs:CreateLogStream
              - logs:PutLogEvents
            Resource: arn:aws:logs:*:*:*
      Roles:
        - !Ref BuildProjectRole
```
*This complete template can be found in /templates/003_pipeline.yaml*

Now, in order to deploy this we are going to have to make a small change to the command we previously used. We need to tell CloudFormation that we're aware that this stack will create named IAM resources, namely our IAM Role and Policy

```bash
$ aws cloudformation deploy --template-file ./pipeline.yaml --stack-name devopsday-pipeline --capabilities CAPABILITY_NAMED_IAM

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - devopsday-pipeline
```

Now we're about to get into the fun stuff. Let's create our CodeBuild Project. This Project will do the actual "building" or "compiling" of our project. Usually this is where you would do things like `pip install ...` or `npm run install` etc... Add the following to your `Resources` section.

```yaml
  BuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Ref ProjectName
      Description: !Ref ProjectName
      ServiceRole: !GetAtt BuildProjectRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/standard:2.0
        PrivilegedMode: true
        EnvironmentVariables:
          - Name: S3Bucket
            Value: !Ref S3Bucket
      Source:
        Type: CODEPIPELINE
        BuildSpec: |
          version: 0.2
          phases:
            install:
              runtime-versions:
                python: 3.7
              commands:
                - ls -R
            build:
              commands:
                - pip install --user aws-sam-cli pytest pytest-mock
                - USER_BASE_PATH=$(python -m site --user-base)
                - export PATH=$PATH:$USER_BASE_PATH/bin
                - python -m pytest tests/ -v
                - sam build --use-container -b ./build/ -t template.yaml
                - aws cloudformation package --template-file ./build/template.yaml --s3-bucket $S3Bucket --s3-prefix twitchext/lambda --output-template-file ./output.yaml
          artifacts:
            files:
              - ./output.yaml
            discard-paths: yes
      TimeoutInMinutes: 10
```
*This complete template can be found in /templates/004_pipeline.yaml*

Let's go ahead and re-deploy our stack and see how it goes.
<!-- Demo will spend a few minutes explaining all of these options -->
```bash
$ aws cloudformation deploy --template-file ./pipeline.yaml --stack-name devopsday-pipeline --capabilities CAPABILITY_NAMED_IAM

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - devopsday-pipeline
```

Now we have a CodeBuild Project and a CodeCommit repository but they're not connected to each other. They're just drifting in the ether of the cloud, let's bring them together with a CodePipeline Pipeline. But first, we need to create an IAM Role and Policy for the Pipeline to use and an IAM Role for CloudFormation to use when building resources.

<!-- Detour into IAM Policies and how important it is to protect them -->

```yaml
  PipeLineRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${ProjectName}-codepipeline-role
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - codepipeline.amazonaws.com
            Action:
              - sts:AssumeRole
      Path: /

  PipelinePolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: !Sub ${ProjectName}-codepipeline-policy
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - 
            Effect: Allow
            Action:
              - cloudformation:CreateStack
              - cloudformation:DeleteStack
              - cloudformation:Describe*
              - cloudformation:UpdateStack
              - cloudformation:CreateChangeSet
              - cloudformation:DeleteChangeSet
              - cloudformation:ExecuteChangeSet
            Resource:
              - !Sub arn:${AWS::Partition}:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/*
              - !Sub arn:${AWS::Partition}:cloudformation:${AWS::Region}:${AWS::AccountId}:changeSet/*
          - 
            Effect: Allow
            Action:
              - s3:GetBucketPolicy
              - s3:GetObject
              - s3:ListBucket
            Resource:
             - !Sub '${S3Bucket.Arn}/*'
             - !Sub '${S3Bucket.Arn}'
          - 
            Effect: Allow
            Action:
              - codecommit:*
            Resource:
             - !GetAtt MyRepo.Arn
          - 
            Effect: Allow
            Action:
              - iam:PassRole
            Resource:
             - !GetAtt CFDeployerRole.Arn
          - 
            Effect: Allow
            Action:
              - codebuild:StartBuild
              - codebuild:BatchGetBuilds
            Resource:
             - !GetAtt BuildProject.Arn
          - 
            Effect: Allow
            Action:
              - s3:PutObject
              - s3:GetBucketPolicy
              - s3:GetObject
              - s3:ListBucket
            Resource:
             - !Join ['',['arn:aws:s3:::',!Ref S3Bucket, '/*']]
             - !Join ['',['arn:aws:s3:::',!Ref S3Bucket]]
          - 
            Effect: Allow
            Action:
              - sts:AssumeRole
            Resource:
              - !GetAtt PipeLineRole.Arn
      Roles:
        - !Ref PipeLineRole

  CFDeployerRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: cloudformationdeployer-role
      ManagedPolicyArns:
        -  arn:aws:iam::aws:policy/AdministratorAccess
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - cloudformation.amazonaws.com
            Action:
              - sts:AssumeRole
      Path: /
```
*This complete template can be found in /templates/005_pipeline.yaml*

Now that we have that out of our system, let's go ahead and create our Pipeline. This is going to be a big resource so we'll start the deployment and I'll discuss the parts while it's deploying.

```yaml
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    DependsOn:
      - PipelinePolicy
    Properties:
      ArtifactStore:
        Type: S3
        Location: !Ref S3Bucket
      RoleArn: !GetAtt PipeLineRole.Arn
      # Name: !Ref AWS::StackName
      Stages:
        - Name: Source
          Actions:
            - Name: App
              ActionTypeId:
                Category: Source
                Owner: AWS
                Version: 1
                Provider: CodeCommit
              Configuration:
                RepositoryName: !Ref ProjectName
                BranchName: master
              OutputArtifacts:
                - Name: SCCheckoutArtifact
              RunOrder: 1
              RoleArn: !GetAtt PipeLineRole.Arn
        - Name: Build
          Actions:
          - Name: Build
            ActionTypeId:
              Category: Build
              Owner: AWS
              Version: 1
              Provider: CodeBuild
            Configuration:
              ProjectName: !Ref BuildProject
            RunOrder: 1
            InputArtifacts:
              - Name: SCCheckoutArtifact
            OutputArtifacts:
              - Name: BuildOutput
        - Name: DeployToTest
          Actions:
            - Name: LambdaCreateChangeSetTest
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Version: 1
                Provider: CloudFormation
              Configuration:
                ChangeSetName: my-app
                ActionMode: CHANGE_SET_REPLACE
                StackName: my-app
                Capabilities: CAPABILITY_NAMED_IAM
                TemplatePath: BuildOutput::output.yaml
                # TemplateConfiguration: "BuildOutput::preProd.json"
                RoleArn: !GetAtt CFDeployerRole.Arn
              InputArtifacts:
                - Name: BuildOutput
              RunOrder: 1
              RoleArn: !GetAtt PipeLineRole.Arn
            - Name: LambdaDeployChangeSetTest
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Version: 1
                Provider: CloudFormation
              Configuration:
                ChangeSetName: my-app
                ActionMode: CHANGE_SET_EXECUTE
                StackName: my-app
                RoleArn: !GetAtt CFDeployerRole.Arn
              InputArtifacts:
                - Name: BuildOutput
              RunOrder: 2
              RoleArn: !GetAtt PipeLineRole.Arn
```
*This complete template can be found in /templates/006_pipeline.yaml*

<!-- DevOpsDays Boston has a very long rant about Pipelines here -->

Great!!!! Now we have a pipeline to nowhere. It'll fail to build because it's trying to deploy an empty respository. Let's fix that.

# Creating Our Application

```bash
$ cd [wherever you want to work]
$ sam init --runtime python3.7 -n MyDevOpsApp
$ cd MyDevOpsApp/
$ git init
 Initialized empty Git repository in /Users/rboyd/dev/DevOpsDaysBoston/MyDevOpsApp/.git/
$ git remote add origin https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDevOpsProject 
$ git add .
$ git commit -m "first commit"
 [master (root-commit) 29dded5] first commit
  9 files changed, 671 insertions(+)
  create mode 100644 .gitignore
  create mode 100644 README.md
  create mode 100644 event.json
  create mode 100644 hello_world/__init__.py
  create mode 100644 hello_world/app.py
  create mode 100644 hello_world/requirements.txt
  create mode 100644 template.yaml
  create mode 100644 tests/unit/__init__.py
  create mode 100644 tests/unit/test_handler.py
$ git push # I know this will fail, it's easier than remembering the right command.
 fatal: The current branch master has no upstream branch.
 To push the current branch and set the remote as upstream, use

    git push --set-upstream origin master

$ git push --set-upstream origin master

 'NoneType' object has no attribute 'secret_key'
 Username for 'https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDevOpsProject': MyRandomUserName-at-121702220927
 Password for 'https://MyRandomUserName-at-121702220927@git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDevOpsProject': 
 Enumerating objects: 13, done.
 Counting objects: 100% (13/13), done.
 Delta compression using up to 8 threads
 Compressing objects: 100% (10/10), done.
 Writing objects: 100% (13/13), 7.83 KiB | 2.61 MiB/s, done.
 Total 13 (delta 1), reused 0 (delta 0)
 To https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDevOpsProject
  * [new branch]      master -> master
 Branch 'master' set up to track remote branch 'master' from 'origin'.
```

# Deploying Our Application

lol, jk. You don't have to do anything to deploy your application. Like all good DevOps practitioners, your deployment worflow starts with `git push`. Go ahead and check out your CodePipeline and see how your deployment is going.

# Making A Change

This is great, but it introduces the change with a big bang which may not be ideal. Let's update our application's template to make the change a bit more gradual. 

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  MyDevOpsApp

  Sample SAM Template for MyDevOpsApp

Globals:
  Function:
    Timeout: 3

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: python3.7
      Timeout: 30
      AutoPublishAlias: live
      DeploymentPreference:
        Type: Linear10PercentEvery1Minute
        Hooks:
          PreTraffic: !Ref preTrafficHook
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /hello
            Method: get

  preTrafficHook:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ops/
      Handler: pre_traffic.lambda_handler
      Runtime: python3.7
      Timeout: 45
      Policies:
        - Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
              - "codedeploy:PutLifecycleEventHookExecutionStatus"
            Resource:
              !Sub 'arn:aws:codedeploy:${AWS::Region}:${AWS::AccountId}:deploymentgroup:${ServerlessDeploymentApplication}/*'
        - Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
              - "lambda:InvokeFunction"
            Resource: !Ref HelloWorldFunction.Version

      # FunctionName: 'CodeDeployHook_preTrafficHook'
      DeploymentPreference:
        Enabled: false
      Timeout: 5
      Environment:
        Variables:
          USER_NAME: "RHBOYD"
          PASSWORD: "password123"

Outputs:
  HelloWorldApi:
    Description: "API Gateway endpoint URL for Prod stage for Hello World function"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello/"
  HelloWorldFunction:
    Description: "Hello World Lambda Function ARN"
    Value: !GetAtt HelloWorldFunction.Arn
  HelloWorldFunctionIamRole:
    Description: "Implicit IAM Role created for Hello World function"
    Value: !GetAtt HelloWorldFunctionRole.Arn

```

# Testing