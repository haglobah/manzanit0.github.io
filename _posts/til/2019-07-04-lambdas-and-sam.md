---
layout: post
title: "Deploying .NET Core applications to AWS lambda"
author: Javier Garcia
category: AWS
tags: aws, .net, c#
---

Today I learnt that you can deploy a lambda to AWS in different ways. The easiest one is probably simply uploading
the code via the AWS Console. I played around with SAM CLI and AWS Cloudformation, which is another way.

Quoted from [AWS docs](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html):
> The AWS Serverless Application Model (AWS SAM) is an open-source framework that you can use to build
> serverless applications on AWS.

To deploy a lambda and create all the necessary infrastructure via Cloudformation, it's necessary to create a
`template.yml` which outlines the infrastructure to create. This is what SAM will use when we tell it to build everything.

```yml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  OptimgFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: Optimg::Optimg.Functions::Get # Format referencing the lambda entry point – namespace::namespace.class::method
      Runtime: dotnetcore2.1
```

To install SAM CLI, you can do it via pip, but I prefer homebrew:

```
brew tap aws/tap
brew install aws-sam-cli
```

If you want to create a sample project, you can run: `sam init --runtime dotnet`. I actually encourage you to do it
even if it's simply to read through the resources/READMEs in the project. It has a lot of useful information.

To package the application you can run `sam package`. That simply adds the `CodeUri` property to the
code in `template.yml` and creates a new file, `package.yml` with it.

```
sam package \
  --template-file template.yml \
  --output-template-file package.yml \
  --s3-bucket optimizations-test
```

After `sam package`, `package.yml`:

```yml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  OptimgFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: Optimg::Optimg.Functions::Get
      Runtime: dotnetcore2.1
      CodeUri: s3://optimizations-test/715f1c9c1e10b233fa20614538ee5190
```

To deploy the lambda to AWS, assuming you have your profile already set up in `~/.aws/credentials`:

```
sam deploy \
  --template-file package.yml \
  --stack-name optimg-1 \
  --capabilities CAPABILITY_IAM
```

Lastly, to invoke the function locally, providing an event

```
sam build
sam local invoke "OptimgFunction" -e event.json
```

Some resources I read before compiling this were:

1. http://aws-cloud.guru/testing-lambdas-locally-with-aws-sam-cli/
2. https://github.com/aws/aws-lambda-dotnet
3. https://abhurtun.github.io/Blog/blogs/awsSamNetCore.html

The project I was working on for which I had to learn this was:

https://github.com/Manzanit0/Optimg