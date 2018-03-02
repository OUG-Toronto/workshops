# Neutron Dynamic Routing Workshop

## Overview

OpenStack Neutron is capable of peering with external routers running Border Gateway Protocol (BGP). This is accomplished via [Neutron Dynamic Routing](https://docs.openstack.org/neutron-dynamic-routing/latest/) (NDR).

Once dynamic routing is configured, certain address scopes are allowed to be advertised by a Neutron agent. Once subnets are created in a particular address scope, they will be shared via BGP and become available externally. This is a form of network automation and helps to integrate an OpenStack cloud into an organizations network.

## Abstract

OpenStack is often considered complex to deploy. However, surprisingly, often the most difficult part of deploying OpenStack is integrating it with an organizations internal network. In this workshop we will show how to use Neutron Dynamic Routing to more easily integrate OpenStack with an internal network. This is accomplished by configuring Neutron to peer with an organizations routers to announce routes. With this peering in place, networks and first hops in the OpenStack cloud will *automatically* become available to a larger internal network.

## What We Will Do

In this lab we will:

* Create a DevStack instance that has the `neutron-dynamic-routing` plugin enabled
* Setup a Quagga instance on the same node as DevStack (though running in a separate Linux network name space)
* Configure OpenStack Neutron to advertise routes to the Quagga instance via BGP
* Observe the advertised routes being accepted by the Quagga instance.

## Requirements

The requirements are fairly low for the lab, especially if you have some experience using the command line (CLI).

* Laptop with WIFI capability
* Ability to use SSH (eg. Putty on Windows)
* Some CLI experience--ability to cut and paste commands from this document into a SSH session

## How to Use This Document

For the most part, each of the `code` sections is meant to be cut and paste into a terminal session on the DevStack instance you will be provided as part of the lab.

It's not necessary to type all the commands in...unless you want to. :)

## Pre-Work

Some pre-work has already been done to speed up this lab, as DevStack can take 30-50 minutes to install. It's being documented here so that the lab is repeatable, but people participating in the lab don't need to setup DevStack.

### Install DevStack

Configure a stack user.

```
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo passwd stack
# add a known password
```

OPTIONAL: Setup ssh access via a public key.

```
sudo mkdir /opt/stack/.ssh
ssh-add -L > authorized_keys
sudo mv authorized_keys /opt/stack/.ssh/
sudo chown stack:stack /opt/stack/.ssh/authorized_keys
```

Log out of the instance and re-login as the `stack` user.

```
ssh stack@${DEVSTACK_IP}
```

Clone DevStack and checkout a particular commit sha.

*NOTE: We are using a specific checkout that is known to work. VERY IMPORTANT to checkout that particular sha.*

```
git clone https://git.openstack.org/openstack-dev/devstack
cd devstack
git checkout a30f89b
```

Obtain a the `local.conf` file we will use for this lab.

```
wget https://raw.githubusercontent.com/OUG-Toronto/workshops/neutron-dynamic-routing-and-quagqa/local.conf
```

From a screen session, start the DevStack install.

*NOTE: We use screen just in case the ssh connection fails for some reason.*

```
screen -R install
./stack.sh
```

Wait for about 30 minutes as the install completes.

### Profile

Add the below to `~/.profile`.

```
export OS_PROJECT_DOMAIN_NAME="default"
export OS_IDENTITY_API_VERSION=3
alias os=openstack
. /opt/stack/devstack/accrc/admin/admin
```

Source it.

```
. ~/.profile
```

## Start the Workshop

This is where we will start the lab/workshop from.

## Quagga

Quagga is a collection of networking daemons, including `bgpd`, which we will be using for this lab.

OpenStack has some documentation in setting up [Quagga](https://docs.openstack.org/neutron-dynamic-routing/ocata/others/testing.html) with NDR, but we will take it a bit further with a network namespace.

### Install

Install quagga.

```
sudo apt-get install quagga quagga-doc
```

### Configure

Install a couple of configuration files.

Change directories to `/etc/quagga`.

```
cd /etc/quagga
```

Install bgpd.conf.

*NOTE: Note the passive setting in the configuration file. The NDR agent is not listening on any ports, so Quagga, or any external router, cannot connect to it. Thus the need for the passive setting.*

```
sudo wget https://raw.githubusercontent.com/OUG-Toronto/workshops/neutron-dynamic-routing-and-quagqa/bgpd.conf
```

Install zebra.conf:

```
sudo wget https://raw.githubusercontent.com/OUG-Toronto/workshops/neutron-dynamic-routing-and-quagqa/zebra.conf
```

### Quagga Network Namespace

We will be using Linux network namespaces to simulate an external router. We only have one host in this lab, so we use network namespaces to make it appear as though the quagga instance is running on a separate host. (Aside: Network namspaces are a powerful feature of Linux, one which has allowed Linux based containers to flourish. They are used heavily in Neutron.)

First create the namespace and add interfaces into it.

```
sudo ip netns create quagga-router
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth1 netns quagga-router
```

Next, we configure interfaces inside and outside of the namespace.

```
sudo ip netns exec quagga-router ip add add 127.0.0.1 lo
sudo ip netns exec quagga-router ip link set dev lo up
sudo ip link set veth0 up
sudo ip add add 10.55.0.1/24 dev veth0
sudo ip netns exec quagga-router ip link set veth1 up
sudo ip netns exec quagga-router ip add add 10.55.0.2/24 dev veth1
sudo ip netns exec quagga-router ip route add default via 10.55.0.1
```

### Start Quagga

Stop quagga if it is running.

*NOTE: We will start quagga by hand, and won't be using systemctl.*

```
sudo systemctl stop quagga
sudo pkill -f quagqa #just in case... :)
```

Access the network namespace. The below will start a bash shell in the quagga-router namespace.

```
sudo ip netns exec quagga-router bash
```

The below commands assume that they are being run from that shell.

In the `quagga-router` network namespace, start `bgpd` and `zebra`.

```
/usr/lib/quagga/zebra --daemon -A 10.55.0.2
/usr/lib/quagga/bgpd --daemon -A 10.55.0.2
```

Exit the namespace.

```
exit
```

### Access Quagga

We should now be able to telnet into the zebra and bgpd instances. Below we will login to the `bgpd` instance.

*NOTE: Remember the password to login is `zebra`.*

Telnet to port 2605 to access `bgpd`.

```
telnet 10.55.0.2 2605
```

Once the password is entered, we should see something like the below.

```
$ telnet 10.55.0.2 2605
Trying 10.55.0.2...
Connected to 10.55.0.2.
Escape character is '^]'.

Hello, this is Quagga (version 0.99.24.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.


User Access Verification

Password:
bgp-devstack-02> show ip bgp
No BGP network exists
bgp-devstack-02>
```

Note that no BGP entries exist.

Exit from that telnet session.

#### Optional: Example of Listening Ports

As an example, in the namespace, show what ports are listening. Notice only a few ports are listening.

```
quagga-router-namespace# netstat -nplt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 10.55.0.2:2601          0.0.0.0:*               LISTEN      28141/zebra     
tcp        0      0 10.55.0.2:2605          0.0.0.0:*               LISTEN      26192/bgpd      
tcp        0      0 0.0.0.0:179             0.0.0.0:*               LISTEN      26192/bgpd      
tcp6       0      0 :::179                  :::*                    LISTEN      26192/bgpd
```

Outside of the namespace, many port ports are listening. These are all the OpenStack APIs and such.

```
$ netstat -nplt
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:8775            0.0.0.0:*               LISTEN      960/nova-api-metauW
tcp        0      0 0.0.0.0:9191            0.0.0.0:*               LISTEN      19884/python    
tcp        0      0 127.0.0.1:60999         0.0.0.0:*               LISTEN      20670/glance-apiuWS
SNIP!
```

## Configure Neutron Dynamic Routing

Download the configuration file into `/etc/neutron`.

```
cd /etc/neutron
sudo wget https://raw.githubusercontent.com/OUG-Toronto/workshops/neutron-dynamic-routing-and-quagqa/bgp_dragent.ini
```

Restart the NDR agent.

```
sudo systemctl restart devstack@q-dr-agent.service
```

NDR is now configured.

## Configure OpenStack Networking

We need to make some changes to default DevStack.

First, make sure `.profile` has been sourced.

```
. ~/.profile
```

Remove default DevStack networks.

```
openstack router remove subnet router1 private-subnet
openstack subnet delete private-subnet
openstack subnet pool delete shared-default-subnetpool-v4
openstack router unset --external-gateway router1
openstack subnet delete public-subnet
```

Configure new scopes and subnets.

```
neutron address-scope-create --shared public 4
neutron subnetpool-create --pool-prefix 172.24.4.0/24 \
  --address-scope public provider
neutron subnetpool-create --pool-prefix 10.0.0.0/16 \
  --address-scope public --shared selfservice
neutron subnet-create --name provider --subnetpool provider --prefixlen 24 public
neutron subnet-create --name selfservice --subnetpool selfservice  --prefixlen 24 private
```

We should now have a few subnets, including `10.0.0.0/24`.

```
$ os subnet list -c Name -c Subnet
+---------------------+--------------------+
| Name                | Subnet             |
+---------------------+--------------------+
| selfservice         | 10.0.0.0/24        |
| provider            | 172.24.4.0/24      |
| ipv6-private-subnet | fdc1:7f2:e3ba::/64 |
| ipv6-public-subnet  | 2001:db8::/64      |
+---------------------+--------------------+
```

Create a new router.

```
openstack router set --external-gateway public router1
openstack router add subnet router1 selfservice
```

Create a BGP speaker.

```
neutron bgp-speaker-create --ip-version 4 \
 --local-as 200 bgp-speaker
neutron bgp-speaker-network-add bgp-speaker public
```

Finally, peer that BGP speaker with the Quagga instance.

```
neutron bgp-peer-create --peer-ip 10.55.0.2 \
  --remote-as 100 bgp-100-200
neutron bgp-speaker-peer-add bgp-speaker bgp-100-200
```

Peering should now be setup between Neutron and Quagga.

## Validate

We will login to the `bgpd` instance and show what BGP routes it knows.

Note that the "Next Hop" will have the "public" IP of the Neutron router we created above, and the network announced will the the subnet we created.

```
telnet 10.55.0.2 2605
# password: zebra
bgp-devstack-02> show ip bgp
BGP table version is 0, local router ID is 10.55.0.2
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.0.0.0/24      172.24.4.8                             0 200 i

Total number of prefixes 1
bgp-devstack-02>
```

Run the below to validate the "Next Hop" IP.

```
os router show router1
```

Look for the `external_gateway_info` line.

## Conclusion

We have now setup quagga in a namespace and setup Neutron to peer with it.

## Extra Credit

### Add Another Subnet

Create another subnet.

```
neutron subnet-create --name selfservice2 --subnetpool selfservice  --prefixlen 24 private
```

Add it to `router1`.

```
openstack router add subnet router1 selfservice2
```

The new subnet should be published to the Quagga router.

Telnet into the Quagga `bgpd` instance to validate.

```
bgp-devstack-02> show ip bgp
BGP table version is 0, local router ID is 10.55.0.2
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.0.0.0/24      172.24.4.8                             0 200 i
*> 10.0.1.0/24      172.24.4.8                             0 200 i

Total number of prefixes 2
bgp-devstack-02>
```

As can be seen above, the second subnet is now known by the Quagga router, and has a next hop of the external interface of `router1`.

## Related Work

### Using Juniper vSRX Instead of Quagga

There is also a workshop available on using a [Juniper vSRX](https://github.com/idx-labs/openstack-network-slicing/blob/master/lab0/README.md) instead of Quagga.

## TODO

* Have zebra inject routes into the namespace and actually be able to ping the internal IP of the Neutron router
* Use openstack-client commands
