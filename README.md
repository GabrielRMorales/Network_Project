Network Infrastructure Homelab

Host System 1
Operating System: Windows 10 Pro
CPU: Intel i7-10700KF (8 Cores, 16 Logical Processors)
RAM: 32 GB DDR4
Storage: 1TB NVMe SSD
Hypervisor: Oracle VirtualBox 7.1.4

VMs:

-pfSense
Type: BSD / FreeBSD (64-bit)
RAM: 1024 MB (1 GB is enough)
CPU: 1 core
Disk: 10 GB (dynamic)

-Ubuntu
Type: Linux / Ubuntu (64-bit)
RAM: 4 GB
CPU: 2 cores
Disk: 25 GB (dynamic)
Network: 1 adapter → Internal Network → vlan10-servers

-for the pfSense VM, ensure under Settings>Network the Adapters are set as:
Adapter 1-Bridged Adapter
Adapter 2-Internal Network vlan10-servers
Adapter 3-Internal Network vlan20-trusted
Adapter 4-Internal Network vlan30-dmz

-once pfSense is installed, reboot it. Hit 2 to assign the interface IP addresses
-for pfSense set the LAN interface as 10.0.10.1 /24 and its name as "SERVERS". Use DHCP and set the IP range as: 10.0.10.100-200
Select 2 again to configure the other interface IP addresses as follows:

OPT1 (VLAN20-Trusted)
Enable DHCP: yes (range 10.0.20.100 to 10.0.20.200)
Description: TRUSTED
IPv4 Configuration: Static
IPv4 Address: 10.0.20.1 /24
Save → Apply Changes
Do not revert to HTTP as the webConfigurator protocol

OPT2 (VLAN30-DMZ)
Enable DHCP: yes (range 10.0.30.100 to 10.0.30.200)
Description: DMZ
IPv4 Configuration: Static
IPv4 Address: 10.0.30.1 /24
Save → Apply Changes
Do not revert to HTTP as the webConfigurator protocol

-Optional: Ping 10.0.10.1 to verify the network connection to the pfSense VM (assuming pfSense VM is running) then
ping out to 8.8.8.8 to verify the internet routing works through pfSense.
-Once Ubuntu VM is running, navigate to https://10.0.10.1 to access the pfSense web portal. 
Change the hostname to "ubuntu-server" and use this configuration:
Domain: lab.home
Primary DNS Server: 1.1.1.1
Secondary DNS Server: 8.8.8.8
Override DNS: Checked

Additional settings can be left as the default/blank. Change the default Admin password.

Proceed to set additional pfSense configurations

