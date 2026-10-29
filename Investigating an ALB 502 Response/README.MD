**AWS Troubleshooting: Investigating an ALB 502 Response**

A 502 from an Application Load Balancer can indicate that the ALB could not obtain a valid response from the target, but the exact cause must be confirmed.

My checks:
- Target health and registered port
- Application process and listening socket
- Target response and application logs
- Protocol mismatch between ALB and target
- Connection resets or malformed responses
- Security Group path from ALB to target

I would correlate the ALB access logs and application logs before declaring a root cause.

#AWS #ALB #Troubleshooting #EC2 #CloudSupport
