AWS VPC Peering Troubleshooting: VPC Peering Connection Is Active, but Traffic Fails

An active VPC Peering connection does not guarantee communication between two VPCs. You must verify the complete network configuration.

1. VPC CIDR ranges: Ensure both VPCs have non-overlapping CIDR ranges. AWS does not support peering between VPCs with overlapping CIDRs.

2. Route Tables: Add routes on both sides. VPC A must have a route to VPC B's CIDR through the peering connection, and VPC B must have a return route to VPC A's CIDR.

3. Security Groups: Verify inbound and outbound rules. For example, if EC2-A accesses EC2-B on TCP port 8080, EC2-B must allow TCP 8080 from VPC A's CIDR. Security Groups are stateful.

4. Network ACLs: Verify inbound and outbound rules on both subnets. NACLs are stateless, so request and response traffic, including ephemeral ports, must be permitted.

5. Host Firewall and Application: Check Linux firewalls such as UFW, firewalld and iptables. Verify that the application is running and listening on the correct private IP and port using sudo ss -tulnp.

6. DNS Resolution: If private IP connectivity works but hostname connectivity fails, check VPC DNS settings, peering DNS options and Route 53 Private Hosted Zone associations.

7. Non-transitive Peering: VPC Peering does not support transitive routing. If VPC A is peered with VPC B, and VPC B is peered with VPC C, VPC A cannot automatically communicate with VPC C. Consider direct peering or AWS Transit Gateway.

8. VPC Flow Logs: Analyze ACCEPT and REJECT traffic records to identify potential network-level blockers. ACCEPT does not guarantee application connectivity.

9. Reachability Analyzer: Analyze the network path between EC2 network interfaces to identify configuration-based connectivity issues.

Practical troubleshooting commands:

ping 10.1.1.10
nc -vz 10.1.1.10 8080
curl -v http://10.1.1.10:8080
sudo ss -tulnp
sudo systemctl status nginx

Recommended troubleshooting order:
1. Verify peering status.
2. Check CIDR ranges.
3. Verify routes on both sides.
4. Check Security Groups.
5. Check NACLs.
6. Check host firewalls.
7. Verify application status and listening ports.
8. Check DNS if using hostnames.
9. Analyze traffic using VPC Flow Logs and Reachability Analyzer.

Key takeaway: An active VPC Peering connection establishes connectivity between VPCs at the logical network level, but routing, security controls and application configuration must also be correct.
