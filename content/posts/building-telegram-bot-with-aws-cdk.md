+++ 
draft = false
date = 2024-09-23T23:20:23+03:00
title = "Building a Telegram Bot with AWS CDK: Lambda, API Gateway, and DynamoDB"
tags = ["AWS", "CDK", "Telegram", "Lambda", "API Gateway", "DynamoDB"]
categories = ["Software Development", "Devops"]
+++

In this post, we'll walk through the process of creating a Telegram bot using AWS CDK, Lambda, API Gateway, and DynamoDB. We'll focus on setting up the infrastructure and handling webhook requests from Telegram.

## Getting Your Telegram Bot Token

Before we start, you'll need to create a Telegram bot and obtain its token. Here's how:

1. Open Telegram and search for the "BotFather" bot.
2. Start a chat with BotFather and send the command `/newbot`.
3. Follow the prompts to choose a name and username for your bot.
4. Once created, BotFather will provide you with a token. This is your `TELEGRAM_BOT_TOKEN`.

Keep this token secure, as it allows control of your bot.

## Setting Up the Infrastructure

First, let's set up our AWS CDK stack to create the necessary resources. We'll need a Lambda function to handle the webhook requests, an API Gateway to expose our Lambda function, and a DynamoDB table to store chat IDs.

Here's the CDK code to set up these resources:

{{< highlight typescript >}}
import * as cdk from 'aws-cdk-lib';
import * as lambdaPython from '@aws-cdk/aws-lambda-python-alpha';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

export class TelegramBotStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Create DynamoDB table for storing chat IDs
    const chatIdsTable = new dynamodb.Table(this, 'ChatIdsTable', {
      partitionKey: { name: 'username', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
    });

    // Create Lambda function to handle webhook
    const webhookHandler = new lambdaPython.PythonFunction(this, 'WebhookHandler', {
      entry: 'lambda',  // directory containing your Python code
      runtime: cdk.aws_lambda.Runtime.PYTHON_3_9,
      handler: 'index.handler',
      environment: {
        TELEGRAM_BOT_TOKEN: process.env.TELEGRAM_BOT_TOKEN || '',
        CHAT_IDS_TABLE_NAME: chatIdsTable.tableName,
      },
    });

    // Grant the Lambda function read/write permissions to the DynamoDB table
    chatIdsTable.grantReadWriteData(webhookHandler);

    // Create API Gateway
    const api = new apigateway.RestApi(this, 'TelegramBotApi', {
      restApiName: 'Telegram Bot API',
    });

    // Create API Gateway resource and method
    const webhook = api.root.addResource('webhook');
    webhook.addMethod('POST', new apigateway.LambdaIntegration(webhookHandler));
  }
}
{{</ highlight >}}

This code creates a DynamoDB table for storing chat IDs, a Lambda function to handle webhook requests, and an API Gateway to expose the Lambda function.

## Handling Webhook Requests and Dependencies

Now, let's create the Lambda function to handle webhook requests from Telegram. We'll use Python for this example. Create a file named `index.py` in the `lambda` directory:

{{< highlight python >}}
import os
import json
import boto3
from telebot import TeleBot

TELEGRAM_BOT_TOKEN = os.environ['TELEGRAM_BOT_TOKEN']
CHAT_IDS_TABLE_NAME = os.environ['CHAT_IDS_TABLE_NAME']

bot = TeleBot(TELEGRAM_BOT_TOKEN)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(CHAT_IDS_TABLE_NAME)

def handler(event, context):
    try:
        body = json.loads(event['body'])
        message = body['message']
        chat_id = message['chat']['id']
        username = message['from']['username']
        text = message.get('text', '')

        if text.startswith('/start'):
            # Save the chat ID to DynamoDB
            table.put_item(Item={
                'username': username,
                'chat_id': str(chat_id)
            })
            bot.send_message(chat_id, "Welcome! Your chat ID has been saved.")
        else:
            bot.send_message(chat_id, "Hello! I received your message.")

        return {
            'statusCode': 200,
            'body': json.dumps('OK')
        }
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps('Error processing webhook')
        }
{{</ highlight >}}

To handle dependencies like `telebot`, we need to create a `requirements.txt` file in the same `lambda` directory. This file should contain all the external libraries our Lambda function needs:

{{< highlight text >}}
telebot==4.0.0
boto3==1.18.0
{{</ highlight >}}

The `aws-lambda-python-alpha` construct will automatically install these dependencies when deploying the Lambda function. This approach ensures that all necessary libraries are available in the Lambda execution environment.

## Deploying the Stack

Before we can set up the webhook, we need to deploy our stack to get the API Gateway URL. To deploy your Telegram bot stack, use the following CDK commands:

```
cdk synth
cdk deploy
```

Make sure you have set the `TELEGRAM_BOT_TOKEN` environment variable before deploying.

After successful deployment, CDK will output the API Gateway URL. You'll need this URL for the next step.

## Setting Up the Webhook

Now that we have our API Gateway URL, we can set up the webhook for our Telegram bot. To receive updates from Telegram, you need to make a request to the Telegram Bot API:

```
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=<YOUR_API_GATEWAY_URL>
```

Replace `<YOUR_BOT_TOKEN>` with your Telegram bot token and `<YOUR_API_GATEWAY_URL>` with the URL of your API Gateway endpoint that you received after deploying the stack.

You can make this request using a web browser or a tool like curl. After setting the webhook, Telegram will start sending updates to your Lambda function through the API Gateway.

## Conclusion

In this post, we've covered how to create a Telegram bot using AWS CDK, Lambda, API Gateway, and DynamoDB. We set up the infrastructure, implemented the webhook handler, deployed the stack, and configured the Telegram webhook. This setup provides a scalable and serverless solution for handling Telegram bot interactions. 

Remember to secure your API Gateway endpoint and handle errors gracefully in your Lambda function. You may also want to add authentication to your bot to ensure only authorized users can interact with it.

You can extend this basic example to add more complex functionality, such as command handling, message broadcasting, or integrating with other AWS services.
