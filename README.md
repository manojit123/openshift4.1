# Installing  Openshift 4.1 cluster on vSphere step by step<br/>

In our lap setup we build openshift in vsphere6.5 with following nodes.All the nodes have  4 Core 16 GB Mem and 120 GB HDD with thin provision. <br/>

basedomain=gtslabs.ibm.com <br/>
clustername=ocp4 <br/>

As per openshift documentation all the host will need to resolve as nodename.clustername.basedoamin i.e nodename.ocp4.gtslabs.ibm.com

| Nodes | IP Address |
|---|:---|
|bastion     |   192.168.60.254(static IP)|
|bootstrap-0 |   192.168.60.253(dhcp IP)  |
|master-0    |   192.168.60.10(dhcp IP)   |
|master-1    |   192.168.60.11(dhcp IP)   |
|master-2    |   192.168.60.12(dhcp IP)   |
|infnod-0    |   192.168.60.20(dhcp IP)   |
|infnod-1    |   192.168.60.21(dhcp IP)   |
|cptnod-0    |   192.168.60.30(dhcp IP)   |
|cptnod-1    |   192.168.60.31(dhcp IP)   |
|cptnod-2    |   192.168.60.32(dhcp IP)   |

We need to extract the MAC address for all the VM for configuration at later stage
We need to setup some advance property setting which I mentioned at the bottom part of the documenattion 

## Create a bastion host with RHEL 8 with following network configuration. 
Hostname: bastion.ocp4.gtslabs.ibm.com <br/>
IP address: 192.168.60.254 <br/>
Netmask: 255.255.255.0 <br/> 
Gateway: 192.168.60.1 <br/>
DNS server: 192.168.60.1 <br/>
DNS domain: ocp4.gtslabs.ibm.com <br/> 

## Then register the machine with Red Hat Subscription Manager and enable the base repositories for Red Hat Enterprise Linux 8

subscription-manager register --username=<rhn_username> <br/>
subscription-manager attach --pool=<rhn_pool> <br/>
subscription-manager repos --disable='*' \
--enable=rhel-8-for-x86_64-baseos-rpms \
--enable=rhel-8-for-x86_64-appstream-rpms <br/>
dnf update -y <br/>
reboot <br/>

## Setup DNS

dnf install -y bind-chroot <br/>
modify the /etc/named.conf as per DNS config files <br/>
 
create the forward zone file "/var/named/ocp4.gtslabs.ibm.com.zone" <br/>
And the reverse zone file "/var/named/60.168.192.in-addr.arpa.zone" <br/>
Now  start the named-chroot service <br/>

systemctl enable --now named-chroot.service <br/>

And allow the service to be accessed at the firewall <br/>

firewall-cmd --permanent --add-service=dns <br/>
firewall-cmd --reload <br/>

update /etc/resolv.conf <br/>

echo -e "search ocp4.gtslabs.ibm.com\nnameserver 192.168.67.254" > /etc/resolv.conf <br/>


## Setup TFTP Server

dnf install -y tftp-server ipxe-bootimgs <br/>
mkdir -p /var/lib/tftpboot <br/>
ln -s /usr/share/ipxe/undionly.kpxe /var/lib/tftpboot <br/>

systemctl enable --now tftp.service <br/>
firewall-cmd --permanent --add-service=tftp <br/>

firewall-cmd --reload <br/>

## Setup DHCP

We create the /etc/dhcp/dhcpd.conf file with static MAC / IP mapping <br/>

We can now start the dhcpd service <br/>
systemctl enable --now dhcpd.service <br/>

And allow the service to be accessed at the firewall level <br/>
firewall-cmd --permanent --add-service=dhcp <br/>
firewall-cmd --reload <br/>

## Setup Matchbox
The first step is to download and extract the latest release: v0.8.0 <br/>
wget https://github.com/poseidon/matchbox/releases/download/v0.8.0/matchbox-v0.8.0-linux-amd64.tar.gz <br/>

