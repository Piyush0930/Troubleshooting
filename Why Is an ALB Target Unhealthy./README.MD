AWS Deep Dive: Troubleshooting Why an ALB Target Becomes Unhealthy

1. Understand the architecture
An Application Load Balancer distributes incoming traffic across registered targets such as EC2 instances. It periodically sends health-check requests to each target. EC2 Running does not mean the application is healthy.

2. How ALB health checks work
The ALB sends HTTP or HTTPS requests to a configured target port and path. Important settings include protocol, port, path, interval, timeout, healthy threshold, unhealthy threshold and success-code matcher.

3. Ten common causes of unhealthy targets
- Incorrect health-check path: The endpoint returns 404 instead of the expected 200. Test using curl -i http://localhost:8080/health.
- Incorrect port: The application listens on 3000 but the Target Group checks 8080. Verify using sudo ss -tulnp.
- Security Group blocking traffic: Allow the application port inbound on the EC2 Security Group from the ALB Security Group. Check ALB outbound rules too.
- Nginx or Apache stopped: Check sudo systemctl status nginx or sudo systemctl status httpd.
- HTTP 500: Investigate application exceptions, database connections, credentials and dependencies.
- Health-check timeout: Check response time, CPU, memory and dependencies using time curl, top and free -h.
- Application listens only on localhost: Check whether it binds to 127.0.0.1 rather than an appropriate network interface.
- Nginx reverse-proxy failure: Check backend availability and /var/log/nginx/error.log.
- Availability Zone not enabled: Verify ALB Availability Zones and target registration.
- Deployment/startup issues: Check application initialization, readiness, deregistration delay and deployment strategy.

4. Target health states
Healthy: Passed checks. Unhealthy: Failed checks. Initial: Initial health checks. Draining: Being deregistered. Unused: Not eligible for use. Unavailable: Health cannot currently be determined.

5. Important reason codes
Target.ResponseCodeMismatch: Unexpected HTTP response. Target.Timeout: No response within timeout. Target.FailedHealthChecks: Consecutive failures. Target.NotInUse: Target Group or Availability Zone configuration issue. Target.NotRegistered: Target not registered. Elb.InternalError: Internal load balancer error.

6. All targets unhealthy
ALB can fail open and route requests to registered unhealthy targets when all targets are unhealthy. This does not mean the targets have recovered.

7. AWS CLI commands
Check target health:
aws elbv2 describe-target-health --target-group-arn YOUR_TARGET_GROUP_ARN --region ap-south-1

Check Target Group configuration:
aws elbv2 describe-target-groups --target-group-arns YOUR_TARGET_GROUP_ARN --region ap-south-1

Check EC2 state:
aws ec2 describe-instances --instance-ids i-xxxxxxxx --region ap-south-1

Check Security Groups:
aws ec2 describe-security-groups --group-ids sg-xxxxxxxx --region ap-south-1

Check EC2 system status:
aws ec2 describe-instance-status --instance-ids i-xxxxxxxx --include-all-instances --region ap-south-1

8. CloudWatch monitoring
HealthyHostCount: Number of healthy targets. UnHealthyHostCount: Number of unhealthy targets. TargetResponseTime: Target response time. HTTPCode_Target_5XX_Count: Target-generated server errors. HTTPCode_ELB_5XX_Count: Load-balancer-generated server errors.

Create alarms based on your application's capacity and baseline. Example conditions: UnHealthyHostCount greater than zero, HealthyHostCount below minimum required capacity, elevated TargetResponseTime and increasing HTTP 5XX errors.

9. Logs to investigate
ALB access logs: Client requests and target responses. Nginx access/error logs: Web-server and proxy issues. Application logs: Code exceptions and backend failures. CloudWatch: Metrics and trends.

10. Production troubleshooting workflow
1. Verify EC2 state and system status.
2. Check Target Group health.
3. Identify the exact reason code.
4. Verify health-check protocol, port and path.
5. Check the application service and listening port.
6. Test using curl.
7. Check Security Groups, NACLs and routing.
8. Review application and web-server logs.
9. Fix the root cause.
10. Verify the target becomes healthy and monitor recovery.

11. Health endpoint best practices
Use a dedicated readiness endpoint such as GET /health that returns HTTP 200 when the application is ready to receive traffic. Avoid blindly accepting error codes or depending on every external service in every health check.

12. Production example
If EC2-02 returns HTTP 502 on /health, inspect Nginx error logs. If they show connection refused to the Node.js backend on port 3000, check the application service using sudo systemctl status myapp and logs using sudo journalctl -u myapp -n 100. Fix the actual application issue, test both backend and Nginx endpoints, and confirm Target Group recovery.

Key lesson: EC2 instance health and application health are different signals. Identify the reason code, reproduce the health check, investigate logs and prove the root cause before making changes.

Official AWS documentation:
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html

#AWS #AmazonWebServices #ALB #ApplicationLoadBalancer #EC2 #TargetGroups #CloudWatch #Monitoring #Troubleshooting #AWSNetworking #DevOps #CloudEngineering #CloudSupport #SRE
