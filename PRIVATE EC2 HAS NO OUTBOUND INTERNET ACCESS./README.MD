AWS NETWORKING TROUBLESHOOTING: PRIVATE EC2 HAS NO OUTBOUND INTERNET ACCESS

1. UNDERSTAND THE PROBLEM

A private EC2 instance can be running normally but still lack outbound internet access. Common symptoms include:
- yum, dnf or apt cannot download packages.
- Application cannot download dependencies.
- External APIs are unreachable.
- curl https://google.com times out.
- DNS resolution may or may not work.

2. PUBLIC VS PRIVATE SUBNETS

Public subnet: Has a route to an Internet Gateway.
Private subnet: Does not provide direct internet access to its instances through an Internet Gateway.
Private EC2 instances can access the internet through an approved egress mechanism such as a Public NAT Gateway.

3. HOW NAT GATEWAY WORKS

Example:
VPC: 10.0.0.0/16
Public subnet: 10.0.1.0/24
Private subnet: 10.0.2.0/24
NAT Gateway private IP: 10.0.1.50
Private EC2: 10.0.2.10
Internet Gateway: Attached to VPC.

Traffic flow:
1. Private EC2 sends an outbound request.
2. Private subnet route table sends 0.0.0.0/0 traffic to NAT Gateway.
3. NAT Gateway translates the source address and port.
4. Public NAT Gateway forwards traffic through Internet Gateway.
5. Destination server receives the request.
6. Response returns through Internet Gateway and NAT Gateway.
7. NAT Gateway translates the response and sends it to private EC2.

NAT Gateway enables outbound connections. It does not allow unsolicited inbound internet connections to private EC2.

4. THREE IMPORTANT ROUTING CONFIGURATIONS

Private subnet route table:
10.0.0.0/16 -> local
0.0.0.0/0 -> NAT Gateway

Public subnet route table:
10.0.0.0/16 -> local
0.0.0.0/0 -> Internet Gateway

NAT Gateway:
- Must be deployed in the correct VPC and public subnet.
- Public NAT Gateway requires an Elastic IP.
- Its subnet needs a valid route to the Internet Gateway.
- Its status should be Available.

5. PUBLIC NAT VS PRIVATE NAT

Public NAT Gateway:
- Uses an Elastic IP.
- Provides outbound IPv4 internet connectivity through an Internet Gateway.
- Commonly used for private EC2 internet access.

Private NAT Gateway:
- Does not use an Elastic IP.
- Used for private network connectivity through supported routing architectures.
- Does not directly provide internet access.

6. TEN COMMON TROUBLESHOOTING AREAS

A. Missing private subnet default route.
Check whether 0.0.0.0/0 points to the intended NAT Gateway.

B. NAT Gateway unavailable.
Check NAT Gateway status, subnet, VPC and Elastic IP.

C. Public subnet missing Internet Gateway route.
Verify 0.0.0.0/0 points to the attached Internet Gateway.

D. Incorrect Security Group outbound rules.
Allow required outbound traffic, such as TCP 443, where appropriate.

E. Restrictive NACL.
NACLs are stateless. Check outbound application traffic and inbound return traffic, including ephemeral ports.

F. DNS resolution failure.
Use nslookup and dig. Verify VPC DNS resolution and DNS hostnames settings.

G. Destination service restrictions.
The external destination may reject or block connections.

H. NAT Gateway packet drops or port allocation issues.
Inspect CloudWatch NAT Gateway metrics.

I. Incorrect routing or subnet association.
Check the actual route table associated with the instance subnet.

J. Network firewall or proxy restrictions.
Check any additional centralized egress controls.

7. LINUX TROUBLESHOOTING COMMANDS

Test DNS:
nslookup google.com
dig google.com

Test HTTPS:
curl -Iv --connect-timeout 10 https://google.com

Test HTTP:
curl -I --connect-timeout 10 http://example.com

Test TCP:
nc -vz google.com 443

Check resolver:
cat /etc/resolv.conf

8. AWS CLI COMMANDS

Check EC2:
aws ec2 describe-instances --instance-ids i-xxxxxxxx --region ap-south-1

Check route table:
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-xxxxxxxx --region ap-south-1

If no explicit subnet association is returned, inspect the VPC main route table.

Check NAT Gateway:
aws ec2 describe-nat-gateways --nat-gateway-ids nat-xxxxxxxx --region ap-south-1

Check Internet Gateway:
aws ec2 describe-internet-gateways --filters Name=attachment.vpc-id,Values=vpc-xxxxxxxx --region ap-south-1

Check Security Group:
aws ec2 describe-security-groups --group-ids sg-xxxxxxxx --region ap-south-1

Check NACL:
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=subnet-xxxxxxxx --region ap-south-1

Check VPC DNS support:
aws ec2 describe-vpc-attribute --vpc-id vpc-xxxxxxxx --attribute enableDnsSupport --region ap-south-1

Check VPC DNS hostnames:
aws ec2 describe-vpc-attribute --vpc-id vpc-xxxxxxxx --attribute enableDnsHostnames --region ap-south-1

These commands require appropriate IAM permissions.

9. SECURITY GROUP VS NACL

