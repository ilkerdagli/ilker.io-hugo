---
title: "Building My Blog: Automating the Process with AWS-CDK"
date: 2023-07-06T15:24:41+03:00
draft: false
tags: ["aws", "cdk", "hugo", "s3", "cloudfront"]
categories: ["Devops"]
---

## Introduction

In this blog post, I will share the process of building my blog and how I automated it using
AWS-CDK (Amazon Web Services Cloud Development Kit). By utilizing AWS-CDK, I was able to
streamline the deployment of my blog, making it more efficient and convenient.
Let's dive into the step-by-step explanation of how I accomplished this.

## Building the Infrastructure

To generate and host my blog, I decided to leverage Hugo, a popular open-source static site generator.
However, instead of manually generating the site on my PC and uploading the files to a web server,
I opted for automation. Given my experience with AWS-CDK in previous projects, I wrote a script using
this tool to create a stack for my blog's infrastructure. You can find the full code for this process
on my [GitHub repository](https://github.com/ilkerdagli/ilker.io-cdk "GitHub repository"). Here are the key components of the stack:

#### S3 Bucket for Website Hosting

I created an S3 bucket to serve my blog's website files. 
This bucket acts as a reliable storage location for the static content of the blog.

{{< highlight typescript >}}
const websiteBucket = new s3.Bucket(this, `${domainName}-WebSiteBucket`, {
  publicReadAccess: false,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
})
{{< / highlight >}}

#### SSL Certificate for CloudFront Distribution

To ensure secure communication between the user and my blog, I obtained a SSL certificate. 
This certificate is used by the CloudFront distribution. If you will use a certificate in
cloudfront distribution, the certificate must be created in __us-east-1__ region.
Because of my stack is in __eu-central-1__ region, I made a seperate stack for certificate.

{{< highlight typescript >}}
import * as acm from "aws-cdk-lib/aws-certificatemanager"
import { Stack, StackProps } from "aws-cdk-lib"

import { Construct } from "constructs"

interface CertificateStackProps extends StackProps {
  domainNames: [string]
}

interface Certificate {
  domainName: string
  certificate: acm.Certificate
}

export class CertificateStack extends Stack {
  readonly certificates = new Map<string, acm.Certificate>()
  constructor(scope: Construct, id: string, props: CertificateStackProps) {
    super(scope, id, props)
    props.domainNames.forEach((domainName) => {
      this.certificates.set(
        domainName,
        new acm.Certificate(this, `${domainName}-Cert`, {
          domainName: domainName,
          validation: acm.CertificateValidation.fromDns(),
        })
      )
    })
  }
}
{{< / highlight >}}

#### CloudFront Distribution

I set up a CloudFront distribution, which acts as a content delivery network (CDN) for my blog.
This allows for faster content delivery to users across different geographic locations.
Because of hugo generates the static html files we need a simple lambda edge function 
that appends index.html to origin request.
And because of we will use this function on CloudFront Distribution this function 
also needs to be on different stack to deploy on __us-east-1__ region like certificate.
We also need an OAI(Origin Access Identity) that has permission to read the S3 bucket that we created earlier.

Here is the lambda edge stack:
{{< highlight typescript >}}
import * as lambda from "aws-cdk-lib/aws-lambda"
import * as cloudfront from "aws-cdk-lib/aws-cloudfront"
import { Stack, StackProps } from "aws-cdk-lib"
import * as path from "path"

import { Construct } from "constructs"

interface LambdaEdgeStackProps extends StackProps {}

export interface LambdaEdgeFunctions {
  defaultIndexLambda: cloudfront.experimental.EdgeFunction
}

export class LamdaEdgeStack extends Stack {
  readonly functions: LambdaEdgeFunctions
  constructor(scope: Construct, id: string, props: LambdaEdgeStackProps) {
    super(scope, id, props)
    this.functions = {
      defaultIndexLambda: new cloudfront.experimental.EdgeFunction(
        this,
        "DefaultIndexFunction",
        {
          runtime: lambda.Runtime.NODEJS_18_X,
          handler: "defaultIndex.handler",
          code: lambda.Code.fromAsset(path.join(__dirname, "lambdaFunctions")),
        }
      ),
    }
  }
}
{{< / highlight >}}

And here is the rest:

{{< highlight typescript >}}
const accessIdentity = new cloudfront.OriginAccessIdentity(this, "OAI")
websiteBucket.grantRead(accessIdentity)

const origin = new origins.S3Origin(websiteBucket, {
  originAccessIdentity: accessIdentity,
})

const cloudfrontDistribution = new cloudfront.Distribution(
  this,
  `${domainName}-CFDist`,
  {
    certificate: props?.certificates.get(domainName),
    domainNames: [domainName],
    defaultRootObject: "index.html",
    defaultBehavior: {
      origin: origin,
      edgeLambdas: [
        {
          functionVersion:
            props.lambdaEdgeFunctions.defaultIndexLambda.currentVersion,
          eventType: cloudfront.LambdaEdgeEventType.ORIGIN_REQUEST,
        },
      ],
      viewerProtocolPolicy:
        cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    },
  }
)

{{< / highlight >}}

#### CodeCommit Git Repository

For version control and collaboration, I created a CodeCommit Git repository.
This repository stores the source code and allows me to track changes made to the blog over time.

{{< highlight typescript >}}
const repo = new codecommit.Repository(
  this,
  `${domainName}-CodeCommitRepository`,
  {
    repositoryName: `${domainName}-hugo`,
    description: `${domainName} website`,
  }
)
{{< / highlight >}}

#### CodeBuild Project for Hugo Site Generation

Using a CodeBuild project, I integrated the Hugo site generation process into my automation workflow.
This project automatically generates the website using Hugo whenever changes are made to the source code.

{{< highlight typescript >}}
const project = new codebuild.PipelineProject(this, "CodeBuildProject", {
  buildSpec: codebuild.BuildSpec.fromObject({
    version: "0.2",
    phases: {
      install: {
        commands: [
          "curl -Ls https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz -o /tmp/hugo.tar.gz",
          'echo "${HUGO_SHA256}  /tmp/hugo.tar.gz" | sha256sum -c -',
          "mkdir /tmp/hugo_${HUGO_VERSION}",
          "tar xf /tmp/hugo.tar.gz -C /tmp/hugo_${HUGO_VERSION}",
          "mv /tmp/hugo_${HUGO_VERSION}/hugo /usr/bin/hugo",
          "rm -rf /tmp/hugo*",
          'git config --global credential.helper "!aws codecommit credential-helper $@"',
          "git config --global credential.UseHttpPath true",
          "git init",
          `git remote add origin ${repo.repositoryCloneUrlHttp}`,
          "git fetch",
          "git checkout -f -t origin/main",
          "git submodule init",
          "git submodule update --recursive",
        ],
      },
      build: {
        commands: ["hugo"],
      },
    },
    artifacts: {
      files: ["**/*"],
      "base-directory": "public",
      name: "$(AWS_REGION)-$(date +%Y-%m-%d)",
    },
  }),
  environmentVariables: {
    HUGO_VERSION: { value: hugoVersion },
    HUGO_SHA256: { value: hugoSHA256 },
  },
})

repo.grantPull(project)
{{< / highlight >}}


#### CodePipeline for Continuous Deployment

To achieve continuous deployment, I set up a CodePipeline. This pipeline is triggered whenever new commits are pushed to the CodeCommit repository. It performs the following tasks:

- Builds the website using the CodeBuild project
- Uploads the generated static files to the S3 bucket
- Invalidates the CloudFront cache, ensuring that users see the latest version of the blog

{{< highlight typescript >}}
const buildOutput = new codepipeline.Artifact()
const buildAction = new codepipeline_actions.CodeBuildAction({
  actionName: "CodeBuild",
  project: project,
  input: sourceOutput,
  outputs: [buildOutput],
})

const deployAction = new codepipeline_actions.S3DeployAction({
  actionName: "S3Deploy",
  bucket: websiteBucket,
  input: buildOutput,
})

const invalidateBuildProject = new codebuild.PipelineProject(
  this,
  `InvalidateProject`,
  {
    buildSpec: codebuild.BuildSpec.fromObject({
      version: "0.2",
      phases: {
        build: {
          commands: [
            'aws cloudfront create-invalidation --distribution-id ${CLOUDFRONT_ID} --paths "/*"',
          ],
        },
      },
    }),
    environmentVariables: {
      CLOUDFRONT_ID: { value: cloudfrontDistribution.distributionId },
    },
  }
)

cloudfrontDistribution.grantCreateInvalidation(invalidateBuildProject)

const invalidateBuildAction = new codepipeline_actions.CodeBuildAction({
  actionName: "InvalidateBuild",
  project: invalidateBuildProject,
  input: buildOutput,
})

const pipeline = new codepipeline.Pipeline(this, "CodePipeline", {
  pipelineName: "HugoCodePipeline",
  stages: [
    {
      stageName: "CodeCommit",
      actions: [sourceAction],
    },
    {
      stageName: "Build",
      actions: [buildAction],
    },
    {
      stageName: "Deploy",
      actions: [deployAction],
    },
    {
      stageName: "CFInvalidate",
      actions: [invalidateBuildAction],
    },
  ],
})
{{< / highlight >}}


#### DNS 'A Record'

Finally, I created a DNS 'A record' that points to the CloudFront distribution.
This allows users to access my blog using a user-friendly domain name.

{{< highlight typescript >}}
new route53.ARecord(this, "ARecord", {
  recordName: domainName,
  target: route53.RecordTarget.fromAlias(
    new route53targets.CloudFrontTarget(cloudfrontDistribution)
  ),
  zone,
})
{{< / highlight >}}

Now I will add this post and push it to the CodeCommit repository that I created using CDK. 
Afterward, I won't make any further changes, and you will be able to read this post. 
I would also like to express my gratitude to Mike Apted, whose stack from four years ago served 
as an inspiration for my work. You can find his original stack on [his Github repository](https://github.com/mikeapted/aws-cdk-hugo-s3 "GitHub repository"). 
My implementation is an updated version of that stack.