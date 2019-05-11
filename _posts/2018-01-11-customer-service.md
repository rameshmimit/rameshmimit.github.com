---
layout: post
title:  "CloudFormation Custom Resources"
author: Ramesh Kumar
categories: [ CloudFormation, Tutorial ]
tags: [CloudFormation, CustomResource]
image: assets/images/cloudformation_logo.jpg
description: "AWS Cloudformation custom resource exaplanation"
featured: true
hidden: true
---

#### Introduction
Custom resources in CloudFormation can be used to incorporate resources or entities into a CloudFormation stack so that they can be managed as part of the stack. CloudFormation will send a CREATE, UPDATE or DELETE request and the Custom Resource provider will perform arbitrary actions based on the request type. Therefore, you have complete control over the provisioning logic supporting the Custom Resource.

The Custom Resource provider will have to provide a service token that will be used by CloudFormation to send requests during workflows related to the custom resource (CREATE, UPDATE, DELETE).

The workflow looks like as below:

Whenever a resource type of AWS::CloudFormation::CustomResource or Custom::<resource-name> is specified in a CloudFormation template, the ServiceToken property must be populated as well. As of today, the only two values that can be used there is the ARN of an SNS Topic or the ARN of a Lambda function. Additionally, input data can be specified which might be required by the Custom Resource provider.

```html
"MyCustomResource" : {
  "Type" : "Custom::TestCustomResource",
  "Properties" : {
    "ServiceToken": "arn:aws:lambda:us-east-1:123456789012:function:CustomResourceFunction",           ----------> REQUIRED
    "testdata1": "1",      ----->|
    "testdata2": "test"          v------------> OPTIONAL
  }
}
```

When a template containing Custom Resource definitions similar to the snippet above is used to create, update, or delete a Custom Resource, CloudFormation will send a request to the service token specified in the template. The service token must be in the same region in which you are creating the stack. The request will be of a JSON format and will always contain the following information.

```html
{
   "RequestType" : "Create",     --------> Will change based on workflow. Can be Create, Update, or Delete.
   "ResponseURL" : "http://pre-signed-S3-url-for-response",       ---------> Will change for each request/response.
   "StackId" : "arn:aws:cloudformation:us-west-2:EXAMPLE/stack-name/guid",       ---------> Will never change during the lifetime of a custom resource.
   "RequestId" : "unique id for thi3. s create request",       ---------> Will change for each request/response.
   "ResourceType" : "Custom::MyCustomResource",       -------> Can be "Custom::<resource-name>" or "AWS::CloudFormation::CustomResource".
   "LogicalResourceId" : "MyCustomResource",       --------> Will never change during the lifetime of a custom resource.
   "ResourceProperties" : {
      "Name" : "Value",                    ----------> Custom values specified in the
      "List" : [ "1", "2", "3" ]           ----------> template that can be changed/updated.
   }
}
```

If a Lambda function is being used as the Custom Resource provider, CloudFormation essentially invokes the Lambda function whose ARN is specified in the template. When invoking the function, it will use the Request Object mentioned above as the input to the Lambda function.

The Custom Resource provider will consume the Request Object and perform arbitrary actions. After these arbitrary actions are performed, based on the SUCCESS or FAILURE of those actions, the custom resource provider will have to send a response back to CloudFormation with either a SUCCESS or FAILURE message. The response will be a JSON object consisting of the following fields.

```html
{
   "Status" : "SUCCESS",     --------> Required - can be "SUCCESS" or "FAILURE".
   "PhysicalResourceId" : "TestResource1",      --------> Required - can be any non-empty string up to 1 Kb in size.
   "StackId" : "arn:aws:cloudformation:us-west-2:EXAMPLE:stack/stack-name/guid",       ---------> Required - copy verbatim from the request object.
   "RequestId" : "unique id for this create request",        ---------> Required - copy verbatim from the request object.
   "LogicalResourceId" : "MyTestResource",       ---------> Required - copy verbatim from the request object.
   "Reason": "Check CloudWatch logs for info."       ---------> Required if Status is FAILURE. Arbitrary string.
   "Data" : {
      "OutputName1" : "Value1",         ---------> Optional name-value pairs that can be returned to
      "OutputName2" : "Value2",         ---------> CloudFormation and can be accessed using Fn::GetAtt.
   }
}
```
Once a response object of this format has been constructed, the Custom Resource provider will then have to upload this to the pre-signed S3 URL that was included in the request object. CloudFormation will be monitoring the pre-signed S3 URL for the response object. Once it detects a response object uploaded there, it will parse through the object to determine the next actions to be taken. If the value returned is SUCCESS, CloudFormation will mark the workflow a success and move on to the next step in the stack workflow. If the value returned is FAILURE, CloudFormation will mark the workflow a failure and display the message that was returned in the "Reason" field of the response object. If no response is detected by CloudFormation, or CloudFormation cannot successfully parse the response object within the timeout period of 60 minutes, the workflow will fail and the stack will start to rollback.
