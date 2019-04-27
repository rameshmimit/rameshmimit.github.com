To create a new WAF through ebextensions we will need to create a minimum of 3 custom resources.

1. WebACL
2. WebACL rule
3. Condition to match in rule

Note: One ACL can have multiple rules and conditions. I have just used one rule and one condition in my example.

Below are the resources that I have created using ebextension in order to associate a WAF to the ElasticBeanstalk environment

1. AWS::WAFRegional::WebACL -> This will create a WebACL which will be associated to the ALB. It refers to the rule for the WebACL created by the resource 'AWS::WAFRegional::Rule'

2. AWS::WAFRegional::Rule -> This resource will create the rule which has been referred in the WebACL. You can create and refer all the conditions for your desired rules in this resource. In my sample template I have referred it to 'AWS::WAFRegional::ByteMatchSet'.

3. AWS::WAFRegional::ByteMatchSet -> This resource will create conditions for the rules which has been referred in the Rule above. In my sample template I have Created a rule to block all traffic which are coming from Firefox browsers.

4. AWS::WAFRegional::WebACLAssociation -> This resource will associate WebACL to the Application Load Balancer of the beanstalk environment.

```yaml
Resources:
  MyWebACL:
    Type: "AWS::WAFRegional::WebACL"
    Properties:
      Name: "MyWebACLForWAF"
      DefaultAction:
        Type: "ALLOW"
      MetricName: "MyWebACL"
      Rules:
        -
          Action:
            Type: "BLOCK"
          Priority: 1
          RuleId:
            Ref: "BadReferersRule"
  MyWebACLAssociation:
    Type: "AWS::WAFRegional::WebACLAssociation"
    Properties:
      ResourceArn:
        Ref: AWSEBV2LoadBalancer
      WebACLId:
        Ref: MyWebACL
  BadReferersRule:
    Type: "AWS::WAFRegional::Rule"
    Properties:
      Name: "BadReferersRule"
      MetricName: "BadReferersRule"
      Predicates:
        -
          DataId:
            Ref: "BadReferers"
          Negated: false
          Type: "ByteMatch"
  BadReferers:
    Type: "AWS::WAFRegional::ByteMatchSet"
    Properties:
      Name: "ByteMatch for matching bad HTTP referers"
      ByteMatchTuples:
        -
          FieldToMatch:
            Type: "HEADER"
            Data: "User-Agent"
          TargetString: "Firefox"
          TextTransformation: "NONE"
          PositionalConstraint: "CONTAINS"
```

Also, if you just want to attach an already created WAF to a new ElasticBeanstalk environment, you can simply use the resource 'AWS::WAFRegional::WebACLAssociation' in ebextensions and give the WebACLId of the present WAF.

```yaml
Resources:
  MyWebACLAssociation:
    Type: "AWS::WAFRegional::WebACLAssociation"
    Properties:
      ResourceArn:
        Ref: AWSEBV2LoadBalancer
      WebACLId: <Id of the Web ACL>
```
