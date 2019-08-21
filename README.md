# openshift4.1 based on vSphere documentation. 
In our lap setup we build openshift in vsphere6.5 with following nodes
All the nodes have  4 Core 16 GB Mem and 120 GB HDD with thin provision
bastion        192.168.60.254(static IP)
bootstrap-0    192.168.60.253(dhcp IP)  
master-0       192.168.60.10(dhcp IP)
master-1       192.168.60.11(dhcp IP)
master-2       192.168.60.12(dhcp IP)
infnod-0       192.168.60.20(dhcp IP)
infnod-1       192.168.60.21(dhcp IP)
cptnod-0       192.168.60.30(dhcp IP)
cptnod-1       192.168.60.31(dhcp IP)
cptnod-2       192.168.60.32(dhcp IP)

We need to extract the MAC address for all the VM for configuration at later stage
We need to setup some advance property setting which I mentioned at the bottom part of the documenattion 

# Create a bastion host with RHEL 8 with following network configuration
Hostname: bastion.ocp4.gtslabs.ibm.com
IP address: 192.168.60.254
Netmask: 255.255.255.0
Gateway: 192.168.60.1
DNS server: 192.168.60.1
DNS domain: ocp4.gtslabs.ibm.com

# Then register the machine with Red Hat Subscription Manager and enable the base repositories for Red Hat Enterprise Linux 8

# subscription-manager register --username=<rhn_username>
# subscription-manager attach --pool=<rhn_pool>
# subscription-manager repos --disable='*' \
--enable=rhel-8-for-x86_64-baseos-rpms \
--enable=rhel-8-for-x86_64-appstream-rpms
# dnf update -y
# reboot

# Setup DNS

