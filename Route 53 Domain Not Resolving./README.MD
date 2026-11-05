AWS DNS TROUBLESHOOTING: ROUTE 53 DOMAIN NOT RESOLVING

1. UNDERSTAND DNS

DNS stands for Domain Name System. It translates human-readable domain names into IP addresses or other DNS records.

Example:
www.example.com -> DNS lookup -> Destination IP address.

General DNS resolution:
1. Browser checks DNS cache.
2. Operating system checks DNS cache.
3. Recursive DNS resolver receives query.
4. Resolver may query root DNS servers.
5. Root servers direct it to TLD nameservers.
6. TLD nameservers direct it to authoritative nameservers.
7. Authoritative DNS server returns the record.
8. Resolver returns the answer to the client.
9. Browser attempts to connect to the destination.

2. DOMAIN REGISTRAR VS DNS HOSTING

Domain Registrar: Company through which the domain is registered, such as GoDaddy or Namecheap.

DNS Hosting Provider: Provider hosting authoritative DNS records, such as Amazon Route 53 or Cloudflare.

A domain can be registered at GoDaddy and have DNS hosted in Route 53.

The registrar must delegate the domain to the correct Route 53 nameservers.

3. ROUTE 53 HOSTED ZONES

A hosted zone is a container for DNS records belonging to a domain.

Example domain: example.com.

Records may include:
A: IPv4 address.
AAAA: IPv6 address.
CNAME: Another hostname.
MX: Mail server.
TXT: Text and verification.
NS: Authoritative nameservers.
SOA: Zone authority information.
PTR: Reverse DNS.
SRV: Service location.

Creating a hosted zone does not automatically make Route 53 authoritative. Correct registrar delegation is required.

4. PUBLIC VS PRIVATE HOSTED ZONES

Public Hosted Zone:
- Intended for public DNS resolution.
- Used for public websites, APIs and public-facing ALBs.
- Does not require VPC association.

Private Hosted Zone:
- Used for internal DNS resolution.
- Associated with one or more VPCs.
- Used for internal applications, private APIs and internal load balancers.
- Not publicly resolvable through the public DNS system.

If a public website record is created only in a private hosted zone, public users cannot resolve it.

5. DNS RECORD TYPES

A: Maps hostname to IPv4 address.
AAAA: Maps hostname to IPv6 address.
CNAME: Maps one hostname to another hostname.
MX: Mail exchange.
TXT: Verification, SPF and other text.
NS: Authoritative nameservers.
SOA: Zone metadata.
PTR: Reverse DNS.
SRV: Service discovery.

A record example:
app.example.com -> 13.234.45.67

CNAME example:
www.example.com -> app.example.com

Route 53 Alias:
Can point to supported AWS resources such as an ALB.
Supports the zone apex.
An Alias A record is generally preferred for an ALB.

A standard CNAME cannot be created at the zone apex.

6. TEN COMMON ROUTE 53 TROUBLESHOOTING ISSUES

Issue 1: Incorrect registrar nameservers.
The registrar points to old nameservers instead of the Route 53 hosted zone.
Solution: Compare registrar delegation with Route 53 NS records.

Issue 2: Wrong hosted zone.
Multiple hosted zones exist for the same domain. Records are created in a non-authoritative zone.
Solution: Identify the zone whose NS values match the registrar delegation.

Issue 3: Incorrect DNS record.
Record points to an old IP or incorrect hostname.
Solution: Verify record type and destination.

Issue 4: Public record created in private hosted zone.
Public users cannot resolve private records.
Solution: Create the record in the intended public hosted zone if public access is required.

Issue 5: Incorrect CNAME or Alias.
A standard CNAME cannot be used at the zone apex.
Solution: Use a supported Route 53 Alias record for an ALB.

Issue 6: DNS caching and TTL.
Resolvers may continue using cached answers.
Solution: Check TTL and authoritative answers, and allow caches to expire.

Issue 7: Incorrect A or AAAA record.
IPv4 may work while IPv6 points to an incorrect destination.
Solution: Check A and AAAA records separately.

Issue 8: DNSSEC misconfiguration.
Incorrect DS or DNSKEY configuration may cause SERVFAIL.
Solution: Validate DNSSEC delegation and signing configuration.

Issue 9: Route 53 failover configuration.
Incorrect routing policies or health checks can send users to unavailable destinations.
Solution: Verify primary/secondary records and health checks.

Issue 10: Domain expiration or registration issue.
Hosted zone exists but domain delegation is no longer working.
Solution: Check registrar and registration status.

7. IMPORTANT LINUX DNS COMMANDS

Basic lookup:
nslookup example.com

Detailed DNS:
dig example.com

