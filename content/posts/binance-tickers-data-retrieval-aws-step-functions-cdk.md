+++ 
draft = false
date = 2023-07-26T14:41:23+03:00
title = "Fetch All Binance Tickers' Price Data in Under 10 Seconds with AWS Step Functions and CDK"
tags = ["aws", "cdk", "step functions", "state machine", "go", "binance", "kline", "candlestick"]
categories = ["Software Development", "Devops"]
+++

For my personal project, I needed to fetch price data of all symbols for specific timeframes from 
Binance Futures API. After retrieving the data, I planned to apply some technical analysis, 
but that aspect won't be covered in this post.

As an avid AWS and CDK enthusiast, I decided to utilize AWS Step Functions for this project. 
To interact with the Binance API, I opted for the Go programming language. For the analysis part, 
I'll be using Python. It took around 5-8 seconds to fetch all tickers' candlesticks (klines) data.

Binance implements rate control based on IP addresses, even when using API keys. Since AWS Lambda 
functions may execute with different IPs on each run (assuming that), I didn't encounter any 
issues with Binance's rate limit. However, for those who wish to guarantee that they won't exceed 
the rate limit, they can create a Virtual Private Cloud (VPC) and a NAT gateway inside it, 
attaching an Elastic IP to the NAT gateway. Then, they can associate the Lambdas with this VPC 
and implement a custom request throttling mechanism.

Let's take a look at the functions and the state machine generation code:

__fetch-tickers/main.go__
{{< highlight go >}}
package main

import (
	"context"
	"os"

	"github.com/adshao/go-binance/v2"
	"github.com/adshao/go-binance/v2/futures"
	"github.com/aws/aws-lambda-go/lambda"
)

var (
	BinanceAPIKey    = os.Getenv("BINANCE_API_KEY")
	BinanceAPISecret = os.Getenv("BINANCE_API_SECRET")
)

type Lambda struct {
	futuresClient *futures.Client
}

func (l *Lambda) Handler(ctx context.Context) (*[]string, error) {

	// Fetch futures tickers
	exchangeInfo, err := l.futuresClient.NewExchangeInfoService().Do(ctx)
	if err != nil {
		return nil, err
	}

	var tickers []string
	for _, symbol := range exchangeInfo.Symbols {
		if symbol.Status == "TRADING" && symbol.QuoteAsset == "USDT" && symbol.ContractType == "PERPETUAL" {
			tickers = append(tickers, symbol.Symbol)
		}
	}

	return &tickers, nil
}

func main() {
	fc := binance.NewFuturesClient(BinanceAPIKey, BinanceAPISecret)
	l := Lambda{
		futuresClient: fc,
	}
	lambda.Start(l.Handler)
}
{{</ highlight >}}

__fetch-ticker-data/main.go__
{{< highlight go >}}
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"

	"github.com/adshao/go-binance/v2"
	"github.com/adshao/go-binance/v2/futures"
	"github.com/aws/aws-lambda-go/lambda"
	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/s3"
)

var (
	TickerDataS3Bucket = os.Getenv("TICKER_DATA_BUCKET")
	BinanceAPIKey      = os.Getenv("BINANCE_API_KEY")
	BinanceAPISecret   = os.Getenv("BINANCE_API_SECRET")
)

type Lambda struct {
	futuresClient *futures.Client
	s3Client      *s3.Client
}

type Input struct {
	Ticker    string `json:"ticker"`
	Timeframe string `json:"timeframe"`
}

func (l *Lambda) Handler(ctx context.Context, i Input) error {
	log.Println(i.Ticker)
	klines, err := l.futuresClient.NewKlinesService().Symbol(i.Ticker).Interval(i.Timeframe).Do(ctx)

	if err != nil {
		log.Printf("Error fetching ticker %s: %v", i.Ticker, err)
		return err
	}

	serializedKLines, err := json.Marshal(klines)
	if err != nil {
		log.Printf("Can't serialize klines: %v", err)
		return err
	}

	_, err = l.s3Client.PutObject(ctx, &s3.PutObjectInput{
		Bucket: aws.String(TickerDataS3Bucket),
		Key:    aws.String(fmt.Sprintf("%s/%s.json", i.Ticker, i.Timeframe)),
		Body:   bytes.NewBuffer(serializedKLines),
	})

	if err != nil {
		return err
	}

	return nil
}

func main() {
	cfg, err := config.LoadDefaultConfig(context.TODO())
	if err != nil {
		panic(err)
	}

	fc := binance.NewFuturesClient(BinanceAPIKey, BinanceAPISecret)
	s3c := s3.NewFromConfig(cfg)
	l := Lambda{
		futuresClient: fc,
		s3Client:      s3c,
	}
	lambda.Start(l.Handler)
}
{{</ highlight >}}

CDK Script for the State Machine Definition:
{{< highlight typescript >}}
import * as cdk from "aws-cdk-lib";
import { Construct } from "constructs";
import { StackProps, StageProps } from "aws-cdk-lib";
import { Tables } from "./dynamodb-stack";
import { Buckets } from "./s3-stack";
import { aws_stepfunctions as sfn } from "aws-cdk-lib";
import { aws_stepfunctions_tasks as sfn_tasks } from "aws-cdk-lib";
import * as lambda from 'aws-cdk-lib/aws-lambda'
import * as lambdago from '@aws-cdk/aws-lambda-go-alpha'
import { DefinitionBody, JsonPath } from "aws-cdk-lib/aws-stepfunctions";
import { aws_logs as logs } from "aws-cdk-lib";
import { Duration, RemovalPolicy } from 'aws-cdk-lib'
import * as path from 'path'
import { aws_events as eventbridge } from 'aws-cdk-lib'
import { aws_events_targets as eventbridge_targets } from 'aws-cdk-lib'
import { aws_s3 as s3 } from "aws-cdk-lib"