# dnf install -y bind-chroot
modify the /etc/named.conf as below 
 
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//
acl internal_nets { 192.168.60.0/24; };
options {
	listen-on port 53 { 127.0.0.1; 192.168.60.254; };
	listen-on-v6 port 53 { none; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	secroots-file	"/var/named/data/named.secroots";
	recursing-file	"/var/named/data/named.recursing";
	allow-query     { localhost;internal_nets; };

	/*
	 - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
	 - If you are building a RECURSIVE (caching) DNS server, you need to enable
	   recursion.
	 - If your recursive DNS server has a public IP address, you MUST enable access
	   control to limit queries to your legitimate users. Failing to do so will
	   cause your server to become part of large scale DNS amplification
	   attacks. Implementing BCP38 within your network would greatly
	   reduce such attack surface
	*/
	recursion yes;
        allow-recursion { localhost; internal_nets; };
        forwarders { 8.8.8.8; 192.168.30.50; };

	dnssec-enable yes;
	dnssec-validation no;

	managed-keys-directory "/var/named/dynamic";

	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";

	/* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
	include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
	type hint;
	file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

zone "ocp4.gtslabs.ibm.com" {
	type master;
	file "ocp4.gtslabs.ibm.com.zone";
	allow-query { any; };
	allow-transfer { none; };
	allow-update { none; };
};
zone "60.168.192.in-addr.arpa" {
	type master;
	file "60.168.192.in-addr.arpa.zone";
	allow-query { any; };
	allow-transfer { none; };
	allow-update { none; };
};

# create the forward zone file "/var/named/ocp4.gtslabs.ibm.com.zone"

$TTL 900
@ IN SOA bastion.ocp4.gtslabs.ibm.com. hostmaster.ocp4.gtslabs.ibm.com. (
2019062002 1D 1H 1W 3H
)

  IN NS bastion.ocp4.gtslabs.ibm.com.

bastion      IN  A  192.168.60.254
bootstrap-0  IN  A  192.168.60.253
master-0     IN  A  192.168.60.10
master-1     IN  A  192.168.60.11
master-2     IN  A  192.168.60.12
infnod-0     IN  A  192.168.60.20
infnod-1     IN  A  192.168.60.21
cptnod-0     IN  A  192.168.60.30
cptnod-1     IN  A  192.168.60.31
cptnod-2     IN  A  192.168.60.32
api          IN  A  192.168.60.254
api-int      IN  A  192.168.60.254
apps         IN  A  192.168.60.254
*.apps       IN  A  192.168.60.254
etcd-0       IN  A  192.168.60.10
etcd-1       IN  A  192.168.60.11
etcd-2       IN  A  192.168.60.12
_etcd-server-ssl._tcp  IN  SRV 0 10 2380 etcd-0.ocp4.gtslabs.ibm.com.
_etcd-server-ssl._tcp  IN  SRV 0 10 2380 etcd-1.ocp4.gtslabs.ibm.com.
_etcd-server-ssl._tcp  IN  SRV 0 10 2380 etcd-2.ocp4.gtslabs.ibm.com.


# And the reverse zone file "/var/named/60.168.192.in-addr.arpa.zone"

$TTL 900
@ IN SOA bastion.ocp4.gtslabs.ibm.com. hostmaster.ocp4.gtslabs.ibm.com. (
2019062001 1D 1H 1W 3H
)

  IN NS bastion.ocp4.gtslabs.ibm.com.

10 IN PTR master-0.ocp4.gtslabs.ibm.com.
11 IN PTR master-1.ocp4.gtslabs.ibm.com.
12 IN PTR master-2.ocp4.gtslabs.ibm.com.
20 IN PTR infnod-0.ocp4.gtslabs.ibm.com.
21 IN PTR infnod-1.ocp4.gtslabs.ibm.com.
30 IN PTR cptnod-0.ocp4.gtslabs.ibm.com.
31 IN PTR cptnod-1.ocp4.gtslabs.ibm.com.
32 IN PTR cptnod-1.ocp4.gtslabs.ibm.com.
253 IN PTR bootstrap-0.ocp4.gtslabs.ibm.com.
254 IN PTR bastion.ocp4.gtslabs.ibm.com.

# Now  start the named-chroot service

# systemctl enable --now named-chroot.service

And allow the service to be accessed at the firewall

# firewall-cmd --permanent --add-service=dns
# firewall-cmd --reload

update /etc/resolv.conf

# echo -e "search ocp4.gtslabs.ibm.com\nnameserver 192.168.67.254" > /etc/resolv.conf


# Setup TFTP Server

# dnf install -y tftp-server ipxe-bootimgs
# mkdir -p /var/lib/tftpboot
# ln -s /usr/share/ipxe/undionly.kpxe /var/lib/tftpboot

# systemctl enable --now tftp.service
# firewall-cmd --permanent --add-service=tftp

# firewall-cmd --reload

# Setup DHCP

We create the /etc/dhcp/dhcpd.conf file with static MAC / IP mapping

#
# DHCP Server Configuration file for GTSLabs Openshift Deployment
###### Generic DHCP Configurations
  default-lease-time 900;
  max-lease-time 7200;
  option domain-name "ocp4.gtslabs.ibm.com";
  option domain-search "ocp4.gtslabs.ibm.com";
  option domain-name-servers 192.168.60.254;
  authoritative;
  log-facility local7;
###### Configurations for Subnet 192.168.60.0
  subnet 192.168.60.0 netmask 255.255.255.0
  {
      option routers        192.168.60.1;
      option subnet-mask    255.255.255.0;
      next-server           192.168.60.254;
      if exists user-class and option user-class = "iPXE"
      {
          filename "http://bastion.ocp4.gtslabs.ibm.com:8080/boot.ipxe";
      }
      else
      {
          filename "undionly.kpxe";
      }
  }
###### Individual Host definitions
  host bootstrap-0
  {
      hardware ethernet 00:50:56:96:96:b9;
      fixed-address 192.168.60.253;
      option host-name "bootstrap-0.ocp4.gtslabs.ibm.com";
  }
  host master-0
  {
      hardware ethernet 00:50:56:96:40:1c;
      fixed-address 192.168.60.10;
      option host-name "master-0.ocp4.gtslabs.ibm.com";
  }
  host master-1
  {
      hardware ethernet 00:50:56:96:34:9c;
      fixed-address 192.168.60.11;
      option host-name "master-1.ocp4.gtslabs.ibm.com";
  }
  host master-2
  {
      hardware ethernet 00:50:56:96:f5:56;
      fixed-address 192.168.60.12;
      option host-name "master-2.ocp4.gtslabs.ibm.com";
  }
  host infnod-0
  {
      hardware ethernet 00:50:56:96:65:85;
      fixed-address 192.168.60.20;
      option host-name "infnod-0.ocp4.gtslabs.ibm.com";
  }
  host infnod-1
  {
      hardware ethernet 00:50:56:96:ce:a1;
      fixed-address 192.168.60.21;
      option host-name "infnod-1.ocp4.gtslabs.ibm.com";
  }
host cptnod-0
  {
      hardware ethernet 00:50:56:96:b6:35;
      fixed-address 192.168.60.30;
      option host-name "cptnod-0.ocp4.gtslabs.ibm.com";
  }
  host cptnod-1
  {
      hardware ethernet 00:50:56:96:eb:72;
      fixed-address 192.168.60.31;
      option host-name "cptnod-1.ocp4.gtslabs.ibm.com";
  }
  host cptnod-2
  {
      hardware ethernet 00:50:56:96:c6:4d;
      fixed-address 192.168.60.32;
      option host-name "cptnod-2.ocp4.gtslabs.ibm.com";
  }

We can now start the dhcpd service:
# systemctl enable --now dhcpd.service

And allow the service to be accessed at the firewall level:
# firewall-cmd --permanent --add-service=dhcp
# firewall-cmd --reload

# Setup Matchbox
the first step is to download and extract the latest release: v0.8.0
# wget https://github.com/poseidon/matchbox/releases/download/v0.8.0/matchbox-v0.8.0-linux-amd64.tar.gz

# tar -xzf matchbox-v0.8.0-linux-amd64.tar.gz
# cd matchbox-v0.8.0-linux-amd64
# cp matchbox /usr/local/bin
# cp contrib/systemd/matchbox-local.service /etc/systemd/system/matchbox.service
# cd

# mkdir /etc/matchbox
# mkdir -p /var/lib/matchbox/{assets,groups,ignition,profiles}
# cd /var/lib/matchbox/assets
# RHCOS_BASEURL=https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/
# wget ${RHCOS_BASEURL}/4.1/latest/rhcos-4.1.0-x86_64-installer-initramfs.img
# wget ${RHCOS_BASEURL}/4.1/latest/rhcos-4.1.0-x86_64-installer-kernel
# wget ${RHCOS_BASEURL}/4.1/latest/rhcos-4.1.0-x86_64-metal-bios.raw.gz

We also create the different profiles we need to deploy OpenShift 4.1: bootstrap, master, infnod and cptnod.

/var/lib/matchbox/profiles/bootstrap.json

{
"id": "bootstrap",
"name": "OCP 4 -  Bootstrap",
"ignition_id": "bootstrap.ign",
"boot": {
"kernel": "/assets/rhcos-4.1.0-x86_64-installer-kernel",
"initrd": [
"/assets/rhcos-4.1.0-x86_64-installer-initramfs.img"
],
"args": [
"ip=dhcp",
"rd.neednet=1",
"console=tty0",
"console=ttyS0",
"coreos.inst=yes",
"coreos.inst.install_dev=sda",
"coreos.inst.image_url=http://bastion.ocp4.gtslabs.ibm.com:8080/assets/rhcos-4.1.0-x86_64-metal-bios.raw.gz",
"coreos.inst.ignition_url=http://bastion.ocp4.gtslabs.ibm.com:8080/ignition?mac=${mac:hexhyp}"
]
}
}

/var/lib/matchbox/profiles/master.json

{
"id": "master",
"name": "OCP 4 -  Master",
"ignition_id": "master.ign",
"boot": {
"kernel": "/assets/rhcos-4.1.0-x86_64-installer-kernel",
"initrd": [
"/assets/rhcos-4.1.0-x86_64-installer-initramfs.img"
],
"args": [
"ip=dhcp",
"rd.neednet=1",
"console=tty0",
"console=ttyS0",
"coreos.inst=yes",
"coreos.inst.install_dev=sda",
"coreos.inst.image_url=http://bastion.ocp4.gtslabs.ibm.com:8080/assets/rhcos-4.1.0-x86_64-metal-bios.raw.gz",
"coreos.inst.ignition_url=http://bastion.ocp4.gtslabs.ibm.com:8080/ignition?mac=${mac:hexhyp}"
]
}
}

/var/lib/matchbox/profiles/infnod.json

{
"id": "infnod",
"name": "OCP 4 - Infrastructure Node",
"ignition_id": "worker.ign",
"boot": {
"kernel": "/assets/rhcos-4.1.0-x86_64-installer-kernel",
"initrd": [
"/assets/rhcos-4.1.0-x86_64-installer-initramfs.img"
],
"args": [
"ip=dhcp",
"rd.neednet=1",
"console=tty0",
"console=ttyS0",
"coreos.inst=yes",
"coreos.inst.install_dev=sda",
"coreos.inst.image_url=http://bastion.ocp4.gtslabs.ibm.com:8080/assets/rhcos-4.1.0-x86_64-metal-bios.raw.gz",
"coreos.inst.ignition_url=http://bastion.ocp4.gtslabs.ibm.com:8080/ignition?mac=${mac:hexhyp}"
]
}
}

/var/lib/matchbox/profiles/cptnod.json

{
"id": "cptnod",
"name": "OCP 4 - Compute Node",
"ignition_id": "worker.ign",
"boot": {
"kernel": "/assets/rhcos-4.1.0-x86_64-installer-kernel",
"initrd": [
"/assets/rhcos-4.1.0-x86_64-installer-initramfs.img"
],
"args": [
"ip=dhcp",
"rd.neednet=1",
"console=tty0",
"console=ttyS0",
"coreos.inst=yes",
"coreos.inst.install_dev=sda",
"coreos.inst.image_url=http://bastion.ocp4.gtslabs.ibm.com:8080/assets/rhcos-4.1.0-x86_64-metal-bios.raw.gz",
"coreos.inst.ignition_url=http://bastion.ocp4.gtslabs.ibm.com:8080/ignition?mac=${mac:hexhyp}"
]
}
}

And we also create the groups files to associate the MAC addresses of the machines to the profiles

/var/lib/matchbox/groups/bootstrap-0.json

{
"id": "bootstrap-0",
"name": "OCP 4 - Bootstrap server",
"profile": "bootstrap",
"selector": {
"mac": "00:50:56:96:96:b9"
}
}

/var/lib/matchbox/groups/master-0.json

{
"id": "master-0",
"name": "OCP 4 - Master 1",
"profile": "master",
"selector": {
"mac": "00:50:56:96:40:1c"
}
}

/var/lib/matchbox/groups/master-1.json

{
"id": "master-1",
"name": "OCP 4 - Master 2",
"profile": "master",
"selector": {
"mac": "00:50:56:96:34:9c"
}
}

/var/lib/matchbox/groups/master-2.json

{
"id": "master-2",
"name": "OCP 4 - Master 3",
"profile": "master",
"selector": {
"mac": "00:50:56:96:f5:56"
}
}

/var/lib/matchbox/groups/infnod-0.json

{
"id": "infnod-0",
"name": "OCP 4 - Infrastructure Node #1",
"profile": "infnod",
"selector": {
"mac": "00:50:56:96:65:85"
}
}

/var/lib/matchbox/groups/infnod-1.json

{
"id": "infnod-1",
"name": "OCP 4 - Infrastructure Node #2",
"profile": "infnod",
"selector": {
"mac": "00:50:56:96:ce:a1"
}
}

/var/lib/matchbox/groups/cptnod-0.json

{
"id": "cptnod-0",
"name": "OCP 4 - Compute node #1",
"profile": "cptnod",
"selector": {
"mac": "00:50:56:96:b6:35"
}
}

/var/lib/matchbox/groups/cptnod-1.json

{
"id": "cptnod-1",
"name": "OCP 4 - Compute node #2",
"profile": "cptnod",
"selector": {
"mac": "00:50:56:96:eb:72"
}
}

/var/lib/matchbox/groups/cptnod-2.json

{
"id": "cptnod-2",
"name": "OCP 4 - Compute node #3",
"profile": "cptnod",
"selector": {
"mac": "00:50:56:96:c6:4d"
}
}

The matchbox binary expects to run as matchbox user, so we create it:

# useradd -U -r matchbox
We can now start the matchbox service
# systemctl daemon-reload
# systemctl enable --now matchbox.service

And allow the service to be accessed at the firewall level:
# firewall-cmd --permanent --add-port=8080/tcp
# firewall-cmd --reload

# Setup HAProxy

# dnf install -y haproxy
We now need to configure it via /etc/haproxy/haproxy.cfg


#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   https://www.haproxy.org/download/1.8/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
#frontend main
#    bind *:5000
#    acl url_static       path_beg       -i /static /images /javascript /stylesheets
#    acl url_static       path_end       -i .jpg .gif .png .css .js

#    use_backend static          if url_static
#    default_backend             app
frontend ocp4-kubernetes-api-server
    mode tcp
    option tcplog
    bind api.ocp4.gtslabs.ibm.com:6443
    bind api-int.ocp4.gtslabs.ibm.com:6443
    default_backend ocp4-kubernetes-api-server

frontend ocp4-machine-config-server
    mode tcp
    option tcplog
    bind api.ocp4.gtslabs.ibm.com:22623
    bind api-int.ocp4.gtslabs.ibm.com:22623
    default_backend ocp4-machine-config-server

frontend ocp4-router-http
    mode tcp
    option tcplog
    bind apps.ocp4.gtslabs.ibm.com:80
    default_backend ocp4-router-http

frontend ocp4-router-https
    mode tcp
    option tcplog
    bind apps.ocp4.gtslabs.ibm.com:443
    default_backend ocp4-router-https


#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
#backend static
#    balance     roundrobin
#    server      static 127.0.0.1:4331 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
#backend app
#    balance     roundrobin
#    server  app1 127.0.0.1:5001 check
#    server  app2 127.0.0.1:5002 check
#    server  app3 127.0.0.1:5003 check
#    server  app4 127.0.0.1:5004 check


backend ocp4-kubernetes-api-server
mode tcp
balance source
server bootstrap-0 bootstrap-0.ocp4.gtslabs.ibm.com:6443 check
server master-0 master-0.ocp4.gtslabs.ibm.com:6443 check
server master-1 master-1.ocp4.gtslabs.ibm.com:6443 check
server master-2 master-2.ocp4.gtslabs.ibm.com:6443 check

backend ocp4-machine-config-server
mode tcp
balance source
server bootstrap-0 bootstrap-0.ocp4.gtslabs.ibm.com:22623 check
server master-0 master-0.ocp4.gtslabs.ibm.com:22623 check
server master-1 master-1.ocp4.gtslabs.ibm.com:22623 check
server master-2 master-2.ocp4.gtslabs.ibm.com:22623 check

backend ocp4-router-http
mode tcp
server infnod-0 infnod-0.ocp4.gtslabs.ibm.com:80 check
server infnod-1 infnod-1.ocp4.gtslabs.ibm.com:80 check

backend ocp4-router-https
mode tcp
server infnod-0 infnod-0.ocp4.gtslabs.ibm.com:443 check
server infnod-1 infnod-1.ocp4.gtslabs.ibm.com:443 check


We also need to enable an SELinux boolean to allow HAProxy to bind to non standard ports for the API frontends
# setsebool -P haproxy_connect_any on

We can now start the haproxy service
# systemctl enable --now haproxy.service

And allow the service to be accessed at the firewall level:
# firewall-cmd --permanent --add-service=http
# firewall-cmd --permanent --add-service=https

# firewall-cmd --permanent --add-port=6443/tcp
# firewall-cmd --permanent --add-port=22623/tcp

# firewall-cmd --reload

# Deploying OpenShift Container Platform 4.1

We are now ready to deploy OpenShift 4.1. There are a few steps to follow, but it's pretty straightforward now that the base services are running.

# Setup SSH keypair

# ssh-keygen -t rsa -b 2048 -N ''  -f /root/.ssh/id_rsa

# Get Pull secret for vsphere deployment from https://try.openshift.com


# Creating Installation configuration

# OCP4_BASEURL=https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest
# curl -s ${OCP4_BASEURL}/openshift-install-linux-4.1.3.tar.gz \
| tar -xzf-  -C /usr/local/bin/ openshift-install

# curl -s ${OCP4_BASEURL}/openshift-client-linux-4.1.3.tar.gz \
| tar -xzf-  -C /usr/local/bin/ oc

The openshift-install command requires a work directory to store the assets it generates:

# mkdir /root/ocp4.1

# Create a file named as  named as "install-config.yaml"

apiVersion: v1
baseDomain: gtslabs.ibm.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
platform:
  vsphere:
    vcenter: 10.94.76.196
    username: "administrator@vsphere.local"
    password: "XXXXXXX"
    datacenter: "datacenter1"
    defaultDatastore: "vsanDatastore"
pullSecret: ''
sshKey: ''

# Setup the vCenter password , pullSecret and ssh public key for bastion hosts

We can now use openshift-install to generate the ignition files

# openshift-install --dir=/root/ocp4.1 create ignition-configs

We simply need to copy them into the matchbox directory:
# cp /root/ocp4.1/*.ign /var/lib/matchbox/ignition

# Creating Red Hat Enterprise Linux CoreOS (RHCOS) machines in vSphere
Create a file in /root/ocp4.1 named append-bootstrap.ign

{
  "ignition": {
    "config": {
      "append": [
        {
          "source": "http://bastion.ocp4.gtslabs.ibm.com:8080/ignition/bootstrap.ign",
          "verification": {}
        }
      ]
    },
    "timeouts": {},
    "version": "2.1.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {}
}

Convert the master, worker, and secondary bootstrap Ignition config files to Base64 encoding

base64 -w0 /root/ocp4.1/master.ign > /root/ocp4.1/master.64
base64 -w0 /root/ocp4.1/worker.ign > /root/ocp4.1/worker.64
base64 -w0 /root/ocp4.1/append-bootstrap.ign > /root/ocp4.1/append-bootstrap.64

Right-click the name of your datacenter and create a folder maching with clustername in our case ocp4

You need to create VM the folder matching the mac address with the required resources as per the documentation. Or after creatig the VM we should extract the mac address and populate in the various config files above. Use thin provision for better storage utlization. On the Select clone options, select Customize this virtual machin's hardware or you can do that later in the VM property files in advance setting

From the Latency Sensitivity list, select High.
Click Edit Configuration, and on the Configuration Parameters window, click Add Configuration Params. Define the following parameter names and values:
guestinfo.ignition.config.data: Paste the contents of the base64-encoded Ignition config file for respective node type

guestinfo.ignition.config.data.encoding: Specify base64.

disk.EnableUUID: Specify TRUE.


We can now boot the virtual machines in PXE mode, in no specific order. When booting the machines use the ignition configuration to install the cluster. We can monitor the deployment via openshift-install command. It checks that the initial cluster operators are available:

# openshift-install --dir=/root/ocp4.1 wait-for bootstrap-complete --log-level debug

While the installation proceeds, we can collect the Kubernetes configuration file generated by openshift-install, at the same time as the ignition files.

# mkdir /root/.kube
# cp /root/ocp4.1/auth/kubeconfig /root/.kube/config






