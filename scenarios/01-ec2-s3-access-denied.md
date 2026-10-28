AWS Troubleshooting: EC2 Cannot Access Amazon S3
When an application on EC2 receives `AccessDenied` from S3, I would avoid assuming the problem is the network first.
My investigation sequence:

1. Confirm the application is using the expected IAM role.
2. Review the role's identity policy.
3. Check the bucket policy and any explicit denies.
4. If SSE-KMS is used, review KMS permissions and key policy.
5. Confirm the bucket and object ARN/prefix are correct.
6. Retest with a minimal, approved operation and review logs.

Lesson: Authentication, authorization and network reachability are separate layers.
#AWS #S3 #IAM #Troubleshooting #CloudSupport