tar -xzf matchbox-v0.8.0-linux-amd64.tar.gz <br/>
cd matchbox-v0.8.0-linux-amd64 <br/>
cp matchbox /usr/local/bin <br/>
cp contrib/systemd/matchbox-local.service /etc/systemd/system/matchbox.service <br/>
cd <br/>

mkdir /etc/matchbox <br/>
mkdir -p /var/lib/matchbox/{assets,groups,ignition,profiles} <br/>
cd /var/lib/matchbox/assets <br/>
RHCOS_BASEURL=https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/ <br/>
wget ${RHCOS_BASEURL}/4.1/latest/rhcos-4.1.0-x86_64-installer-initramfs.img <br/>
wget ${RHCOS_BASEURL}/4.1/latest/rhcos-4.1.0-x86_64-installer-kernel <br/>
wget ${RHCOS_BASEURL}/4.1/latest/rhcos-4.1.0-x86_64-metal-bios.raw.gz <br/>

We also create the different profiles we need to deploy OpenShift 4.1: bootstrap, master, infnod and cptnod. <br/>

/var/lib/matchbox/profiles/bootstrap.json <br/>
/var/lib/matchbox/profiles/master.json <br/>
/var/lib/matchbox/profiles/infnod.json <br/>
/var/lib/matchbox/profiles/cptnod.json <br/>

And we also create the groups files to associate the MAC addresses of the machines to the profiles <br/>

/var/lib/matchbox/groups/bootstrap-0.json <br/>
/var/lib/matchbox/groups/master-0.json <br/>
/var/lib/matchbox/groups/master-1.json <br/>
/var/lib/matchbox/groups/master-2.json <br/>
/var/lib/matchbox/groups/infnod-0.json <br/>
/var/lib/matchbox/groups/infnod-1.json <br/>
/var/lib/matchbox/groups/cptnod-0.json <br/>
/var/lib/matchbox/groups/cptnod-1.json <br/>
/var/lib/matchbox/groups/cptnod-2.json <br/>

The matchbox binary expects to run as matchbox user, so we create it: <br/>

useradd -U -r matchbox <br/>
We can now start the matchbox service <br/>
systemctl daemon-reload <br/>
systemctl enable --now matchbox.service <br/>

And allow the service to be accessed at the firewall level: <br/>
firewall-cmd --permanent --add-port=8080/tcp <br/>
firewall-cmd --reload <br/>

## Setup HAProxy

dnf install -y haproxy <br/>
We now need to configure it via /etc/haproxy/haproxy.cfg <br/>

We also need to enable an SELinux boolean to allow HAProxy to bind to non standard ports for the API frontends <br/>
setsebool -P haproxy_connect_any on <br/>

We can now start the haproxy service <br/>
systemctl enable --now haproxy.service <br/>

And allow the service to be accessed at the firewall level: <br/>
firewall-cmd --permanent --add-service=http <br/>
firewall-cmd --permanent --add-service=https <br/>

firewall-cmd --permanent --add-port=6443/tcp <br/>
firewall-cmd --permanent --add-port=22623/tcp <br/>

firewall-cmd --reload <br/>

## Deploying OpenShift Container Platform 4.1

We are now ready to deploy OpenShift 4.1. There are a few steps to follow, but it's pretty straightforward now that the base services are running.

## Setup SSH keypair

ssh-keygen -t rsa -b 2048 -N ''  -f /root/.ssh/id_rsa

Get Pull secret for vsphere deployment from https://try.openshift.com


## Creating Installation configuration

OCP4_BASEURL=https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest <br/>
curl -s ${OCP4_BASEURL}/openshift-install-linux-4.1.3.tar.gz \
| tar -xzf-  -C /usr/local/bin/ openshift-install <br/>

curl -s ${OCP4_BASEURL}/openshift-client-linux-4.1.3.tar.gz \
| tar -xzf-  -C /usr/local/bin/ oc <br/>

