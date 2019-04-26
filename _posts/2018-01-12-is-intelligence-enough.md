---
layout: post
title:  "CloudFormation Custom Resource Lambda Function - Python 2.7 "
author: Ramesh Kumar
categories: [ CloudFormation, CustomResources, AWS]
tags: [CloudFormation, CustomResource]
image: assets/images/cloudformation_logo.jpg
description: "AWS Cloudformation custom resource exaplanation"
featured: false
hidden: true
---
The following is a function "skeleton" that can be copied as is and only the customer's workflow logic needs to be added to it. It already has some other best practices/hacks that will ensure that a customer does not have to wait for the resource to timeout in CloudFormation. As long as they can test that the code they added is valid and the function does not throw any errors, this function should handle everything else. It implements a try-catch model so that a failure response is sent in case something unexpected happens. It prints the original request sent by CloudFormation so that as a worst-case scenario, the customer will be able to see the pre-signed S3 URL and they can manually do a POST to it with a SUCCESS/FAILURE message. Additionally, a retry logic has been implemented and the number of retries can either be hard-coded in the function, or passed as a parameter from the CloudFormation stack. No extra modules need to be included with the function zip that is stored in S3. This works out of the box.

```python
import json
import time
from botocore.vendored import requests
def lambda_handler(event, context):
    print 'REQUEST BODY:n' + str(event)
    count = 1
    #count = int(event['ResourceProperties']['count'])    #Uncomment this line if you are configuring the number of retries through the CFN template
    attempts = 0
    if count <= 3:
        count = 3
    while attempts < count:
        try:
            if event['RequestType'] == 'Delete':
                print "delete"
                #The rest of your delete logic goes here
            elif event['RequestType'] == 'Create':
                print "create"
                #The rest of your create logic goes here
            elif event['RequestType'] == 'Update':
                print "update"
                #The rest of your update logic goes here
            responseStatus = 'SUCCESS'
            responseData = {'Success': 'Everything worked.'}
            break
        except:
            responseStatus = 'FAILURE'
            responseData = {'Failure': 'Something bad happened.'}
            attempts += 1
            time.sleep(3)
    sendResponse(event, context, responseStatus, responseData)

def sendResponse(event, context, responseStatus, responseData, reason=None, physical_resource_id=None):
    responseBody = {'Status': responseStatus,
                    'Reason': 'See the details in CloudWatch Log Stream: ' + context.log_stream_name,
                    'PhysicalResourceId': physical_resource_id or context.log_stream_name,
                    'StackId': event['StackId'],
                    'RequestId': event['RequestId'],
                    'LogicalResourceId': event['LogicalResourceId'],
                    'Data': responseData}
    print 'RESPONSE BODY:n' + json.dumps(responseBody)
    responseUrl = event['ResponseURL']
    json_responseBody = json.dumps(responseBody)
    headers = {
        'content-type' : '',
        'content-length' : str(len(json_responseBody))
    }
    try:
        response = requests.put(responseUrl,
                                data=json_responseBody,
                                headers=headers)
        print "Status code: " + response.reason
    except Exception as e:
        print "send(..) failed executing requests.put(..): " + str(e)
```
