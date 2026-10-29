# AWS Troubleshooting: EC2 Application Is Not Reachable

I use a layered approach to isolate connectivity issues rather than
changing multiple settings at once. Each layer is checked and either
ruled out or confirmed before moving to the next.

---

## Symptom
Users cannot reach an application hosted on an EC2 instance.
The browser or client shows a timeout, "connection refused", or a
5xx error.

## First clue: timeout vs refused

| What the client sees | What it usually means |
|----------------------|-----------------------|
| **Connection timed out** | Packets are being dropped on the way: Security Group, NACL, route table, or no public IP |
| **Connection refused** | Packets reached the server, but nothing is listening on that port, or the OS firewall is rejecting them |
| **502 / 503 / 504** | Traffic reached a load balancer or proxy, but the backend is unhealthy or not responding |

This one observation often narrows the problem to the right layer
before any console check.

---

## Troubleshooting flow

```
Client → DNS → Internet Gateway → Route table → NACL
       → Security Group → OS firewall → Application (listening port)
```

I check from the **application outward** (is it listening?) and from the
**client inward** (can the packet get there?). The problem lies where
those two checks meet.

---

## Step 1: Is the application running and listening on the expected port?

Log in to the instance (SSH, or Session Manager if SSH is not available).

```bash
# Is the service running?
sudo systemctl status nginx

# Which ports is the server listening on?
sudo ss -tulnp

# Test the app locally on the instance itself
curl -I http://localhost:80
```

What to look for:
- Service is **inactive or failed**: check logs (`journalctl -u nginx`,
  `/var/log/nginx/error.log`).
- Port shows `127.0.0.1:80` instead of `0.0.0.0:80` or `*:80`: the app is
  bound to localhost only and will reject outside traffic.
- Wrong port: the app listens on 8080 but the Security Group opens 80.

If `curl localhost` fails, the problem is the application, not the network.

---

## Step 2: Does the OS firewall allow the connection?

Even if the AWS network is correct, the firewall inside the OS can block it.

```bash
# Ubuntu
sudo ufw status

# Amazon Linux / RHEL / CentOS
sudo systemctl status firewalld
sudo firewall-cmd --list-all

# Raw iptables rules
sudo iptables -L -n
```

What to look for: the application port is not allowed, or a default
DROP/REJECT policy applies. A rejected packet often shows up as
"connection refused", while a dropped one shows as a timeout.

---

## Step 3: Does the Security Group allow the required source and port?

EC2 console → Instance → **Security** tab → Inbound rules.

Check:
- Correct **port** (80, 443, or custom application port)
- Correct **protocol** (TCP)
- Correct **source**: `0.0.0.0/0` for public access, a specific CIDR, or
  another Security Group
- The right Security Group is actually attached to the instance
  (and to the load balancer, if there is one)

Key facts:
- Security Groups are **stateful**: return traffic is allowed
  automatically.
- Security Groups only have **allow** rules, with no deny.
- If the source is wrong (for example your office IP changed), the
  result is a timeout.

---

## Step 4: Do the route table and gateway design match the intended path?

VPC console → Subnet → **Route table** tab.

| Intended access | Required route |
|-----------------|----------------|
| Public instance, inbound from internet | `0.0.0.0/0` → **Internet Gateway** (`igw-...`) |
| Private instance, outbound only | `0.0.0.0/0` → **NAT Gateway** (`nat-...`) in a public subnet |
| Private instance via load balancer | ALB in public subnets, instance in private subnets |

Check:
- The subnet is associated with the **correct** route table (not the
  main table by accident)
- The Internet Gateway is **attached** to the VPC
- The NAT Gateway sits in a **public** subnet with its own route to the IGW
- A subnet is "public" only if its route table points to an IGW

---

## Step 5: Are NACL rules blocking either direction?

VPC console → Subnet → **Network ACL** tab.

NACLs are **stateless**. Return traffic is not automatically allowed, so
both directions must be correct.

Check:
- **Inbound** allows the application port from the client
- **Outbound** allows **ephemeral ports (1024–65535)** back to the client
- Rule numbers: rules are evaluated in order, and a lower-numbered
  **DENY** overrides a higher-numbered ALLOW
- The default NACL allows everything, but a custom NACL denies everything
  until rules are added

---

## Step 6: Is the instance publicly addressable if direct public access is intended?

Check:
- The instance has a **public IPv4 address** or an **Elastic IP**
- The subnet's "auto-assign public IP" setting, if relying on it
- A public IP is **not** kept after stop/start unless it is an Elastic IP
- If the instance is private by design, access must go through a load
  balancer, VPN, bastion, or Session Manager, not a direct public IP

---

## Extra checks when a load balancer is involved

- **Target group health**: unhealthy targets cause 502/503
- Health check path, port, and expected status code match the app
- The **instance Security Group allows traffic from the ALB Security
  Group**
- The ALB listener port and target port are correct

## Extra checks for DNS

- Route 53 record points to the correct IP or ALB
- TTL: an old record may still be cached
- `nslookup yourdomain.com` and `dig yourdomain.com` confirm what the
  client actually resolves

---

## Tools that speed this up

| Tool | What it shows |
|------|---------------|
| **VPC Reachability Analyzer** | Finds the exact component (SG, NACL, route) blocking a path between two resources |
| **VPC Flow Logs** | `ACCEPT` or `REJECT` for each connection, which tells you whether the traffic arrived and who rejected it |
| **CloudWatch metrics** | CPU, status checks, network in/out |
| **EC2 Instance Connect / Session Manager** | Access the instance when SSH is not working |

Testing from the client side:

```bash
curl -v http://<public-ip>:80
nc -zv <public-ip> 80
```

---

## Quick reference: symptom to likely cause

| Symptom | Most likely layer |
|---------|-------------------|
| Timeout from the internet | Security Group, NACL, missing IGW route, or no public IP |
| Refused | Application not listening, or OS firewall rejecting |
| Works on the instance via `localhost`, fails from outside | Bound to 127.0.0.1, OS firewall, or network layer |
| Worked yesterday, fails today | Public IP changed after stop/start, or a recent SG/NACL/route change |
| Works from one client, not another | Source CIDR in SG or NACL |
| 502/503 behind ALB | Unhealthy targets, or the SG doesn't allow the ALB |

---

## Example (illustrative)

**Issue:** A web application on an EC2 instance timed out from the internet.

**Investigation:**
1. `curl localhost` returned the page, so the application was fine.
2. OS firewall was inactive.
3. Security Group allowed port 443 but not port 80, which the client was using.

**Fix:** Added an inbound rule for TCP 80 from the required source and
retested. *(Replace this with a real scenario you handled, with client
names, IPs, and account IDs removed.)*

---

## Prevention
- CloudWatch alarms on `StatusCheckFailed` and ALB `UnHealthyHostCount`
- Use Elastic IPs or a load balancer instead of relying on auto-assigned
  public IPs
- Review Security Group, NACL, and route changes through a change
  approval process
- Keep Flow Logs enabled on critical VPCs
- Document the intended network path for each application

---

## Lesson
Verify the path **from the application outward** and **from the client
inward**. Change one thing at a time, so you always know what fixed it.

`#AWS` `#EC2` `#Networking` `#Troubleshooting`
