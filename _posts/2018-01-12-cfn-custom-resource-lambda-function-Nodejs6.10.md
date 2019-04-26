---
layout: post
title:  "CloudFormation Custom Resource Lambda Function - Nodejs6.10 "
author: Ramesh Kumar
categories: [ CloudFormation, CustomResources, AWS]
tags: [CloudFormation, CustomResource]
image: assets/images/cloudformation_logo.jpg
description: "AWS Cloudformation custom resource exaplanation"
---
The following is a function "skeleton" that can be copied as is and only the customer's workflow logic needs to be added to it. It already has some other best practices/hacks that will ensure that a customer does not have to wait for the resource to timeout in CloudFormation. As long as they can test that the code they added is valid and the function does not throw any errors, this function should handle everything else. It implements a try-catch model so that a failure response is sent in case something unexpected happens. It prints the original request sent by CloudFormation so that as a worst-case scenario, the customer will be able to see the pre-signed S3 URL and they can manually do a POST to it with a SUCCESS/FAILURE message. Additionally, a retry logic has been implemented and the number of retries can either be hard-codpage2/ed in the function, or passed as a parameter from the CloudFormation stack. No extra modules need to be included with the function zip that is stored in S3. This works out of the box.


```nodejs
var aws = require("aws-sdk");

exports.handler = function(event, context) {

    console.log("REQUEST RECEIVED:\n" + JSON.stringify(event));

    var count = 1;
//    count = parseInt(event.ResourceProperties.count)     // Uncomment this line if you are configuring the number of retries through the CFN template

    var attempts = 0;

    if (count <= 3) {
        var count = 3;
    }

    while (attempts < count){
        try {
            if (event.RequestType == "Delete") {
                console.log("delete\n");
                // The rest of your delete logic goes herepage2/
            } else if (event.RequestType == "Create") {
                console.log("create\n");
                // The rest of your create logic goes here
            } else if (event.RequestType == "Update") {
                console.log("update\n");
                // The rest of your update logic goes here
            }

            var responseStatus = "SUCCESS";
            var responseData = {"Success": "Everything worked."}

            break;
        }
        catch(err) {
            var responseStatus = "FAILURE";page2/
            var responseData = {"Failure": "Something bad happened."};

            var attempts = attempts + 1;
        }
    }
  sendResponse(event, context, responseStatus, responseData);
};

// Send response to the pre-signed S3 URL
function sendResponse(event, context, responseStatus, responseData) {

    var responseBody = JSON.stringify({
        Status: responseStatus,
        Reason: "See the details in CloudWatch Log Stream: " + context.logStreamName,
        PhysicalResourceId: context.logStreamName,
        StackId: event.StackId,
        RequestId: event.RequestId,
        LogicalResourceId: event.LogicalResourceId,
        Data: responseData
    });

    console.log("RESPONSE BODY:\n", responseBody);

    var https = require("https");
    var url = require("url");

    var parsedUrl = url.parse(event.ResponseURL);
    var options = {
        hostname: parsedUrl.hostname,
        port: 443,
        path: parsedUrl.path,
        method: "PUT",
        headers: {
            "content-type": "",
            "content-length": responseBody.length
        }
    };

    console.log("SENDING RESPONSE...\n");

    var request = https.request(options, function(response) {
        console.log("STATUS: " + response.statusCode);
        console.log("HEADERS: " + JSON.stringify(response.headers));
        // Tell AWS Lambda that the function execution is done
        context.done();
    });

    request.on("error", function(error) {
        console.log("sendResponse Error:" + error);
        // Tell AWS Lambda that the function execution is done
        context.done();
    });

    // write data to request body
    request.write(responseBody);
    request.end();
}
```
