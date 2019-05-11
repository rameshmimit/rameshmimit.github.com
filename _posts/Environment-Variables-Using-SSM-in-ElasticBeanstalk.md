---
layout: post
title:  "Environment Variables Using SSM in ElasticBeanstalk"
author: Ramesh
categories: [ ElasticBeanstalk, Ebextensions, CustomResource ]
image: assets/images/elasticbeanstalk.png
hidden: true
---
####Introduction
At this moment (Aug 2018), Elastic Beanstalk does not support Systems Manager Parameter Store for environment variables although CloudFormation supports the Parameter Store as parameters under some particular conditions.

You can use the "Parameters" sections with SSM Parameter Types in a CloudFormation template (launched in Dec 2017), and Elastic Beanstalk supports the "Parameters" key in .ebextensions files. However, this does not support the "SecureString" type in CloudFormation at this moment (Aug 2018), and also Elastic Beanstalk does not support the SSM Parameter types .

Example ebextension:
```yaml
Parameters:
  SSMparameter1:
    Type: 'AWS::SSM::Parameter::Value<String>'
    Default: db-user-name

option_settings:
  aws:elasticbeanstalk:application:environment:
    DB_USER: '`{"Ref" : "SSMparameter1"}`'
```
This causes the following error when you deploy it into an Elastic Beanstalk environment:

```html
ERROR: InvalidParameterValue: Unknown Parameter Type: AWS::SSM::Parameter::Value<String>
```
Also, CloudFormation supports Dynamic References (launched in Aug 2018), which can be used with secure strings stored in AWS Systems Manager Parameter Store. However, this supports only specific resources and properties, so it is impossible to use the dynamic references in .ebextensions files at this moment.

This article shows how to use the Systems Manager Parameter Store as a source of environment variables in an Elastic Beanstalk environment.

Solution
This approach is to use a Lambda-backed Custom Resource in an .ebextensions file and refer the response data from the Lambda function as a source of environment variables in the Elastic Beanstalk environment. I created a sample Lambda function including a CloudFormation template for the convenience to provision it.

####Solution

* Create a Lambda function using a CloudFormation template attached in this article (GetParameterStoreKeyValuesByPath.yml) or manually create a Lambda function using the Lambda console or AWS CLI (GetParameterStoreKeyValuesByPath.py). This Lambda function can be used in any Elastic Beanstalk environments and also standalone CloudFormation stacks as a service token in custom resources.

* Create key-values parameters in the Parameter Store. You can use the default or custom KMS keys for secure strings. Please visit the below documentation link of Systems Manager Parameter Store for more information.

* Create an .ebextensions file and then deploy it.

Example .ebextensions file)

```yaml
Resources:
  ParameterStoreCustomResource:
    Type: Custom::ParameterStore
    Properties:
      ServiceToken: arn:aws:lambda:ap-southeast-2:123456789012:function:GetParameterStoreKeyValuesByPath
      parameterPath: "/ElasticBeanstalk/MyApplication/MyEnvironmentProd/"

option_settings:
  aws:elasticbeanstalk:application:environment:
    DB_HOST: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource", "production-db-host" ] }`'
    DB_NAME: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource", "production-db-name" ] }`'
    DB_USER: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource", "production-db-user" ] }`'
    DB_PASSWORD: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource", "production-db-password" ] }`'
    DB_TABLE: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource", "production-db-table" ] }`'
```
Now your application can use the above environment variables (i.e.: DB_HOST, DB_NAME, DB_USER, DB_PASSWORD, DB_TABLE).

In the Parameter Store, you should have the following keys for the above example:

```html
/ElasticBeanstalk/MyApplication/MyEnvironmentProd/production-db-host
/ElasticBeanstalk/MyApplication/MyEnvironmentProd/production-db-name
/ElasticBeanstalk/MyApplication/MyEnvironmentProd/production-db-user
/ElasticBeanstalk/MyApplication/MyEnvironmentProd/production-db-password
/ElasticBeanstalk/MyApplication/MyEnvironmentProd/production-db-table
```
You can encrypt values for the above keys in the Parameter Store using KMS keys. Please note that the service role in the Lambda function should have the "kms:Decrypt" permission if the values are encrypted.



Please note that even if you update the values in the Parameter Store, the values are not automatically reflected in the Elastic Beanstalk environment. You need to modify the logical resource name and resource name used by "Fn::GetAtt" intrinsic function in the .ebextensions file and then make a deployment  to reflect the new values into instances. In addition, if you would like to add new environment variables, you also need to modify the logical resource name and then deploy it to reflect them.

Example) Change from the "ParameterStoreCustomResource" to "ParameterStoreCustomResource20180818" so that CloudFormation replaces the resource, which means invoking the Lambda function again.

```yaml
Resources:
  ParameterStoreCustomResource20180818:
    Type: Custom::ParameterStore
    Properties:
      ServiceToken: arn:aws:lambda:ap-southeast-2:123456789012:function:GetParameterStoreKeyValuesByPath
      parameterPath: "/ElasticBeanstalk/MyApplication/MyEnvironmentProd/"

option_settings:
  aws:elasticbeanstalk:application:environment:
    DB_HOST: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource20180818", "production-db-host" ] }`'
    DB_NAME: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource20180818", "production-db-name" ] }`'
    DB_USER: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource20180818", "production-db-user" ] }`'
    DB_PASSWORD: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource20180818", "production-db-password" ] }`'
    DB_TABLE: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource20180818", "production-db-table" ] }`'
    NEW_KEY: '`{ "Fn::GetAtt": [ "ParameterStoreCustomResource20180818", "production-new-key" ] }`'
```

####Code:
GetParameterStoreKeyValuesByPath.yaml
```yaml
AWSTemplateFormatVersion: 2010-09-09