-Optional (if interfaces names weren't configured earlier) Configure Interface Names
Navigate to Interfaces>LAN/OPT1/OPT2
In the Description field, change the names as below:
LAN-SERVERS
OPT1-TRUSTED
OPT2-DMZ

1. DHCP Static Lease

Under Services>DHCP Server>LAN, change the Available range to 10.0.10.100-10.0.10.200 (it should match what was set earlier on the pfSense VM). Ensure a similar configuration has been done for VLAN20 and VLAN30.

For VLAN20 (Services>DHCP Server>OPT1), the IP range would be 10.0.20.100 to 10.0.20.200
For VLAN30 (Services>DHCP Server>OPT2), the IP range would be 10.0.30.100 to 10.0.30.200

Then set up the DHCP Static IP mapping. Include the Ubuntu VM's MAC address, hostname (ie ubuntu-server), and IP address.  Ensure the IP is outside the dynamic DHCP range (ex. 10.0.10.10)

2. Firewall Rules

Go to Firewall>Aliases>Add

Alias 1:

Name: RFC1918
Type: Network
Networks: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
Description: "All private IP ranges"

Alias 2:

Name: MGMT_PORTS
Type: Port
Ports: 22 (SSH), 443 (HTTPS/WebUI), 80 (HTTP)
Description: "Common management ports"

Alias 3:

Name: UBUNTU_SERVER
Type: Host
IP: 10.0.10.10
Description: "Ubuntu server VM"

Then go to Firewall>Rules>LAN
Click the Add (up arrow) and set these rules:
Rule 1 — Allow UBUNTU_SERVER to reach WAN (for updates):

Action: Pass
Interface: LAN
Protocol: TCP
Source: Single host → 10.0.10.10
Destination: not RFC1918 (check the "not" box)
Dest port: 80, 443
Description: "Ubuntu server — internet updates only"

Rule 2 — Allow LAN to access pfSense WebUI:

Action: Pass
Interface: LAN
Protocol: TCP
Source: LAN subnet
Destination: LAN address (this means pfSense's own IP)
Dest port: 443
Description: "Allow WebUI access from VLAN10"

Rule 3 — Block LAN to all other private ranges:

Action: Block
Interface: LAN
Protocol: any
Source: LAN subnet
Destination: RFC1918
Description: "Block VLAN10 reaching other private networks"

Rule 4 — Default deny (optional but good practice):

Action: Block
Interface: LAN
Protocol: any
Source: any
Destination: any
Description: "Default deny all"

Delete the Default allow LAN to any (IPv4) and Default allow LAN IPv6 to any rules. Ensure the remaining rules are set in the proper order of:
-Allow Ubuntu VM to access the WAN
-Allow VLAN10 to access the pfSense WebUI
-Deny VLAN10 access to private networks
-Deny by default

WAN Firewall rules can remain unchanged. Note that for the purposes of this homelab, the WAN is actually a private home network that then sends data to a router and the internet. Because of this, under Interfaces>WAN, these two options must be unchecked:
-Block private networks and loopback addresses
-Block bogon networks 

Since the WAN interface is on a private 192.168.1.x network rather than a public IP, pfSense would otherwise block traffic on this interface due to RFC1918 filtering.

Then go to Firewall>Rules>OPT1/TRUSTED to set rules for VLAN20
Click the Add (up arrow) and set these rules:
Rule 1 — Allow TRUSTED to reach pfSense management:

Action: Pass
Protocol: TCP/UDP
Source: TRUSTED subnet
Destination: TRUSTED address
Dest port: 443, 53
Description: "Allow WebUI and DNS from TRUSTED"

Rule 2 — Allow TRUSTED to reach VLAN 10 servers:

Action: Pass
Protocol: TCP
Source: TRUSTED subnet
Destination: 10.0.10.10
Dest port: 80, 443, 22
Description: "Allow TRUSTED to reach servers"

Rule 3 — Allow TRUSTED to internet:

Action: Pass
Protocol: TCP
Source: TRUSTED subnet
Destination: not RFC1918
Dest port: 80, 443
Description: "Allow TRUSTED internet access"

Rule 4 — Allow ICMP:

Action: Pass
Protocol: ICMP
Source: TRUSTED subnet
Destination: any
Description: "Allow ICMP from TRUSTED"

Rule 5 — Block TRUSTED to RFC1918:

Action: Block
Protocol: any
Source: TRUSTED subnet
Destination: RFC1918
Description: "Block TRUSTED to other private ranges"

Rule 6 — Default deny:

Action: Block
Protocol: any
Source: any
Destination: any
Description: "Default deny all"

Then go to Firewall>Rules>OPT2/DMZ to set rules for VLAN30
Click the Add (up arrow) and set these rules:

Rule 1 — Allow DMZ DNS (so that DMZ VMs can query pfSenses's DNS resolver on the DMZ interface):

Action: Pass
Protocol: TCP/UDP
Source: DMZ subnet
Destination: DMZ address
Dest port: 53
Description: "Allow DNS from DMZ"

Rule 2 — Allow DMZ to internet only:

Action: Pass
Protocol: TCP
Source: DMZ subnet
Destination: not RFC1918
Dest port: 80, 443
Description: "DMZ internet only"

Rule 3 — Block DMZ to everything else:

Action: Block
Protocol: any
Source: DMZ subnet
Destination: any
Description: "Block DMZ to all internal networks"

Test access by generating a new VM in both VLAN20 and 30.
Ensure that the new VM is on the proper network.
-Go to Settings>Network>Adapter 1
-Select vlan20-trusted or vlan30-dmz

TESTS FOR VLAN 20-run these:
# internet access
ping -c 4 1.1.1.1 (passes)

# DNS
nslookup google.com (passes)

# Reach VLAN 10 Ubuntu server
ping 10.0.10.10 (passes)

# pfSense WebUI
curl -k https://10.0.20.1 (passes)

# DMZ is not accessible
ping 10.0.30.100 (fails)

# Scan VLAN10 VM with nmap to see open ports

First set up SSH to listen by running:
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
Then run:
nmap -Pn -p 22,80,443 10.0.10.10

This should result in:
-Port 22 State: open
-Port 80 State: closed
-Port 443 State: closed

TESTS FOR VLAN 30-run these:

# DNS
nslookup google.com (passes)

# reach Google
curl -I https://www.google.com (passes)

# Scan VLAN10 VM with nmap to see open ports
nmap -Pn -p 22,80,443 10.0.10.10
This should result in:
-Port 22 State: filtered
-Port 80 State: filtered
-Port 443 State: filtered

3. DNS Resolver

Navigate to Services>DNS Resolver
The configuration should be:
Enable DNS Resolver: Checked
Network Interfaces: All
Outgoing Network Interfaces: All
DNSSEC: Checked

Scroll down to Host Overrides and click Add. Use these values:
Host: ubuntu-server
Domain: lab.home
IP address: 10.0.10.10
Description: "VLAN10 Ubuntu server"

Run these tests (from VLAN20) to verify success:

# SSHing into 10.0.10.10 should work from VLAN 20
ssh user@ubuntu-server.lab.home

# Should work from VLAN 20
ping ubuntu-server.lab.home

4. NAT reflection-used to resolve asymmetric routing. Without this, a request to an internal VLAN by way of an external WAN IP address will NAT (through pfSense) to the internal VLAN IP address. However, the return packet will bypass pfSense and attempt to send directly to the source IP address, causing the TCP handshake to fail and the connection to be dropped.

On the pfSense WebUI, navigate to System → Advanced → Firewall & NAT > Network Address Translation

Configure as below:
NAT Reflection mode for port forwards: NAT + Proxy
Enable NAT Reflection for 1:1 NAT: Checked
Enable automatic outbound NAT for Reflection: Checked

Setup a Port Forwarding rule to work with NAT
Navigate to Firewall → NAT → Port Forward → Add:

Use these values:
Interface: WAN
Protocol: TCP
Destination: WAN address
Destination port:  22
Redirect target IP:  10.0.10.10
Redirect target port: 22
Description: "Port forward SSH to Ubuntu server"

The NAT Port Forwading and inbound SSH redirection can be verified by running from the WAN IP (in this case, the host Windows 10 computer*):
ssh gmonitor@192.168.1.217

*Because a private IP range was set on the WAN interface, RFC1918 filtering created a conflicting rule when testing from internal VLANS. This was specifically due to the private IP range and wouldn't be expected from a public WAN IP. As a workaround, NAT port forwarding was verified through the use of SSH from the Windows host machine to 192.168.1.217. This SSH requested was successfully redirected to the 10.0.10.10 address and allowed an SSH connection to the Ubuntu VM.

Troubleshooting:

1. After setting the Firewall rules for VLAN10, these failed (but should have succeeded):
-ping to 1.1.1.1
-curl https://10.0.10.1 (pfSense WebUI)
-nslookup google.com 10.0.10.1
Cause: The RFC1918 block rule was catching traffic destined for 10.0.10.1 (pfSense's LAN IP) before it could reach pfSense's DNS resolver and WebUI, since 10.0.10.1 falls within a private IP range.
Resolution: additional firewall rules were required

Additional Rule 1 (added to the top)
Action: Pass
Protocol: ICMP
ICMP type: any
Source: LAN subnet
Destination: any
Description: "Allow ICMP ping from VLAN10"

Additional Rule 2
Action: Pass
Protocol: TCP/UDP
Source: LAN subnet
Destination: LAN address
Destination port: DNS (53)
Description: "Allow DNS queries to pfSense"

Verified by running:
curl -I http://1.1.1.1

curl -I https://google.com

ping -c 4 1.1.1.1

ping -c 4 10.0.10.1

Each passed

2. When setting Firewall rules, Port Aliases failed to populate in the Port Range field.
Note that IP Aliases appeared. Attempted to re-add the Port Aliases, but this didn't work.
Resolution: This appears to be a known UI limitation in pfSense 2.6.0 where Port aliases do not populate in the port range dropdown. As a workaround, individual port rules were created manually for each required port