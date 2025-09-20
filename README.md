# Kluster

This repository contains documentation and deployment scripts for my homelab cluster.

## Hardware

TODO

## Software

As I moved away from a single node cluster, I wanted to add high-availability to the cluster and enable easy multi-node deployment. While Kubernetes would be the defacto choice for such a use-cases, I decided to keep things simple and go with Docker Swarm instead. I've ran K3S before on Raspberry Pi's and while I know my way around the basics of kubernetes, I often ended up spending too much time on getting Kubernetes to work properly.

### Docker Swarm

I started of with a clean install of Ubuntu 24.04 LTS on each node and used snap to install Docker

```
sudo snap install docker
```

From there, easy to enable Docker Swarm, simply initializing a swarm on one of the nodes.

```
sudo docker swarm init
```

In my 3 node setup, all nodes will run as manager. On the first node, you can simply retrieve the join token/command to execute on the other nodes and have them join them join the swarm as managers. You can technically run your cluster with 1 manager and 2 workers, but for proper HA you need at least 3 managers.

```
sudo docker swarm join-token manager
```

After onboarding the nodes to the swarm, you can then check the nodes in your swarm to make sure everything went as expected. (needs to be executed from one of the masters)

```
sudo docker node ls

ID                            HOSTNAME         STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
lsxdv0hrp4rqth63w3aibkba7     kluster-03       Ready     Active         Reachable        28.1.1+1
o9br7xamt36cphvd5tsz5hyi1 *   kluster-01       Ready     Active         Leader           28.1.1+1
j0o5wnjq3z337r5f4a3d1dnp6     kluster-02       Ready     Active         Reachable        28.1.1+1
```

### Keepalived

While we now have a swarm that is capable to keep our workloads running in case of node failure, on a network level we would still be tying everything to IPs of individual nodes. So to create real high-availability, we also need to create a virtual IP that binds our 3 master nodes and makes sure traffic is rerouted in case of a node failure. While you could solve this with an external load balancer, we are keeping things a bit simpler in our homelab and will use keepalived to bind our nodes to a virtual IP.

We start with installing keepalived on each of the nodes

```
sudo apt-get install keepalived
```

The we need to configure keepalived on each of the nodes, which you needs to be created at /etc/keepalived/keepalived.conf. There is a sample config file available but also providing the config I used below with some explanation.

```
sudo nano /etc/keepalived/keepalived.conf
```

I am running 3 nodes with the IP addresses below and want to bind them to a virtual IP at 192.168.100.111. This will obviously be different for your setup, but important for the configuration to have these details.

```
kluster-01      192.168.100.167
kluster-02      192.168.100.155
kluster-03      192.168.100.169
```

The config file requires a few things, on each node you need to define the IP addresses of the other nodes and the interface on which those nodes are reachable. This will likely be different on your machine, in the below response I am only interested in **enp0s31f6** which is the interface on my internal management subnet that connects my 3 nodes. (example on kluster-01, this may be different on each node)


```
ifconfig

enp0s31f6: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.100.167  netmask 255.255.255.0  broadcast 192.168.100.255
        inet6 fe80::6e4b:90ff:fe71:d032  prefixlen 64  scopeid 0x20<link>
        ether 6c:4b:90:71:d0:32  txqueuelen 1000  (Ethernet)
        RX packets 14348  bytes 1575788 (1.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 12507  bytes 1417027 (1.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 17  memory 0xf7200000-f7220000  
```

We will need to pick one node as master, for me that will be kluster-01. Do notice that the config file contains a password for authentication across all nodes, this should be 8 characters and in the config below has the value "password", **make sure you change that.**

```
# kluster-01 @ 192.168.100.167

vrrp_instance VI_1 {
    state MASTER
    interface enp0s31f6
    virtual_router_id 51
    priority 200
    advert_int 1              
    authentication {
        auth_type PASS
        auth_pass password
    }
    unicast_peer {
        192.168.100.155
        192.168.100.169
    }
    virtual_ipaddress {
        192.168.100.111
    }
}
```

And we need to configure the other nodes as backup, below the config for kluster-02 and kluster-03

```
# kluster-02 @ 192.168.100.155

vrrp_instance VI_1 {
    state BACKUP
    interface enp0s31f6
    virtual_router_id 51
    priority 200
    advert_int 1              
    authentication {
        auth_type PASS
        auth_pass password
    }
    unicast_peer {
        192.168.100.167
        192.168.100.169
    }
    virtual_ipaddress {
        192.168.100.111
    }
}

#kluster-03 @ 192.168.100.169

vrrp_instance VI_1 {
    state BACKUP
    interface enp0s31f6
    virtual_router_id 51
    priority 200
    advert_int 1              
    authentication {
        auth_type PASS
        auth_pass password
    }
    unicast_peer {
        192.168.100.167
        192.168.100.155
    }
    virtual_ipaddress {
        192.168.100.111
    }
}
```

We can then enable and start the service on all of the nodes

```
sudo systemctl enable keepalived.service
sudo systemctl start keepalived.service
```

Once up and running you can at all times check the status on each of the nodes.

```
sudo systemctl status keepalived.service
```

Just to test whether everything is working, you can now run a ping against the virtual IP (in my case 192.168.100.111) and reboot the master node. If all goes well, the ping should just continue to run successfully, as in the background keepalived has switch master to one of the other 2 nodes once it detected the master went down.

## Services

TODO
