---
layout: post
title:  "Best practices for implementing Lambda-backed custom CloudFormation resources"
author: Ramesh
categories: [ CloudFormation, CustomResources, DevOps ]
image: assets/images/CloudFormation-Custom-Resource.png
featured: true
hidden: true
---

When implementing Lambda-backed custom resources in your CloudFormation stack, consider the following best practices:

#### Build your custom resources to report, log, and handle failure gracefully
Exceptions can cause your function code to exit without sending a response. Because CloudFormation requires an HTTPS response to confirm whether the operation was a success or a failure, an unreported exception will cause CloudFormation to wait until the operation times out before starting a stack rollback. If the exception occurs again on rollback, CloudFormation will wait again for a timeout before ultimately ending in a rollback failure. During this time, your stack is unusable, and timeout issues can be time-consuming to troubleshoot.

To avoid this, make sure that your function's code has logic to handle exceptions, the ability to log the failure to help you troubleshoot, and if needed, the ability to  respond back to CloudFormation with an HTTPS response confirming that an operation failed.

#### Set reasonable timeout periods, and report when they're about to be exceeded
If an operation doesn't execute within its defined timeout period, the function raises an exception and no response is sent to CloudFormation.

To avoid this, ensure that the timeout value for your Lambda functions is set high enough to handle variations in processing time and network conditions. Consider also setting a timer in your function to respond to CloudFormation with an error when a function is about to time out; this can help prevent function timeouts from causing custom resource timeouts and delays.

#### Understand and build around [Create](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref-requesttypes-create.html), [Update](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref-requesttypes-update.html), and [Delete](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref-requesttypes-delete.html) events
Depending on the stack action, CloudFormation sends your function a Create, Update, or Delete event. Each event is handled distinctively, so you should ensure that there are no unintended behaviors when any of the three event types is received.

For more information, see [Custom Resource Request Types](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref-requesttypes.html).

#### Make sure that your functions are designed with idempotency in mind
An idempotent function can be repeated any number of times with the same inputs, and the result will be the same as if it had been done only once. Idempotency is valuable when working with CloudFormation to ensure that retries, updates, and rollbacks don't cause the creation of duplicate resources, errors on rollback or delete, or other unintended effects.

For example, if CloudFormation invokes your function to create a resource, but doesn't receive a response that the resource was created successfully, CloudFormation might invoke the function again, resulting in the creation of a second resource; the first resource may become orphaned.

How to address this can differ depending on the action your function is intended to perform, but a common technique is to use a uniqueness token that CloudFormation can use to check for preexisting resources. For example, a hash of the StackId and LogicalResourceId could be stored in the resource's metadata or in a DynamoDB table.


#### Rollbacks
If a stack operation fails, CloudFormation attempts to roll back, reverting all resources to their prior state. This results in different behaviors depending on whether the update caused a resource replacement.

Ensuring that replacements are properly handled and the old resources are not implicitly removed until a delete event is received will help ensure that rollbacks are executed smoothly.



To help implement best practices when using custom resources, consider using the [Custom Resource Helper](https://github.com/awslabs/aws-cloudformation-templates/tree/master/community/custom_resources/python_custom_resource_helper) provided by [awslabs](https://github.com/awslabs), which can assist with exception and timeout trapping, sending responses to CloudFormation, and logging.
