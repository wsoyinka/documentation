---
title: NSD Authoritative DNS
author: Neel Chauhan
contributors: Steven Spencer, Ganna Zhyrnova
tested_with: 9.4
tags:
  - dns
---

An alternative to BIND, [NSD](https://www.nlnetlabs.nl/projects/nsd/about/) (Name Server Daemon) is a modern authoritative-only DNS server maintained by [NLnet Labs](https://www.nlnetlabs.nl/).

## Prerequisites and assumptions

- A server running Rocky Linux
- Able to use `firewalld` for creating firewall rules
- A domain name or internal recursive DNS server pointed to your authoritative DNS server

## Introduction

External, or public, DNS servers map hostnames to IP addresses and, in the case of PTR (known as "pointer" or "reverse") records, map the IP addresses to the hostname. This is an essential part of the Internet. It makes your mail server, web server, FTP server, or many other servers and services work as expected no matter where you are.

## Installing and enabling NSD

First, install EPEL:

```bash
dnf install epel-release
```

Next, install NSD:

```bash
dnf install nsd
```

## Configuring NSD

Before making changes to any configuration file, copy the original installed working file, `nsd.conf`:

```bash
cp /etc/nsd/nsd.conf /etc/nsd/nsd.conf.orig
```

That will help in the future if the introduction of errors into the configuration file occurs. It is *always* a good idea to make a backup copy before making changes.

Edit the *nsd.conf* file. The author uses *vi*, but you can substitute your favorite command line editor:

```bash
vi /etc/nsd/nsd.conf
```

Navigate to the bottom and insert the following:

```bash
zone:
    name: example.com
    zonefile: /etc/nsd/example.com.zone
```

Replace `example.com` with the domain name for which you run a nameserver.

Next, create the zone files:

```bash
vi /etc/nsd/example.com.zone
```

The DNS zone files are BIND compatible. In the file, insert:

```bash
$TTL    86400 ; How long should records last?
; $TTL used for all RRs without explicit TTL value
$ORIGIN example.com. ; Define our domain name
@  1D  IN  SOA ns1.example.com. hostmaster.example.com. (
                              2024061301 ; serial
                              3h ; refresh duration
                              15 ; retry duration
                              1w ; expiry duration
                              3h ; nxdomain error ttl
                             )
       IN  NS     ns1.example.com. ; in the domain
       IN  MX  10 mail.another.com. ; external mail provider
       IN  A      172.20.0.100 ; default A record
; server host definitions
ns1    IN  A      172.20.0.100 ; name server definition
www    IN  A      172.20.0.101 ; web server definition
mail   IN  A      172.20.0.102 ; mail server definition
```

If you need help customizing BIND-style zone files, Oracle has [a good introduction to zone files](https://docs.oracle.com/en-us/iaas/Content/DNS/Reference/formattingzonefile.htm).

Save your changes.

## Enabling NSD

Next, allow DNS ports in `firewalld` and enable NSD:

```bash
firewall-cmd --add-service=dns --zone=public
firewall-cmd --runtime-to-permanent
systemctl enable --now nsd
```

Check DNS resolution with the `host` command:

```bash
% host example.com 172.20.0.100
Using domain server:
Name: 172.20.0.100
Address: 172.20.0.100#53
Aliases:

example.com has address 172.20.0.100
example.com mail is handled by 10 mail.another.com.
%
```

## Secondary DNS server

Running one or more secondary authoritative DNS servers is generally the norm. This is especially useful when the primary server goes down. NSD has feature allows for syncing DNS records from a primary server to one or many backup servers.

To enable a backup server, generate the signing keys on the primary zone:

```bash
nsd-control-setup
```

You will need to copy the following files to the backup server in the `/etc/nsd/` directory:

- `nsd_control.key`
- `nsd_control.pem`
- `nsd_server.key`
- `nsd_server.pem`

On all the DNS servers, add the following before the `zone:` directive:

```bash
remote-control:
        control-enable: yes
        control-interface: 0.0.0.0
        control-port: 8952
        server-key-file: "/etc/nsd/nsd_server.key"
        server-cert-file: "/etc/nsd/nsd_server.pem"
        control-key-file: "/etc/nsd/nsd_control.key"
        control-cert-file: "/etc/nsd/nsd_control.pem"
```

Also, enable the firewall entries:

```bash
firewall-cmd --zone=public --add-port=8952/tcp
firewall-cmd --runtime-to-permanent
```

On the primary server, change the zone to match the following:

```bash
zone:
    name: example.com
    zonefile: /etc/nsd/example.com.zone
    allow-notify: NS2_IP NOKEY
    provide-xfr: NS2_IP NOKEY
    outgoing-interface: NS1_IP
```

Replace `NS1_IP1` and `NS2_IP2` with the public IP addresses of the primary and secondary nameservers.

On the secondary server, add the zone:

```bash
zone:
    name: fourplex.net
    notify: NS1_IP NOKEY
    request-xfr: NS1_IP NOKEY
    outgoing-interface: NS2_IP
```

Replace `NS1_IP1` and `NS2_IP2` with the public IP addresses of the primary and secondary nameservers.

Restart NSD on both nameservers:

```bash
NS1# systemctl restart --now nsd
```

To download the zone file to the secondary nameserver from the primary one:

```bash
nsd-control notify -s NS2_IP
```

Replace `NS2_IP2` with the public IP addresses of the secondary nameserver.

## Conclusion

Most people use third-party DNS services. However, there are scenarios where self-hosting DNS is desirable. For example, telecommunications, hosting, and social media companies maintain their own DNS entries when third-party DNS services are undesirable.

NSD is one of many open source tools that make hosting DNS possible.