The openshift-install command requires a work directory to store the assets it generates: <br/>

mkdir /root/ocp4.1 <br/>

## Create a file named as  named as "install-config.yaml"
```
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
```
Setup the vCenter password , pullSecret and ssh public key for bastion hosts <br/>

We can now use openshift-install to generate the ignition files <br/>

openshift-install --dir=/root/ocp4.1 create ignition-configs <br/>

We simply need to copy them into the matchbox directory: <br/>
cp /root/ocp4.1/*.ign /var/lib/matchbox/ignition <br/>

## Creating Red Hat Enterprise Linux CoreOS (RHCOS) machines in vSphere
Create a file in /root/ocp4.1 named append-bootstrap.ign <br/>
```
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
```
Convert the master, worker, and secondary bootstrap Ignition config files to Base64 encoding <br/>

base64 -w0 /root/ocp4.1/master.ign > /root/ocp4.1/master.64 <br/>
base64 -w0 /root/ocp4.1/worker.ign > /root/ocp4.1/worker.64 <br/>
base64 -w0 /root/ocp4.1/append-bootstrap.ign > /root/ocp4.1/append-bootstrap.64 <br/>

Right-click the name of your datacenter and create a folder maching with clustername in our case ocp4 <br/>

You need to create VM the folder matching the mac address with the required resources as per the documentation. Or after creatig the VM we should extract the mac address and populate in the various config files above. Use thin provision for better storage utlization. On the Select clone options, select Customize this virtual machin's hardware or you can do that later in the VM property files in advance setting

From the Latency Sensitivity list, select High.
Click Edit Configuration, and on the Configuration Parameters window, click Add Configuration Params. Define the following parameter names and values: <br/>

guestinfo.ignition.config.data: Paste the contents of the base64-encoded Ignition config file for respective node type <br/>

guestinfo.ignition.config.data.encoding: Specify base64. <br/>

disk.EnableUUID: Specify TRUE. <br/>

Reference-style: 
![alt text](https://github.com/manojit123/openshift4.1/blob/master/configfiles/Screenshot%202019-08-21%20at%2010.46.59%20PM.png)

![alt text](https://github.com/manojit123/openshift4.1/blob/master/configfiles/Screenshot%202019-08-21%20at%2010.49.08%20PM.png)

We can now boot the virtual machines in PXE mode, in no specific order. When booting the machines use the ignition configuration to install the cluster. We can monitor the deployment via openshift-install command. It checks that the initial cluster operators are available: <br/>

openshift-install --dir=/root/ocp4.1 wait-for bootstrap-complete --log-level debug <br/>

While the installation proceeds, we can collect the Kubernetes configuration file generated by openshift-install, at the same time as the ignition files.

mkdir /root/.kube <br/>
cp /root/ocp4.1/auth/kubeconfig /root/.kube/config <br/>

Test the deployment by 
```
oc get nodes 
NAME                            STATUS   ROLES    AGE     VERSION
cptnod-0.ocp4.gtslabs.ibm.com   Ready    worker   6d20h   v1.13.4+205da2b4a
cptnod-1.ocp4.gtslabs.ibm.com   Ready    worker   6d20h   v1.13.4+205da2b4a
cptnod-2.ocp4.gtslabs.ibm.com   Ready    worker   6d20h   v1.13.4+205da2b4a
infnod-0.ocp4.gtslabs.ibm.com   Ready    worker   6d20h   v1.13.4+205da2b4a
infnod-1.ocp4.gtslabs.ibm.com   Ready    worker   6d20h   v1.13.4+205da2b4a
master-0.ocp4.gtslabs.ibm.com   Ready    master   6d20h   v1.13.4+205da2b4a
master-1.ocp4.gtslabs.ibm.com   Ready    master   6d20h   v1.13.4+205da2b4a
master-2.ocp4.gtslabs.ibm.com   Ready    master   6d20h   v1.13.4+205da2b4a
```





