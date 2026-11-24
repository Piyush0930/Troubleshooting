AWS Monitoring Troubleshooting: CloudWatch Alarm Did Not Trigger
A CloudWatch alarm not triggering does not always mean the threshold is configured incorrectly.

The actual problem could be that CloudWatch is not receiving metric data, the alarm is evaluating the wrong metric, the evaluation configuration is incorrect, or the alarm notification is not being delivered.

The correct troubleshooting approach is to investigate the complete monitoring flow, from metric collection to notification delivery.
1. Understand how a CloudWatch alarm works
Before troubleshooting, understand the flow.
AWS Resource

EC2, RDS, Lambda, Application Load Balancer, etc.

Amazon CloudWatch Metrics

Collects and stores monitoring data

CloudWatch Alarm

Evaluates metrics against configured conditions

Alarm Action

Amazon SNS

Email, SMS or other subscribed endpoints


If any part of this flow is misconfigured, the expected notification might not arrive.
2. Verify the metric configuration
This is the first thing I would check.

CloudWatch alarms monitor a specific metric. If the alarm points to the wrong metric, it may never trigger.

Verify these properties:
Configuration	What to verify
Namespace	Correct service namespace, such as AWS/EC2
Metric name	Correct metric, such as CPUUtilization
Dimensions	Correct Instance ID, Load Balancer or other resource dimensions
Region	Alarm and metric are in the correct AWS Region
Account	Correct AWS account, especially in cross-account monitoring

Practical example:

Suppose an EC2 instance has an ID:

i-0123456789abcdef0

You create an alarm for CPU utilization above 80%.

But the alarm is configured with the wrong Instance ID.

Even if the actual EC2 instance reaches 95% CPU utilization, the alarm will not evaluate that instance's metric.

Troubleshooting: Open CloudWatch → Metrics → EC2 → Per-Instance Metrics and verify that the correct instance metric is available.








docs.aws.amazon.com


3. Verify whether metric data is arriving
An alarm cannot evaluate data that CloudWatch is not receiving.

For example:

You configured:
- Metric: CPUUtilization
- Threshold: Greater than 80%
- Period: 5 minutes
- Evaluation periods: 2
But the metric is not being published correctly.

The alarm might remain in INSUFFICIENT_DATA instead of entering ALARM.

Check:
1. Is the EC2 instance running?
2. Is the selected metric available?
3. When was the last datapoint received?
4. Is the metric publishing at the expected frequency?
5. Is the metric custom or provided by AWS?
Important distinction:
- EC2 basic monitoring typically provides 5-minute metric intervals.
- EC2 detailed monitoring provides 1-minute metric intervals.
- The CloudWatch Agent can publish additional OS-level metrics such as memory and disk utilization.
For example, EC2 does not provide memory utilization as a standard built-in metric. You generally need the CloudWatch Agent or another metric-publishing mechanism.








techtarget.com


4. Verify threshold, statistic and evaluation periods
A common mistake is configuring the correct metric but using the wrong evaluation settings.

Consider this example:
Example CloudWatch alarm configuration
Metric

CPUUtilization

Threshold

Above 80%

Period

5 minutes

Statistic

Average

Evaluation periods

2

Datapoints to alarm

2 out of 2



What does this mean?

CloudWatch evaluates CPU utilization over 5-minute periods. Both evaluation datapoints must breach the 80% threshold before the alarm enters ALARM.

For example:
Period	Average CPU	Result
10:00–10:05	85%	Breaching
10:05–10:10	90%	Breaching
10:10–10:15	60%	Not breaching

The alarm enters ALARM after the first two breaching datapoints.

If the configuration requires 2 out of 2 datapoints, a single CPU spike is not enough.
Understand the statistics
Statistic	Meaning
Average	Average metric value
Maximum	Highest metric value
Minimum	Lowest metric value
Sum	Total of metric values
SampleCount	Number of samples received

For CPU utilization monitoring, Average is commonly used. For detecting brief spikes, Maximum may be more appropriate.
5. Check missing-data treatment
This is particularly important in CloudWatch alarm troubleshooting.

CloudWatch supports four missing-data treatments:
Setting	Behavior
missing	Missing datapoints are treated as missing
notBreaching	Missing datapoints are treated as healthy
breaching	Missing datapoints are treated as breaching
ignore	Retains the current alarm state

