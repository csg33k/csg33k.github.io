+++ 
date = "2020-08-26"
title = "Docker IPV6 Guide"
#slug = "do" 
tags = ["docker", "linux", "opensource", "kubernetes"]
categories = ["blog"]
authors = ["csgeek"]
+++

I spent a good bit of time trying to figure this out, so I thought I'd record this for posterity's sake and others might benefit.

**Assumptions**:

 1. You are somewhat familiar with [docker](www.docker.com)
 2. You have some exposure with [docker-compose](https://docs.docker.com/compose/)
 3. You have at least a basic understanding of networking fundamentals.  I'm not a networking expert but you should have an understanding of TCP, the existence of IPV4 and IPV6 and how they differ.
 4. Some basic understanding of firewalls, iptables would be good.  

---

First of all as a developer I find docker to be a glorious thing.  Though it's not limited to just docker, containers in general have made software development so much easier.  It allows a user to run a simple command and have a database, cache, application pretty much any component at your disposal without having the need to be a system administrator and understand the intricacies of configurations in order to get all of these services up and running.

That being said, in production there are many MANY questions that need to be addressed before going from a dev's use case of docker to a production environment but still it's really a great tool.

Now, back to the problem at hand, networking.

# Networking 

## IPV4

Docker was initially built with IPV4 in mind and IPV6 feels like an afterthought.  For IPV4, if you want to expose a service, linked multiple docker containers together, do load balancing everything seems to 'Just work'.  Everything is taken care of for you under the hood and you really don't need to do much.


This is an example configuration pulled from the [wordpress](https://hub.docker.com/_/wordpress) docker hug.


```yaml

version: '3.1'

services:

  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:

```

Bringing up different services and wiring them together is a breeze.  All the iptables rules are taken care of and things just work.  I can connect to http://localhost:8080 and I will see the wordpress getting started guide that walks me through the steps to configure my application.


Now, IF you are operating in an IPV6 only world.  Let's see what this nightmare is about.

### IPV6 Networking

#### Step 1, enable in the Daemon

You need to edit /etc/docker/daemon.json and add these options.

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00::/80"
}
```

The fixed-cidr6 doesn't matter too much but you shouldn't clash with other networks already defined.   The official documentation can be found [here](https://docs.docker.com/config/daemon/ipv6/) and I wouldn't stray too far from it unless you know what you're doing.

Now, the official instructions say all you need to do is restart the service, but I've found you may need to restart your computer. 

At this point, the network for docker0 default network has been updated to use IPv6.  This does NOT mean that every docker network is IPv6 enabled.

### Step 2. FIrewall rules

For some reason this is not automatic and needs to be applied manually.

```sh
ip6tables -t nat -A POSTROUTING -s fd00::/80 ! -o docker0 -j MASQUERADE

```


At this point basic functionality works fine.  So to test this out we can run this:

```sh
docker run --rm -t busybox ping6 -c 4 google.com
```

Which results in:

```sh
PING google.com (2607:f8b0:400a:808::200e): 56 data bytes
64 bytes from 2607:f8b0:400a:808::200e: seq=0 ttl=119 time=17.133 ms
64 bytes from 2607:f8b0:400a:808::200e: seq=1 ttl=119 time=17.119 ms
64 bytes from 2607:f8b0:400a:808::200e: seq=2 ttl=119 time=17.281 ms
64 bytes from 2607:f8b0:400a:808::200e: seq=3 ttl=119 time=17.430 ms

--- google.com ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 17.119/17.240/17.430 ms
```

### Docker Compose + IPV6

At this point docker has support for IPV6, but since docker-compose generally creates a new network for each docker-compose.yml definition it won't work as expected.


The big issue with docker-compose is that it seems IPV6 is not supported for any schema version higher than 2.1 (Current version is 3.7).  


Here is an equivalent version using IPV6 and docker-compose

```yaml
version: "2.1"
services:
  busy:
    image: busybox
    command: ping6 -c 4 google.com
    networks:
      - app_net


networks:
  app_net:
    enable_ipv6: true
    driver: bridge
    driver_opts:
      com.docker.network.enable_ipv6: "true"
    ipam:
      driver: default
      config:
       - subnet: 2001:3984:3989::/64
         gateway: 2001:3984:3989::1
```

Of course now we also need to update the iptable rules:

```
sudo ip6tables -t nat -A POSTROUTING -s  2001:3984:3989::/64 ! -o docker0 -j MASQUERADE
```

ANDDDDDD.. once you're done you'll have to remove them.

```
sudo ip6tables -t nat -D POSTROUTING -s  2001:3984:3989::/64 ! -o docker0 -j MASQUERADE

```

A few notes and warnings

1. The network cannot clash with the existing docker0 network you gave the daemon.
2. Version of the schema must not be higher than 2.1.  Docker-compose Binary can be the latest version, but the schema has to be 2.1.  You can read more info on the issue [here](https://github.com/docker/compose/issues/3988).

### NAT Issues

Okay, so the way IPV4 works is that all traffic is masked so that if  the host's IP is 10.5.5.23 (for example) and we have 5 different containers all with their own individual addresses, let's say 1.1.1.2-1.1.1.7 respectively.   Any traffic that goes out is [NAT](https://en.wikipedia.org/wiki/Network_address_translation)ed  and as far as the outside world and internet network is concerned the request came from 10.5.5.23. 

For IPV6, The request is NOT NATed so all request are coming from the IPV6 docker address and if the firewall doesn't explicitly allow that address then the request will be dropped.  

This is especially troublesome for internal networks where we want to ensure we don't allow external traffic in.  

At this point we have 2 options.


1. Allocate docker a range of IPv6 and update all firewall rules to allow traffic in from those IPs to the appropriate resources.

This I find to be particularly annoying because docker in general is intended to be a transient resource.  You create/destroy resources as needed to allow you to scale up and down.  Having a permanent allocation of address in your network and having to subdivide that range across your multiple compose files can become a logistic nightmare.


2. IPV6 NAT-ing

In general this practice is frowned upon.  I'm not a network expert so I will let you all make your decision on this but from a Developer perspective this solution is great.  Though I do encourage you to read up more about this before just copy/pasting this in.


I ran across this github project called [docker-ipv6nat](https://github.com/robbertkl/docker-ipv6nat) which solved all of my problems.  Someone should really buy the author a beer.    


Let's convert our busybox ping project to use this then I'll explain everything it does.  


```
version: "2.1"
services:
  busy:
    image: busybox
    command: ping6 -c 4 google.com
    networks:
      - app_net


  ipv6:
    image: robbertkl/ipv6nat
    restart: unless-stopped
    network_mode: "host"
    privileged: true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /lib/modules:/lib/modules:ro

networks:
  beef:
    enable_ipv6: true
    driver: bridge
    driver_opts:
      com.docker.network.enable_ipv6: "true"
    ipam:
      driver: default
      config:
       - subnet: 2001:3984:3989::/64
```

This approach no longer requires you to create custom iptables rules AND it NATs ipv6 so the request will be accepted without having to update all firewall rules with your favorite IPv6 network.  The project also cleans up after itself and removes any routing rules it creates.

This unblocked me personally and allowed me to have a docker container work as it used under IPV4.

Now, is this ideal? It depends.

From a networking perspective, it's an anti-pattern because we're doing doing things we shouldn't be with IPV6 
From a dev perspective it's perfect.

The alternative though is to make things work as desired from a networking perspective and create an administration hell which makes docker have to be treated as if it was bare metal.


Final thoughts.

For the love of god docker needs to update its IPV6 policy and the way their application supports IPV6.  This is so overly complicated for no reason.

I would love to hear everyone's thoughts on this and if there are better ways of doing this, but for now for my own personal use I will use IPV6 NAT whenever and wherever i need IPV6 support.1G