export class FetchAndAnalyzeStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);


    const tickerDataBucket = new s3.Bucket(this, "TickerData", {
      publicReadAccess: false,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      accessControl: s3.BucketAccessControl.PRIVATE,
      objectOwnership: s3.ObjectOwnership.BUCKET_OWNER_ENFORCED,
      encryption: s3.BucketEncryption.S3_MANAGED,
      autoDeleteObjects: true,
    });

    const fetchTickerDataLambda = new lambdago.GoFunction(
      this,
      "FetchTickerDataLambda",
      {
        entry: path.join(__dirname, "lambdaFunctions", "go", "fetch-ticker-data"),
        environment: {
          BINANCE_API_KEY: "",
          BINANCE_API_SECRET: "",
          TICKER_DATA_BUCKET: tickerDataBucket.bucketName,
        },
        memorySize: 128,
        architecture: lambda.Architecture.ARM_64,
        timeout: Duration.seconds(900),
        logRetention: logs.RetentionDays.THREE_MONTHS,
      },
    );
    tickerDataBucket.grantReadWrite(fetchTickerDataLambda);

    const fetchTickersLambda = new lambdago.GoFunction(this, "FetchTickersLambda", {
      entry: path.join(__dirname, "lambdaFunctions", "go", "fetch-tickers"),
      environment: {
        BINANCE_API_KEY: "",
        BINANCE_API_SECRET: "",
      },
      memorySize: 128,
      architecture: lambda.Architecture.ARM_64,
      timeout: Duration.seconds(900),
      logRetention: logs.RetentionDays.THREE_MONTHS,
    });

    const sMLogGroup = new logs.LogGroup(this, 'FetchAndAnalyzeSMLogs', {
      retention: logs.RetentionDays.ONE_DAY,
      removalPolicy: RemovalPolicy.DESTROY,
    });

    const fetchAndAnalyzeSM = new sfn.StateMachine(this, "FetchAndAnalyzeSM", {
      stateMachineType: sfn.StateMachineType.EXPRESS,
      logs: {
        destination: sMLogGroup,
        level: sfn.LogLevel.ALL,
        includeExecutionData: true,
      },
      definitionBody: DefinitionBody.fromChainable(
        new sfn_tasks.LambdaInvoke(this, "FetchTickers", {
          lambdaFunction: fetchTickersLambda,
          resultPath: "$.tickers",
          payloadResponseOnly: true,
        }).next(
          new sfn.Map(this, "DistributeTickers", {
            itemsPath: "$.tickers",
            parameters: {
              "ticker.$": "$$.Map.Item.Value",
              "timeframe.$": "$.timeframe",
            },
            maxConcurrency: 100,
          }).iterator(
            new sfn_tasks.LambdaInvoke(this, "FetchTickerData", {
              lambdaFunction: fetchTickerDataLambda,
              resultPath: JsonPath.DISCARD,
            }).addCatch(new sfn.Pass(this, "FetchTickerDataFailPass")),
            //   .next(new sfn_tasks.LambdaInvoke(this, "ScanForPatterns", {
            //     lambdaFunction: analyzeTickerDataLambda,
            //     resultPath: JsonPath.DISCARD,
            //   })
            // )
          ),
        ),
      ),
    });
  }
}
{{</ highlight >}}
Let me explain some aspects of my state machine definition. I chose the "Express" type because 
its execution time is less than five minutes, and we make 194 state transitions in each execution 
with the current ticker list. Assuming we run this machine every 5 minutes, it will perform 
60/5 * 194, resulting in 2328 transitions per hour. The pricing for standard state machines is 
based on state transitions, but for "Express" type, it's calculated based on the execution count,
 making it more cost-efficient in our case.

In the "Map" task, I set the *maxConcurrency* to 100. Initially, my account's maximum concurrent 
Lambda limit was 10, but I requested an increase and updated it to 1000. Before increasing the 
limit, I had set *maxConcurrency* to 5, which caused the task to take around 25-30 seconds. I also 
used *.addCatch* to catch any errors that occur in the *fetchTickerDataLambda* function. 
Without this, any error during data fetching would have resulted in the state machine halting 
with a "failed" state.

As you can see, I've left the comment section for the last task, which involves scanning for patterns. 
You may choose to continue with this step or add additional steps depending on your requirements. 
For instance, you could incorporate a choice task to handle situations when a pattern is found or not.

__Bonus:__ An eventbridge rule to execute the state machine every 4 hour:
{{< highlight typescript >}}
const fetchAndAnalyzeEvery4HourRule = new eventbridge.Rule(
  this,
  "FetchAndAnalyzeEvery4HourRule",
  {
    schedule: eventbridge.Schedule.cron({
      minute: "0",
      hour: "*/4",
      day: "*",
      month: "*",
      year: "*",
    }),
    targets: [
      new eventbridge_targets.SfnStateMachine(fetchAndAnalyzeSM, {
        retryAttempts: 0,
        input: eventbridge.RuleTargetInput.fromObject({ timeframe: "4h" }),
      }),
    ],
  },
);
{{</ highlight >}}

Thank you for reading, and see you soon!