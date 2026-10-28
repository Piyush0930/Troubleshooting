**AWS Troubleshooting: EC2 Application Is Not Reachable**

I use a layered approach to isolate connectivity issues rather than changing multiple settings at once.

Check in this order:
- Is the application running and listening on the expected port?
- Does the OS firewall allow the connection?
- Does the Security Group allow the required source and port?
- Do the subnet route table and internet gateway/NAT design match the intended path?
- Are NACL rules blocking either direction?
- Is the instance publicly addressable if direct public access is intended?

**Lesson:** Verify the path from the application outward and from the client inward.

#AWS #EC2 #Networking #Troubleshooting
