--- 
title: "Compiling and Deploying Smart Contract with AWS Lambda"
date: 2023-07-11T09:08:14+03:00
draft: false
tags: ["aws", "cdk", "python", "lambda", "smart contract", "solc", "py-solcx", "solidity"]
categories: ["Software Development", "Devops"]
---

In this post, we'll walk through the process of compiling and deploying a smart contract on 
Ethereum's Sepolia Network. We'll use Infura as the node provider and handle our operations 
from an AWS Lambda function. We'll also be using the AWS Secrets Manager to store/retrieve Ethereum Private Key.

To start, we need to create a Lambda Python function through CDK to compile and deploy the contract.
One challenge I faced was installing solc, as it requires write permission to the $HOME path. 
I managed to bypass this issue by altering the HOME enviroment variable, as shown below:

{{< highlight typescript >}}
import * as lambdaPython from "@aws-cdk/aws-lambda-python-alpha"
import * as lambda from "aws-cdk-lib/aws-lambda"
import * as secretsmanager from "aws-cdk-lib/aws-secretsmanager"

const INFURA_ENDPOINT = "https://sepolia.infura.io/v3/YOUR_API_KEY"
const SECRETS_ARN = "YOUR_SECRETS_ARN_THAT_STORES_ETH_PRIVATE_KEY"

const secrets = secretsmanager.Secret.fromSecretCompleteArn(
  this,
  "secrets",
  SECRETS_ARN
)

const compileAndDeployContractLambda = new lambdaPython.PythonFunction(
  this,
  "CompileAndDeployContractLambda",
  {
    entry: path.join(
      __dirname,
      "lambdaFunctions",
      "python",
      "compile-and-deploy-contract"
    ),
    bundling: {
      assetExcludes: [".venv"],
    },
    environment: {
      SECRETS_ARN: secrets.secretFullArn!,
      INFURA_ENDPOINT: INFURA_ENDPOINT,
      HOME: "/tmp", //Installing solc requires write permission to $HOME
    },
    runtime: lambda.Runtime.PYTHON_3_9,
    memorySize: 256,
    architecture: lambda.Architecture.X86_64,
    timeout: Duration.seconds(900),
    logRetention: logs.RetentionDays.SIX_MONTHS,
  }
)
{{</ highlight >}}

Next, we'll create our handler function. By default, the PythonFunction of aws-lambda-python-alpha looks for an index.py file in the entry path unless another filename is specified. An important point to note here is that if a *requirements.txt* file is present in the entry path, the PythonFunction will install the necessary packages specified in the file during the bundling process. Here's what your *requirements.txt* might look like:

{{< highlight python >}}
web3==6.4.0
urllib3==1.26.15
py-solc-x==1.1.1
boto3==1.26.147
{{</ highlight >}}

Following that, we'll create our Lambda function in the *lambdaFunctions/python/compile-and-deploy-contract/index.py* path. Here's the code for that:

{{< highlight python >}}
import os
import boto3
import json
import logging
import traceback
from web3 import Web3
from eth_account import Account
from solcx import compile_source, install_solc

secretsArn = os.environ.get('SECRETS_ARN')
INFURA_ENDPOINT = os.environ.get('INFURA_ENDPOINT')

_solc_version = "0.8.17"
install_solc(_solc_version)
ethPrivKey = None
secretsClient = boto3.client('secretsmanager')

try:
    secrets = json.loads(secretsClient.get_secret_value(
        SecretId=secretsArn)['SecretString'])
    ethPrivKey = secrets['ETH_PRIV_KEY']
except Exception as e:
    logging.error('Cant retrieve secret from aws secrets manager:{}\n'.format(
        traceback.format_exc()))

def handler(event, context):
    contract = """
    // SPDX-License-Identifier: MIT
    // compiler version must be greater than or equal to 0.8.17 and less than 0.9.0
    pragma solidity ^0.8.17;

    contract HelloWorld {
        string public greet = "Hello World!";
    }
    """

    # Compile the contract
    compiledSolDict = compile_source(
        contract,
        output_values=['abi', 'bin'],
        solc_version=_solc_version,
        allow_paths='/opt'
    )

    compiledSolDict = compiledSolDict.get('<stdin>:HelloWorld')

    if compiledSolDict:
        # Deploy the contract
        binOutput = compiledSolDict['bin']
        abiOutput = compiledSolDict['abi']

        try:
            # Connect to the Ethereum network using Infura
            w3 = Web3(Web3.HTTPProvider(INFURA_ENDPOINT))

            # Create an account from the private key
            account = Account.from_key(ethPrivKey)

            # Set the default account for contract deployment
            w3.eth.default_account = account.address

            # Create a contract instance
            contract = w3.eth.contract(abi=abiOutput, bytecode=binOutput)

            # Build the transaction to deploy the contract
            transaction = contract.constructor().build_transaction({
                'nonce': w3.eth.get_transaction_count(account.address),
            })

            # Estimate the gas required for contract deployment
            gas_estimate = w3.eth.estimate_gas(transaction)

            # Update the transaction with the estimated gas
            transaction['gas'] = gas_estimate

            # Sign the transaction with the private key
            signed_transaction = account.sign_transaction(transaction)

            # Send the signed transaction to the network
            tx_hash = w3.eth.send_raw_transaction(
                signed_transaction.rawTransaction)

            # Wait for the transaction to be mined
            tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

            # Retrieve the contract address from the transaction receipt
            contractAddress = tx_receipt['contractAddress']
            logging.info(
                "Deployed contract address: {}".format(contractAddress))
        except:
            error = traceback.format_exc()
            logging.error('Failed to deploy contract:{}\n'.format(error))
{{</ highlight >}}

This code was designed to streamline contract deployments for a recent project. 
Unlike the code provided here, my project used separate Lambda function to compile 
Solidity files and store the outputs in S3. Another Lambda function then got triggered 
via S3's PUT Object event to deploy the compiled contracts. Another Lambda function 
-which can be another post in the future- also verifies the deployed contract on 
Etherscan using their API.

This code has been simplified for this post and is untested, so feel free to reach out 
if you encounter any issues.