Security Groups:
- Stateful.
- Attached to network interfaces.
- Control inbound and outbound traffic.
- Return traffic is automatically allowed for permitted connections.

Network ACLs:
- Stateless.
- Applied at subnet level.
- Require explicit inbound and outbound rules.
- Check ephemeral ports for return traffic.

NAT Gateway does not have a Security Group.

10. VPC FLOW LOGS

VPC Flow Logs capture IP traffic metadata.

ACCEPT: Traffic was accepted at the monitored network interface.
REJECT: Traffic was rejected.

Example:
10.0.2.10 142.250.183.14 50000 443 ACCEPT

Use Flow Logs to investigate:
- Source and destination IP.
- Source and destination ports.
- Traffic direction.
- ACCEPT and REJECT records.
- Relevant timestamps.

Flow Logs are not packet captures and do not show full HTTP requests or prove that the destination returned an application response.

11. NAT GATEWAY CLOUDWATCH METRICS

BytesInFromSource: Bytes received from sources.
BytesOutToDestination: Bytes sent toward destinations.
BytesInFromDestination: Bytes received from destinations.
BytesOutToSource: Bytes returned toward sources.
ErrorPortAllocation: Port allocation errors.
PacketsDropCount: Dropped packets.
ActiveConnectionCount: Active connections.

Correlate metric patterns with Flow Logs and instance-level tests.

12. HIGH AVAILABILITY

Recommended production design:
AZ-a: Private subnet A -> NAT Gateway A in public subnet A.
AZ-b: Private subnet B -> NAT Gateway B in public subnet B.

Benefits:
- Reduces cross-AZ dependency.
- Improves fault isolation.
- Avoids cross-AZ NAT traffic charges in this design.
- Improves resilience to AZ-level failures.

NAT Gateway is an AZ-scoped resource, not a multi-AZ resource.

13. NAT GATEWAY VS VPC ENDPOINTS

Public websites and external APIs: NAT Gateway or approved egress proxy.
S3: S3 Gateway Endpoint.
DynamoDB: DynamoDB Gateway Endpoint.
Systems Manager: Required SSM Interface Endpoints.
CloudWatch: CloudWatch Interface Endpoints.
ECR: ECR Interface Endpoints and required S3 endpoint.

VPC Endpoints can provide private access to supported AWS services without routing that traffic through NAT.

14. COST OPTIMIZATION

NAT Gateway charges generally include hourly usage and data processing, plus applicable data transfer costs.

Optimization:
- Use VPC Endpoints where appropriate.
- Avoid unnecessary external internet requests.
- Use internal package repositories or caching.
- Monitor NAT Gateway usage.
- Avoid unnecessary cross-AZ NAT traffic.

15. PRODUCTION TROUBLESHOOTING WORKFLOW

1. Confirm EC2 instance is running.
2. Identify its subnet.
3. Identify the associated route table.
4. Check 0.0.0.0/0 route to NAT Gateway.
5. Check NAT Gateway state is Available.
6. Verify NAT Gateway subnet is public.
7. Verify Elastic IP association.
8. Check public subnet route to Internet Gateway.
9. Confirm Internet Gateway attachment.
10. Review EC2 Security Group outbound rules.
11. Review public and private subnet NACLs.
12. Test DNS resolution.
13. Test HTTP and HTTPS connectivity.
14. Inspect VPC Flow Logs.
15. Review NAT Gateway CloudWatch metrics.
16. Identify root cause.
17. Obtain approval before changing production networking.
18. Apply the fix.
19. Verify recovery and monitor.

16. PRODUCTION INCIDENT EXAMPLE

Client reports that a private EC2 instance cannot download security updates.

DNS resolution succeeds, but curl to an external HTTPS destination times out.

Investigation:
- NAT Gateway is Available.
- NAT Gateway has an Elastic IP.
- Public subnet has a default route to the Internet Gateway.
- Internet Gateway is attached.
- Private subnet route table has no default route to NAT Gateway.

Root cause: Missing 0.0.0.0/0 route in the private subnet route table.

Approved fix: Add the default route to the correct NAT Gateway.

Verification:
curl -I https://google.com

A successful HTTP response confirms that this connectivity test works.

17. IMPORTANT TAKEAWAYS

- A private EC2 instance does not require a public IP for NAT-based outbound internet access.
- A NAT Gateway must be in the correct subnet with the correct routing.
- The private subnet needs a route to NAT Gateway.
- The public NAT Gateway subnet needs a route to Internet Gateway.
- Security Groups are stateful; NACLs are stateless.
- DNS and internet connectivity are separate troubleshooting areas.
- VPC Flow Logs and CloudWatch metrics help identify networking problems.
- Use VPC Endpoints for supported AWS services where appropriate.
- Deploy NAT Gateways per AZ for resilient production designs.
- Never change production route tables without understanding impact and obtaining approval.

OFFICIAL AWS DOCUMENTATION:
https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html
https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html
https://docs.aws.amazon.com/vpc/latest/userguide/vpc-flow-logs.html
https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html
https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-cloudwatch.html

#AWS #NATGateway #VPC #Networking #Troubleshooting #EC2 #CloudSupport #CloudEngineering #DevOps #AWSNetworking