Check nameservers:
dig NS example.com

Check IPv4:
dig example.com A

Check IPv6:
dig example.com AAAA

Check MX:
dig MX example.com

Check TXT:
dig TXT example.com

Trace delegation:
dig +trace example.com

Google DNS:
dig @8.8.8.8 example.com

Cloudflare DNS:
dig @1.1.1.1 example.com

Check HTTP:
curl -I https://example.com

Detailed HTTPS:
curl -v https://example.com

8. UNDERSTANDING DIG +TRACE

dig +trace example.com helps investigate the DNS delegation path:
- Root referral.
- TLD delegation.
- Authoritative nameservers.
- Authoritative record response.

If delegation is incorrect, investigate the registrar.
If authoritative DNS is correct but the record is missing, investigate the hosted zone.

9. DNS RESPONSE CODES

NOERROR: Query completed without DNS error. Check the answer section.
NXDOMAIN: Queried name does not exist.
SERVFAIL: Resolver could not complete the query. Investigate DNSSEC, delegation and authoritative DNS.
REFUSED: DNS server refused the query.
NOERROR with empty answer: Name may exist but requested record type may not.
TIMEOUT: No DNS response received in time.

10. DNS RESOLUTION VS APPLICATION AVAILABILITY

DNS resolution and application availability are different stages.

Example:
www.example.com resolves successfully to the intended ALB, but the website is unavailable.

Investigate:
- TCP connectivity.
- Security Groups.
- NACLs.
- ALB listeners.
- Target Group health.
- TLS certificate.
- Application logs.

Useful commands:
nc -vz www.example.com 443
curl -Iv https://www.example.com
curl -4 -I https://www.example.com
curl -6 -I https://www.example.com

HTTP response examples:
200: Request succeeded.
403: Access denied.
404: Resource not found.
502: Backend or proxy error.
503: Service unavailable.

11. ROUTE 53 PRODUCTION TROUBLESHOOTING WORKFLOW

1. Check domain registration and expiration.
2. Verify registrar nameservers.
3. Compare nameservers with Route 53 NS records.
4. Identify the correct authoritative hosted zone.
5. Verify public or private hosted zone.
6. Check record type and target.
7. Check TTL and DNS caching.
8. Run dig and nslookup.
9. Run dig +trace.
10. Compare multiple public resolvers.
11. Investigate DNSSEC if SERVFAIL occurs.
12. Test TCP connectivity.
13. Check TLS and certificates.
14. Verify ALB listener and Target Group health.
15. Check application logs.
16. Confirm recovery from multiple networks.

12. DNSSEC

DNSSEC means Domain Name System Security Extensions.
It helps validate the authenticity and integrity of DNS responses.

Commands:
dig example.com DS
dig example.com DNSKEY
dig +dnssec example.com

Incorrect DS or DNSKEY configuration can cause validating resolvers to return SERVFAIL.

13. ROUTE 53 BEST PRACTICES

- Avoid unnecessary duplicate public hosted zones.
- Keep registrar nameservers synchronized.
- Use Alias records for supported AWS resources.
- Choose TTL based on the workload and migration requirements.
- Configure health checks and failover carefully.
- Use IAM least privilege and MFA.
- Monitor domain expiration.
- Audit DNS changes.
- Use private hosted zones for internal services.
- Avoid exposing internal infrastructure unnecessarily.

14. QUICK ROOT CAUSE REFERENCE

Domain doesn't resolve: Registrar or nameserver delegation.
NS records don't match: Registrar configuration.
Record exists but isn't returned: Wrong hosted zone.
Public users cannot resolve record: Private hosted zone.
Old IP returned: Stale cache or incorrect record.
Root domain CNAME issue: Use Alias record.
IPv4 works but IPv6 fails: AAAA or IPv6 connectivity.
SERVFAIL: DNSSEC or authoritative DNS.
NXDOMAIN: Domain name, record or delegation.
DNS works but website fails: TCP, TLS, ALB or application.
HTTP 502: Backend or reverse proxy.
HTTP 503: Target availability or application capacity.

15. OFFICIAL AWS DOCUMENTATION

Route 53 Developer Guide:
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html

Hosted Zones:
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html

Alias vs Non-Alias:
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html

DNS Failover:
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html

Route 53 Troubleshooting:
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/troubleshooting.html

KEY LESSON:
DNS resolution and application availability are two separate troubleshooting stages. Always identify whether the failure is at domain registration, delegation, authoritative DNS, DNS record configuration, network connectivity, TLS or application level.

#AWS #Route53 #DNS #Troubleshooting #AWSNetworking #CloudSupport #CloudEngineering #DevOps #CloudComputing