Practical scenario:

Imagine you monitor an application using a custom CloudWatch metric.

The application normally publishes a metric every minute. Suddenly, the application stops publishing data.

Your alarm's missing-data configuration determines how CloudWatch handles this situation.

For example:
- notBreaching: Missing data does not cause an alarm.
- breaching: Missing data can cause an alarm.
- ignore: The alarm retains its previous state.
- missing: CloudWatch treats the datapoints as missing and determines whether it has enough data to evaluate the alarm.
Important: Missing data is not automatically the same as zero.

If your monitoring requirement is to detect when an application stops publishing heartbeats, treating missing data as breaching can be useful.
6. Understand CloudWatch alarm states
An alarm can have three main states.
OK
Condition is not breached

The metric is within the configured threshold.
ALARM
Condition is breached

The configured alarm condition has been met.
INSUFFICIENT_DATA
Not enough information

CloudWatch does not have enough usable datapoints to determine the alarm state.
A common troubleshooting mistake is assuming INSUFFICIENT_DATA means the threshold is wrong.

It can instead indicate:
- Missing metric data.
- Incorrect dimensions.
- A newly created alarm that has not received enough data.
- A metric publishing problem.
7. Verify alarm history
CloudWatch alarm history helps identify what happened and when.

Navigate to:

CloudWatch → Alarms → Select your alarm → History

Check:
- When did the alarm state change?
- Did it transition from OK to ALARM?
- Did it transition to INSUFFICIENT_DATA?
- Was an alarm action executed?
- Was the alarm configuration modified?
This helps distinguish between a metric evaluation issue and a notification delivery issue.








stackoverflow.com


8. Verify SNS notification configuration
Sometimes the alarm triggers correctly, but the notification never reaches the intended recipient.

For example:
- CloudWatch alarm enters ALARM.
- SNS notification action is configured.
- Email notification is not received.
Check:
1. Is the correct SNS topic configured in the alarm action?
2. Is the SNS topic in the correct Region?
3. Is the email address subscribed?
4. Has the subscription been confirmed?
5. Does the SNS topic policy allow the CloudWatch alarm to publish?
6. Is the notification action enabled?
Important distinction:

If the alarm is in ALARM but the email does not arrive, investigate SNS and notification delivery. Do not immediately change the metric threshold.








blog.cloudmentor.pro


9. Practical troubleshooting scenario
Imagine a production EC2 instance is monitored using CloudWatch.
Scenario: CPU utilization reaches 95%, but no email alert arrives.
Expected alert not received
Troubleshooting sequence
1. Check the EC2 metric
   
   Confirm that CPUUtilization is publishing data for the correct Instance ID.
2. Check the metric graph
   
   Verify that the actual CPU utilization reached 95%.
3. Check alarm configuration
   
   Verify threshold, statistic, period and evaluation periods.
4. Check alarm state
   
   Determine whether it is OK, ALARM or INSUFFICIENT_DATA.
5. Check alarm history
   
   Identify whether CloudWatch detected a state transition.
6. Check SNS
   
   Verify the topic, subscription confirmation and delivery configuration.

This approach helps isolate the actual failure instead of randomly modifying alarm settings.
10. Useful AWS CLI commands
When you have the required IAM permissions, you can troubleshoot from the AWS CLI.

Check alarm configuration and state
aws cloudwatch describe-alarms \
  --alarm-names "High-CPU-Alarm" \
  --region ap-south-1


Check recent metric datapoints
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --start-time 2026-11-24T10:00:00Z \
  --end-time 2026-11-24T12:00:00Z \
  --period 300 \
  --statistics Average \
  --region ap-south-1


Check alarm history
aws cloudwatch describe-alarm-history \
  --alarm-name "High-CPU-Alarm" \
  --history-item-type StateUpdate \
  --max-records 10 \
  --region ap-south-1


Check SNS subscriptions
aws sns list-subscriptions-by-topic \
  --topic-arn "arn:aws:sns:ap-south-1:123456789012:cloudwatch-alerts" \
  --region ap-south-1


Replace the example instance ID, alarm name, account ID and topic ARN with your actual values.
