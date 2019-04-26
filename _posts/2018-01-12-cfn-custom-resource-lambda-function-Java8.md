---
layout: post
title:  "CloudFormation Custom Resource Lambda Function - Java8 "
author: Ramesh Kumar
categories: [ CloudFormation, CustomResources, AWS]
tags: [CloudFormation, CustomResource]
image: assets/images/cloudformation_logo.jpg
description: "AWS Cloudformation custom resource exaplanation"
---
The following is a function "skeleton" that can be copied as is and only the customer's workflow logic needs to be added to it. It already has some other best practices/hacks that will ensure that a customer does not have to wait for the resource to timeout in CloudFormation. As long as they can test that the code they added is valid and the function does not throw any errors, this function should handle everything else. It implements a try-catch model so that a failure response is sent in case something unexpected happens. It prints the original request sent by CloudFormation so that as a worst-case scenario, the customer will be able to see the pre-signed S3 URL and they can manually do a POST to it with a SUCCESS/FAILURE message.

```java
package com.cfn.customresource;

import java.io.IOException;
import java.io.OutputStreamWriter;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.Map;
import org.json.JSONObject;
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.LambdaLogger;
import com.amazonaws.services.lambda.runtime.RequestHandler;

public class CustomResourceHandler implements RequestHandler<Map<String, Object>, Object> {

	@Override
	public Object handleRequest(Map<String, Object> input, Context context) {
		String responseStatus = "";
		LambdaLogger logger = context.getLogger();
		logger.log("Input: " + input);

    int count = 3;

    int attempts = 0;

		String requestType = (String) input.get("RequestType");
    JSONObject responseData = new JSONObject();

    while (attempts < count) {
		try {
			if (requestType.equalsIgnoreCase("Create")) {
				logger.log("CREATE!");
				// Put your custom create logic here
			} else if (requestType.equalsIgnoreCase("Update")) {
				logger.log("UPDATE!");
				// Put your custom update logic here
			} else if (requestType.equalsIgnoreCase("Delete")) {
				logger.log("DELETE!");
				// Put your custom delete logic here
			}
			responseData.put("Message", "Everything worked.");
			responseStatus = "SUCCESS";
			break;
		} catch (Exception e) {
			responseStatus = "FAILURE";

			logger.log("FAILURE!");
			responseData.put("Message", "Something bad happened.");
			attempts = attempts + 1;
		}
	}
    return sendResponse(input, context, responseStatus, responseData);
	}

	private Object sendResponse(final Map<String, Object> input, final Context context, final String responseStatus,
			JSONObject responseData) {

		String responseURL = (String) input.get("ResponseURL");

		context.getLogger().log("ResponseURL: " + responseURL);

		URL url;
		try {
			url = new URL(responseURL);

			HttpURLConnection connection = (HttpURLConnection) url.openConnection();
			connection.setDoOutput(true);
			connection.setRequestMethod("PUT");
			OutputStreamWriter response = new OutputStreamWriter(connection.getOutputStream());
			JSONObject responseBody = new JSONObject();

			responseBody.put("Status", responseStatus);
			responseBody.put("PhysicalResourceId", context.getLogStreamName());
			responseBody.put("StackId", input.get("StackId"));
			responseBody.put("RequestId", input.get("RequestId"));
			responseBody.put("LogicalResourceId", input.get("LogicalResourceId"));
			responseBody.put("Data", responseData);

			response.write(responseBody.toString());
			System.out.println("");
			response.close();
			context.getLogger().log("Response Code: " + connection.getResponseCode());

		} catch (IOException e) {
			e.printStackTrace();
		}

		return null;
	}

}
```
Customers can use the zip file attached to create a JAR file. They will have to unzip the contents and locate the file called CustomResourceHandler.java. They can modify this file and add their custom create,update, and delete logic and save it. Once saved, they will have to use maven to rebuild the package which should create a new JAR file called customresource-1.0.0.jar in the "target" directory. Use this JAR file to create the lambda function. Specify the handler as "com.cfn.customresource.CustomResourceHandler::handleRequest".

Download: [CustomResouces.zip](http://CustomResource.zip)
