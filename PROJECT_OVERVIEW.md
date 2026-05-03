# Network Infrastructure Homelab

A segmented network security and infrastructure lab built using pfSense and Ubuntu virtual machines to simulate enterprise-style VLAN segmentation, firewall policy enforcement, and secure service access control.

---

# Project Objective

Design and implement a virtualized network environment that demonstrates:

- Network segmentation using VLANs
- Firewall rule enforcement and traffic isolation
- Secure internal and external routing
- DNS resolution within segmented networks
- NAT-based service exposure and access control

This project simulates a small enterprise network with isolated trust zones and controlled communication paths.

---

# Architecture Overview

The environment is built around a pfSense firewall acting as the central routing and segmentation layer.

### Network Segments:

- **VLAN 10 – SERVERS (10.0.10.0/24)**
  - Ubuntu server VM
  - Internal services and SSH access

- **VLAN 20 – TRUSTED (10.0.20.0/24)**
  - Administrative / trusted client network
  - Access to servers and internet

- **VLAN 30 – DMZ (10.0.30.0/24)**
  - Isolated network segment
  - Restricted external access only

### Core Flow:

pfSense → VLAN routing + firewall rules → segmented Ubuntu VM environments

---

# Virtual Infrastructure

## Host System
- Windows 10 Pro
- Intel i7-10700KF
- 32 GB RAM
- VirtualBox 7.1.x

---

## Virtual Machines

### pfSense Firewall
:contentReference[oaicite:0]{index=0}
- FreeBSD-based firewall appliance
- Interfaces:
  - WAN (bridged)
  - VLAN10 (SERVERS)
  - VLAN20 (TRUSTED)
  - VLAN30 (DMZ)
- Handles:
  - DHCP per VLAN
  - DNS resolution
  - NAT and port forwarding
  - Firewall policy enforcement

---

### Ubuntu Server VM
:contentReference[oaicite:1]{index=1}
- Acts as internal server host (VLAN10)
- Services:
  - SSH (OpenSSH server)
- Static IP assignment via DHCP reservation

---

# Key Security Design

## VLAN Segmentation

- VLAN10: internal server network
- VLAN20: trusted user/admin network
- VLAN30: isolated DMZ network

Each VLAN is strictly controlled via pfSense firewall rules.

---

## Firewall Policy Model

### TRUSTED (VLAN20)
- Allowed:
  - Internet access (HTTP/HTTPS)
  - Access to internal servers (SSH/HTTP/HTTPS)
  - DNS resolution via pfSense
- Restricted:
  - No direct access to other private networks unless explicitly allowed

### DMZ (VLAN30)
- Allowed:
  - Internet access only (HTTP/HTTPS)
  - DNS resolution
- Restricted:
  - Blocked from all internal VLANs

### SERVERS (VLAN10)
- Accessible from TRUSTED network
- Isolated from DMZ by default

---

# DNS & Name Resolution

pfSense DNS Resolver configured with:

- Domain: `lab.home`
- Host override:
  - `ubuntu-server.lab.home → 10.0.10.10`

Enables internal name-based resolution within VLANs.

---

# NAT & External Access

Configured NAT port forwarding enables controlled external access:

- WAN → SSH forwarding to internal Ubuntu server
- NAT reflection enabled for internal testing consistency

This simulates real-world service exposure while maintaining segmentation.

---

# Validation & Testing

## VLAN20 (TRUSTED)
- Internet connectivity verified
- DNS resolution confirmed
- Access to VLAN10 server via SSH
- Isolation from VLAN30 confirmed

## VLAN30 (DMZ)
- Internet-only access verified
- No access to internal VLANs
- Internal port scans show filtered services

---

# Key Concepts Demonstrated

- Network segmentation using VLANs
- Stateful firewall rule design
- Private IP routing (RFC1918 awareness)
- NAT traversal and port forwarding
- DNS host overrides in internal networks
- Traffic isolation between trust zones

---

# 🧰 Technologies Used

- pfSense firewall
- VirtualBox virtualization
- Ubuntu Server
- DNS Resolver (Unbound)
- NAT + firewall rule engine
- VLAN-based network segmentation

---

# 🚀 Future Enhancements

- Add monitoring stack (Prometheus + Grafana)
- Add IDS/IPS (Suricata on pfSense)
- Expand VLAN topology (IoT / Guest networks)
- Integrate SNMP-based device monitoring

---

# 📌 Summary

This project demonstrates the design and implementation of a segmented, policy-driven network infrastructure using virtualized networking. It focuses on real-world concepts including firewall architecture, secure routing, and network isolation strategies commonly used in enterprise environments.