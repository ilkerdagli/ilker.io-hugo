---
title: 'Deploying a Next.js 14 App to AWS Amplify using CDK'
date: 2024-03-06T23:49:14+03:00
draft: false
tags: ['aws', 'cdk', 'amplify', 'nextjs', 'nextjs14', 'github']
categories: ['Software Development', 'Devops']
---

In this blog post, I'll guide you through deploying a Next.js 14 app to AWS Amplify using CDK. 
Please note that as of writing this post, AWS Amplify does not officially support Next.js 14.

While I personally believe that [Vercel](https://vercel.com "Vercel") is a better solution for now, 
I encountered challenges deploying my app to Amplify. This post aims to help others who might face similar issues.

Before we begin, ensure you have a GitHub access token obtained from the
[GitHub Settings Page](https://github.com/settings/tokens "GitHub Settings"). Store the token securely 
in AWS Secrets Manager.

Let's start by creating an IAM role and a managed policy for use in the App construct of _aws_amplify_alpha_.

{{< highlight typescript >}}
const role = new cdk.aws_iam.Role(this, 'AmplifyRole', {
    assumedBy: new cdk.aws_iam.ServicePrincipal('amplify.amazonaws.com'),
    description: 'CDK Amplify Role',
})

const managedPolicy = cdk.aws_iam.ManagedPolicy.fromAwsManagedPolicyName(
    'AdministratorAccess',
);

role.addManagedPolicy(managedPolicy)
{{</ highlight >}}

Now, let's create an Amplify App. Key details are explained below.

{{< highlight typescript >}}
const amplifyApp = new aws_amplify_alpha.App(this, 'MyApp', {
    description: 'My App',
    sourceCodeProvider: new aws_amplify_alpha.GitHubSourceCodeProvider({
        owner: 'ilkerdagli',
        repository: 'myApp',
        oauthToken: cdk.SecretValue.secretsManager('GithubAccessToken'),
    }),
    customRules: [
        {
            source: '/<*>',
            target: '/index.html',
            status: aws_amplify_alpha.RedirectStatus.NOT_FOUND_REWRITE,
        },
    ],
    environmentVariables: {
        "AMPLIFY_MONOREPO_APP_ROOT": "frontend",
        "AMPLIFY_DIFF_DEPLOY": "false",
        "_CUSTOM_IMAGE": "public.ecr.aws/docker/library/node:20.11",
        "NEXT_PUBLIC_AWS_AMPLIFY_REGION": "eu-central-1",
    },
    buildSpec: cdk.aws_codebuild.BuildSpec.fromObjectToYaml({
        version: '1.0',
        applications: [
            {
                appRoot: 'frontend',
                frontend: {
                    phases: {
                        preBuild: {
                            commands: ['yarn install --frozen-lockfile'],
                        },
                        build: {
                            commands: ['env | grep -e NEXT_PUBLIC_ >> .env.production', 'yarn run build'],
                        },
                    },
                    artifacts: {
                        baseDirectory: '.next',
                        files: ['**/*'],
                    },
                    cache: {
                        paths: ['node_modules/**/*', '.next/cache/**/*'],

                    },
                },
            }
        ]
    }),
    role: role,
})

const devBranch = amplifyApp.addBranch('dev', {
    description: 'dev branch',
});
{{</ highlight >}}

My app was located in the __frontend__ directory of my GitHub repository. 
Therefore, the environment variable _AMPLIFY_MONOREPO_APP_ROOT_ is set to _frontend_, 
which must match the buildSpec's applications __appRoot__ definition.

Since Amplify's default build image doesn't support Next.js 14 due to an older node version, 
we use a custom image. Set the _CUSTOM_IMAGE_ environment variable accordingly.

Additionally, we add a _dev_ branch to the app for automatic redeployment upon each push.

To complete the setup, we need run the following commands to run the app on Amplify:

{{< highlight bash >}}
aws amplify update-app --app-id APPID --platform WEB_COMPUTE 
aws amplify update-branch --app-id APPID --branch-name dev --framework 'Next.js - SSR'
{{</ highlight >}}

However, for automation, consider using CDK custom resources as demonstrated below:

{{< highlight typescript >}}
new cdk.custom_resources.AwsCustomResource(this, 'AmplifyExtraSettings', {
    onUpdate: {
        service: 'Amplify',
        action: 'updateApp',
        parameters: {
            appId: amplifyApp.appId,
            platform: 'WEB_COMPUTE',
        },
        physicalResourceId: cdk.custom_resources.PhysicalResourceId.of(Date.now().toString()),
    },

    policy: cdk.custom_resources.AwsCustomResourcePolicy.fromSdkCalls({
        resources: [amplifyApp.arn],
    }),

});

new cdk.custom_resources.AwsCustomResource(this, 'AmplifyUpdateDevBranch', {
    onUpdate: {
        service: 'Amplify',
        action: 'updateBranch',
        parameters: {
            appId: amplifyApp.appId,
            branchName: 'dev',
            framework: 'Next.js - SSR',
        },
        physicalResourceId: cdk.custom_resources.PhysicalResourceId.of(Date.now().toString()),
    },
    policy: cdk.custom_resources.AwsCustomResourcePolicy.fromSdkCalls({
        resources: [amplifyApp.arn, devBranch.arn],
    }),
});
{{</ highlight >}}