Parameters:
  LambdaFuncName:
    Type: String
    Description: Lambda Function name
    Default: GetParameterStoreKeyValuesByPath

Resources:
  LambdaFunction:
    Type: "AWS::Lambda::Function"
    Properties:
      Description: Retrieving Parameter Store by path as a Lambda-backed Custom Resource in CloudFormation
      FunctionName:
        Ref: LambdaFuncName
      Handler: "index.lambda_handler"
      Role:
        Fn::GetAtt: [LambdaExecutionRole, Arn]
      Code:
        ZipFile: !Sub |
          # A sample Lambda function.
          from __future__ import print_function
          from botocore.vendored import requests
          import json
          import boto3

          def sendResponse(event, context, responseStatus, responseData):
              responseUrl = event['ResponseURL']
              responseBody = {}
              responseBody['Status'] = responseStatus
              responseBody['Reason'] = 'See the details in CloudWatch Log Stream: ' + context.log_stream_name
              responseBody['PhysicalResourceId'] = context.log_stream_name
              responseBody['StackId'] = event['StackId']
              responseBody['RequestId'] = event['RequestId']
              responseBody['LogicalResourceId'] = event['LogicalResourceId']
              responseBody['Data'] = responseData
              json_responseBody = json.dumps(responseBody)
              headers = {
                  'content-type' : '',
                  'content-length' : str(len(json_responseBody))
              }
              try:
                  response = requests.put(responseUrl,
                                          data=json_responseBody,
                                          headers=headers)
                  print("Response status code: " + response.reason)
              except Exception as e:
                  print("send(..) failed executing requests.put(..): " + str(e))

          def lambda_handler(event, context):
              try:
                  # Dump
                  print("Received event: " + json.dumps(event))
                  eventType = event['RequestType']

                  # Code to Handle Delete Request
                  if eventType == "Delete":
                      # No data to be sent back for delete request
                      responseStatus = "SUCCESS"
                      responseData = None
                      sendResponse(event, context, responseStatus, responseData)

                  # Code to Handle Create Request: eventType == 'Create' or Update
                  else:
                      # Extract a parameter key
                      parameterPath = event['ResourceProperties']['parameterPath']
                      # Get values from the Parameter Store
                      ssm = boto3.client('ssm')
                      response = ssm.get_parameters_by_path(
                          Path=parameterPath,
                          Recursive=True,
                          WithDecryption=True
                      )
                      responseStatus = "SUCCESS"
                      responseData = {}
                      for item in response['Parameters']:
                          ssmKey = item['Name'].split('/')[-1]
                          responseData[ssmKey] = item['Value']
                      sendResponse(event, context, responseStatus, responseData)

              except Exception as e:
                  # If any other exceptions which we didn't expect are raised
                  # then fail the job and log the exception message.
                  print('Function failed due to exception.')
                  print(e)
                  responseStatus = "FAILED"
                  responseData = None
                  sendResponse(event, context, responseStatus, responseData)

              return "Complete."

      Runtime: "python3.6"
      Timeout: "10"

  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
            Action:
              - sts:AssumeRole
      Path: "/"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMReadOnlyAccess
      Policies:
        - PolicyName: CloudWatchLogs
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource:
                  - !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:*'
        - PolicyName: KMS
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - kms:Decrypt
                Resource:
                  - !Sub 'arn:aws:kms:${AWS::Region}:${AWS::AccountId}:key/*'
```

GetParameterStoreKeyValuesByPath.py

```python
# A sample Lambda function.
from __future__ import print_function
from botocore.vendored import requests
import json
import boto3

def sendResponse(event, context, responseStatus, responseData):
    responseUrl = event['ResponseURL']
    responseBody = {}
    responseBody['Status'] = responseStatus
    responseBody['Reason'] = 'See the details in CloudWatch Log Stream: ' + context.log_stream_name
    responseBody['PhysicalResourceId'] = context.log_stream_name
    responseBody['StackId'] = event['StackId']
    responseBody['RequestId'] = event['RequestId']
    responseBody['LogicalResourceId'] = event['LogicalResourceId']
    responseBody['Data'] = responseData
    json_responseBody = json.dumps(responseBody)
    headers = {
        'content-type' : '',
        'content-length' : str(len(json_responseBody))
    }
    try:
        response = requests.put(responseUrl,
                                data=json_responseBody,
                                headers=headers)
        print("Response status code: " + response.reason)
    except Exception as e:
        print("send(..) failed executing requests.put(..): " + str(e))

def lambda_handler(event, context):
    try:
        # Dump
        print("Received event: " + json.dumps(event))
        eventType = event['RequestType']

        # Code to Handle Delete Request
        if eventType == "Delete":
            # No data to be sent back for delete request
            responseStatus = "SUCCESS"
            responseData = None
            sendResponse(event, context, responseStatus, responseData)

        # Code to Handle Create Request: eventType == 'Create' or Update
        else:
            # Extract a parameter key
            parameterPath = event['ResourceProperties']['parameterPath']
            # Get values from the Parameter Store
            ssm = boto3.client('ssm')
            response = ssm.get_parameters_by_path(
                Path=parameterPath,
                Recursive=True,
                WithDecryption=True
            )
            responseStatus = "SUCCESS"
            responseData = {}
            for item in response['Parameters']:
                ssmKey = item['Name'].split('/')[-1]
                responseData[ssmKey] = item['Value']
            sendResponse(event, context, responseStatus, responseData)

    except Exception as e:
        # If any other exceptions which we didn't expect are raised
        # then fail the job and log the exception message.
        print('Function failed due to exception.')
        print(e)
        responseStatus = "FAILED"
        responseData = None
        sendResponse(event, context, responseStatus, responseData)

    return "Complete."
```
