+++
fragment = "content"
weight = 100

title = "Automating CI/CD pipeline creation with AWS"
background = "light"

[sidebar]
  sticky = true
+++
A simple way to create blueprints for your microservice and your deployment pipeline.
<!--more-->


### Problem

With advent of microsevices, we create many smaller logical chunks and are kept either in mono repo or individual repos. Because of the sheer number of services, It is difficult to manage the commonalities and bring some standards whether it is deployment or code structure or code reuse. This is an effort to solve the problem of creating blueprints so that all your services share common structure and deployment approaches. Imagine, you have three microservices in three different repo and all of them have CI/CD pipelines to deploy to your cloud infrastructure. If you are the only one person in the team, You might organize them similarly. When you have multiple people working on the team and many microservices created during the development, you need to create a process which can easily be used by any of your developer to create a repo with the skeleton structure, create a pipeline so that CI/CD happens as your developer merges PR.

I assume you have some familiarity with AWS Services. 

### Solution Architecture
<img align="right" width="400" height="400" src="cicd_aws.png">

We will be creating an [AWS Service Catalog](https://aws.amazon.com/servicecatalog/) product. This product will let your developer to spin up an [AWS Cloudformation](https://aws.amazon.com/cloudformation/) stack which will then create a source control repository in [AWS CodeCommit](https://aws.amazon.com/codecommit/) with the skeleton code from [S3](https://aws.amazon.com/s3/) bucket. It will also create a [AWS CodeBuild](https://aws.amazon.com/codebuild/) project which will automatically be invoked as the developer merges code to the repo. You will also be creating a [AWS Lambda](https://aws.amazon.com/lambda/) function separately that will trigger the codebuild job.


#### Trigger

Create a lambda function with the following code. This lambda will trigger the codebuild as soon as a pull request is merged into the repository.
```
const codebuild =  new AWS.CodeBuild();
exports.handler = async(event, context) => {
  let record = event.Records[0];
  let cb = record.customData;
  let reference = record.codecommit.references[0];
  let env = "stage";
  if(reference.ref === 'refs/head/master')
    env = "prod";

  
  let params = {
    projectName: cb,
    environmentVariablesOverride:[{
      name: "DEPLOYMENT_ENVIRONMENT",
      value: env
    }],
    sourceVersion: reference.commit

  };
  await codebuild.startBuild(params).promise();
}
```
### Blueprint

Create a base code for your microservice. Consider, you use serverless to create your lambda function and your team uses nodeJS as the language, You can create a following base code
```
src
├── index.js
|── package.json
|── Serverless.yml
|── test
|   ├── index.test.js
buildspec.yml
env.prod.yml
env.dev.yml
.gitignore
README.md

```
create a zip of your base code and upload to the S3 bucket. We will need the location of the S3 in the template we are going to create.

### Cloudformation Templates

We are going to create a cloudformation template with the following resources
#### Codecommit
```
CodeCommitRepo:
  Type: AWS::CodeCommit::Repository
  Properties:
    RepositoryName: !Ref CodeCommitRepoName
    RepositoryDescription: !Sub 'Repo created through Service catalog'
    Code:
      S3:
        Bucket: !Ref CodeBucket
        Key: !Ref CodePath
    Triggers:
    - Name: !Join ["-", [!Ref CodeCommitRepoName, !Ref BuildTriggerLambdaName]]
      CustomData: !Sub '${CodeCommitRepoName}-codebuild'
      DestinationArn: !Sub 'arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:${BuildTriggerLambdaName}'
      Branches:
      - Master
      - Stage
      Events:
      - updateReference
```
#### Create permission for CodeCommit to trigger lambda

```
CodeCommitRepoLambdaPermission:
  Type: AWS::Lambda::Permission
  DependsOn:
     - CodeCommitRepo
  Properties: 
    Action: lambda:InvokeFunction
    FunctionName: !Ref BuildTriggerLambdaName
    Principal: 'codecommit.amazonaws.com'
    SourceArn: !GetAtt CodeCommitRepo.Arn
```
#### Create a role and policy for CodeBuild to assume
```
  CodeBuildRole:
    Description: CodeBuild Service role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
        - Action: sts:AssumeRole
          Effect: Allow
          Principal:
            Service: codebuild.amazonaws.com
      Path: /
      RoleName: !Join
        - '-'
        - - !Ref 'AWS::StackName'
          - CodeBuild
    Type: AWS::IAM::Role
```
Create a policy with all the permission that would require for your codebuild and attach to the codebuild role.
#### Create a CodeBuild Project
```
  CodeBuildProject:
    Type: AWS::CodeBuild::Project
    DependsOn:
    - CodeBuildPolicy
    Properties:
      Artifacts:
        Type: NO_ARTIFACTS
      Description: !Join
        - ''
        - - 'CodeBuild Project for '
          - !Ref 'AWS::StackName'
      Environment:
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/nodejs:8.11.0
        Type: LINUX_CONTAINER
        EnvironmentVariables:
          - Name: DEPLOYMENT_ENVIRONMENT
            Type: PlainText
            Value: !Ref 'Environment'
      Name: !Sub '${CodeCommitRepoName}-codebuild'
      ServiceRole: !Ref 'CodeBuildRole'
      Source:
        Type: CODECOMMIT
        Location: !GetAtt [CodeCommitRepo, CloneUrlHttp]
        BuildSpec: 'buildspec.yml'
```
### Product in AWS Service Catalog

Create your product in AWS Service catalog and add it to a portfolio. Refer the documentation [here](https://docs.aws.amazon.com/servicecatalog/latest/adminguide/productmgmt-cloudresource.html).
Make sure your version of the product is active and portfolio is shared with the IAM users. You can do it from the portfolio panel of AWS Service Catalog.
IAM users can login and view the product and launch the product.
When they launch the product, It creates all the resources outlined above.

### References

- [AWS Samples](https://github.com/aws-samples/aws-codebuild-samples)
