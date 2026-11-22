**AWS Monitoring Troubleshooting: CloudWatch Alarm Did Not Trigger**

When an alarm does not behave as expected, I would verify its configuration and underlying data before changing the threshold.

My checklist:
- Correct metric, namespace, dimensions and Region
- Metric data arriving at the expected frequency
- Threshold, statistic and evaluation periods
- Missing-data treatment
- Alarm state history
- Notification action configuration and SNS subscription status

An alarm in `INSUFFICIENT_DATA` is different from an alarm in `OK` or `ALARM`.

#AWS #CloudWatch #Monitoring #Troubleshooting
