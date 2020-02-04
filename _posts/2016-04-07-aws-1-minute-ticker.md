---
layout: post
title: A CloudFormation recipe for scheduling Lambda functions with 1-minute frequency
categories: [infrastructures]
tags: [aws, cloudformation, lambda, cron, cloud]
---

Among the plethora of tools which Amazon has given us there's _AWS CloudFormation_. IMO, this proves that AWS is (still) in a class of it's own. It's the embodiment of the "infrastructure as code" concept. And, it's battle-tested, it works! So well, in fact, that they _ElasticBeanstalk_ right on-top of it.

## Disclaimer:

I have not used this in production yet, I'm currently running a longer test to gain some confidence. Therefore I do not recommend you rely on this for critical stuff. Test first!

## Automation not supported

However, once you gain some experience with this beast, you start realizing that it also has faults. In fact, they tend to pile up; that's one reason we now have some promissing tools like [Terraform](http://terraform.io) to play with. One such issue is the fact that **CloudFormation doesn't support the creation of scheduled (recurring) Lambda functions**. And that's a _big_ problem, because you're forced to do it _by hand_, by creating a [scheduled event source](http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/RunLambdaSchedule.html) using the AWS CLI, APIs or the Web Console.

## Striving for 1-minute rate

Moreover, even if you _do_ schedule a Lambda function by hand, the fastest rate at which you can invoke it is [**5 minutes**, no less](http://docs.aws.amazon.com/lambda/latest/dg/tutorial-scheduled-events-schedule-expressions.html).

Recently, I needed an **automatically created** Lambda function to be periodically invoked at **1-minute intervals**. No more, no less. And, eventually, I found a way.

<!--more-->

## Strategy

The trick is to use _a custom metric_ called `Tick` (published by our lambda) which can be either 0(zero) or 1(one). We set two CloudWatch alarms which trigger on each state, respectively: _TickState0_ triggers when `Tick == 0` for _1+ minutes_ and _TickState1_ triggers when `Tick == 1` for _1+ minutes_.

```json
{
  "TickState0": {
    "Type": "AWS::CloudWatch::Alarm",
    "Properties": {
      "AlarmName": "TickState0",
      "Namespace": "Tick",
      "MetricName": "Tick",
      "Statistic": "Average",
      "EvaluationPeriods": 1,
      "Period": 60,
      "Threshold": 0,
      "ComparisonOperator": "LessThanOrEqualToThreshold",
      "AlarmActions": [{ "Ref": "TickTopic" }]
    }
  },

  "TickState1": {
    "Type": "AWS::CloudWatch::Alarm",
    "Properties": {
      "AlarmName": "TickState1",
      "Namespace": "Tick",
      "MetricName": "Tick",
      "Statistic": "Average",
      "EvaluationPeriods": 1,
      "Period": 60,
      "Threshold": 1,
      "ComparisonOperator": "GreaterThanOrEqualToThreshold",
      "AlarmActions": [{ "Ref": "TickTopic" }]
    }
  }
}
```

Each Alarm triggers the Lambda (via an SNS topic) and the Lambda toggles the `Tick` metric (which will trigger the opposite alarm after ~1 minute).

```python
# Get message body(stack update or alarm).
message = json.loads(event['Records'][0]['Sns']['message'])

# Detect source event.
source = message.get('RequestType', 'Alarm')

# Confirm stack deletion.
if source == 'Delete':
  return send(message, context, SUCCESS)

# Set / toggle state.
if source == 'Alarm':
  state = int(not float(message['Trigger']['threshold']))
else :
  state = 0

# Set metric to state.
cw = boto3.client('cloudwatch')
cw.put_metric_data(
  Namespace = 'Tick',
  MetricData = [{
    'MetricName': 'Tick',
    'Value': state
  }]
)
```

What's left is to use a [CF custom resource](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html) to ensure the Lambda is called on stack operations (create/update/delete) so the whole setup is self-managed.

```json
{
  "StartTick": {
    "Type": "Custom::StartTick",
    "DependsOn": ["TickState0", "TickState1"],
    "Properties": {
      "ServiceToken": { "Ref": "TickTopic" }
    }
  }
}
```

## Test Drive!

- Clone the code (see below) and use it to launch a CF stack. You'll have to upload the lambda function (the `tick.zip` file) on S3 somewhere (and update it's URI in the CF JSON file).
- After it started, you should observe in the CloudWatch -> Logs section that a log message is published every minute by the Lambda function (see screenshot below).

![Invocation traces](/assets/lambda-ticker-logs.png)

## Some considerations

- Please note that the actual Lambda has additional code and is prepared to be invoked by either the _CloudWatch Alarms_ or the _custom resource events_ (i.e. stack updates).
- Expecting a shorter trigger period using this method is not realistic, since AWS CloudWatch will always aggregate metrics inside a 1-minute time period.
- In practice, however, you will observe that this actually tends to be triggered _faster_ than on 1-minute intervals; this is influenced by the different check-times of each CloudWatch alarm (check times differ).

[See complete code on GitHub](https://github.com/vpalos/aws-1minute-timer)

## That's it, enjoy! :)
