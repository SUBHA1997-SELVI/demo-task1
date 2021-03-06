# demo-task1
AWSTemplateFormatVersion: "2010-09-09"
Description: Serverless deployment pipeline for Terraform projects
Parameters:
  GithubOauthToken:
    Type: String
    Description: see http://docs.aws.amazon.com/codepipeline/latest/userguide/integrations-action-type.html for instructions
  GithubRepoOwner:
    Type: String
    Description: The Github owner of the repository
  GithubRepoName:
    Type: String
    Description: The GitHub repository where the Terraform files (to be executed) are located
  GithubRepoBranch:
    Type: String
    Default: master
    Description: The Git branch to be used
  TerraformVersion:
    Type: String
    Default: 0.8.7
    Description: The Terraform version to use
  TerraformSha256:
    Type: String
    Default: 7ca424d8d0e06697cc7f492b162223aef525bfbcd69248134a0ce0b529285c8c
    Description: HASHICORP - Y U NO PACKAGE REPOSITORY
Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: Source Code Repository
        Parameters:
          - GithubRepoOwner
          - GithubRepoName
          - GithubRepoBranch
          - GithubOauthToken
      - Label:
          default: Terraform
        Parameters:
          - TerraformVersion
          - TerraformSha256
Resources:
  TerraformStateBucket:
    Type: AWS::S3::Bucket
    Properties:
      VersioningConfiguration:
        Status: Enabled
  ArtifactStoreBucket:
    Type: AWS::S3::Bucket
    Properties:
      VersioningConfiguration:
        Status: Enabled
      AccessControl: BucketOwnerFullControl
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      RoleArn: !GetAtt PipelineRole.Arn
      ArtifactStore:
        Location: !Ref ArtifactStoreBucket
        Type: S3
      Stages:
        - Name: Source
          Actions:
            - InputArtifacts: []
              Name: Source
              ActionTypeId:
                Category: Source
                Owner: ThirdParty
                Version: 1
                Provider: GitHub
              OutputArtifacts:
                - Name: SourceOutput
              Configuration:
                Owner: !Ref GithubRepoOwner
                Repo: !Ref GithubRepoName
                Branch: !Ref GithubRepoBranch
                OAuthToken: !Ref GithubOauthToken
              RunOrder: 1
        - Name: InvokeTerraform
          Actions:
          - Name: InvokeTerraformAction
            ActionTypeId:
                Category: Build
                Owner: AWS
                Version: 1
                Provider: CodeBuild
            OutputArtifacts:
              - Name: InvokeTerraformOutput
            InputArtifacts:
              - Name: SourceOutput
            Configuration:
                ProjectName: !Ref InvokeTerraformBuild
            RunOrder: 1
  PipelineRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          Effect: Allow
          Principal:
            Service: codepipeline.amazonaws.com
          Action: sts:AssumeRole
      Path: /
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AdministratorAccess

  InvokeTerraformBuild:
    Type: AWS::CodeBuild::Project
    Properties:
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/eb-go-1.5-amazonlinux-64:2.1.3
        Type: LINUX_CONTAINER
      Name: !Sub ${AWS::StackName}-InvokeTerraformBuild
      ServiceRole: !Ref InvokeTerraformBuildRole
      Source:
        Type: CODEPIPELINE
        BuildSpec: !Sub |
          version: 0.1
          phases:
            install:
              commands:
                - yum -y install jq
                - cd /tmp && curl -o terraform.zip https://releases.hashicorp.com/terraform/${TerraformVersion}/terraform_${TerraformVersion}_linux_amd64.zip && echo "${TerraformSha256} terraform.zip" | sha256sum -c --quiet && unzip terraform.zip && mv terraform /usr/bin
            build:
              commands:
                - terraform remote config -backend=s3 -backend-config="bucket=${TerraformStateBucket}" -backend-config="key=terraform.tfstate"
                - terraform apply
  InvokeTerraformBuildRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          Effect: Allow
          Principal:
            Service: codebuild.amazonaws.com
          Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/PowerUserAccess
