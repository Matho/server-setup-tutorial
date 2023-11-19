# How to setup Linux infrastructure for hosting Ruby on Rails websites via Docker Swarm, GlusterFS, Traefik, Portainer and PostgreSQL in High Availability with Zabbix monitoring
Read time: 40 minutes

## Introduction and VPS provider selection
This is third edition of my tutorial about setuping system infrastructure via Docker. This time we will install infrastructure on amd64 arch VPS server, in VPS provider from Germany, called  [Contabo](https://contabo.com/en/). On their website they says, that they are "the best priced VPS on the Planet". And yes, their prices are very good. My infrastructure is not business-critical, so I dont expect much from such a cheap provider. If you want to follow this tutorial but use another VPS provider like [DigitalOcean](https://www.digitalocean.com/), feel free to do it.

I have selected product `VPS S SSD` with 4 vCPU cores, 8GB ram, 50 GB NVMe / 200 GB SSD storage and 1 snapshot for 5.40 EUR/month. Because I have selected monthly payment, I needed to pay initial setup fee for another 5.40 EUR. Because I want to setup HA, I have ordered another one VPS, with the same specification. If you want to communicate via private network between the servers, I recommend to pay extra 2.75 EUR per server. So basically, the monthly fee (without setup fee) for one server is in total 8.15 EUR. Which is great price!

`NOTE on 19.11.2023:` At this date, Contabo has long outage in the one of Germany datacentre, where I have this two vps. It completely broke my private networking for half a day, so I do not recommend this provider for hosting business critical websites. 

## 1 Initial server preparation
The process of buying VPS from Contabo is fully automate. You can pay via card and save the card for next payments. The new VPS was prepared in 5 minutes. I strongly recommend to activate private networking at the beginning of server setuping. The process of activating private network will cause the data loss on the VPS!

At the beginning I have setup DNS A record for both the VPS to `vps1.matho.sk`  and `vps2.matho.sk` . Feel free to change it according your needs, but we will use this domains in the following tutorial. If you want to read more about private networking in Contabo, check the following [article](https://contabo.com/blog/introducing-private-networking-isolated-server-environment/).

In this tutorial we will use Ubuntu 22.04.3 LTS. It is the latest available LTS version at the time of preparing this tutorial (Oct 2023).

## 2 Initial configuration
I have bought 2 VPS, but it is recommended to use at least 3 VPS for Docker cluster. In this tutorial we will use only 2. But it should not be too hard to migrate to 3, if you need.

### 2.1 Change hostnames
I prefer to see my own hostnames, when I login to VPS server via ssh. I will change the hostname of first VPS to `ubuntu1` and the second to `ubuntu2`. This will help me to better understand to which VPS I'm currently logged in.

To list current hostname, execute:  
`$ hostnamectl`

To change it to name `ubuntu1`, execute:  
`$ sudo hostnamectl set-hostname ubuntu1`

Verify new changes:  
`$ hostnamectl`

To see the changes, simply logout and login via ssh.

### 2.2 SSH settings
It is recommended to use ssh keys instead of password. With this way, you will not need to enter password each time you will login to server.

On both nodes create folder for ssh keys, if do not exists:  
`$ mkdir ~/.ssh`

Verify, if this file is created, if not, create  
`$ vim ~/.ssh/authorized_keys`

Copy paste here your public ssh key (run from your notebook):  
`$ cat ~/.ssh/id_rsa.pub`

and insert to server  
`$ vim ~/.ssh/authorized_keys`

Next, it is not safe to use default port 22, so we will change it to 7777 (use the value you want). Open  
`$ sudo vim /etc/ssh/sshd_config`

Disable login via password - set this option to no  
`PasswordAuthentication no`

and change the port to 7777, at the begining of file  
`Port 7777`

You can restart your ssh service via  
`$ sudo systemctl restart ssh.service`

It is recommended not to logout from your active shell. Instead, open new terminal window. If you broke the things, this will ensure, you are able to login to server and you will not end up locked.

Open new terminal and try to login:  
`$ ssh root@vps1.matho.sk -p 7777`

You should be logged in correctly, now. Change the setting for the second VPS, also.

### 2.3 Firewall
I will use `ufw` firewall.

``` 
$ sudo apt-get update
$ sudo apt-get install ufw
```

Lets deny all incoming traffic, by default via:  
`$ sudo ufw default deny incoming`

Allow some basic required ports:  
for ssh: `$ sudo ufw allow 7777`  
for https: `$ sudo ufw allow 443`  
for http: `$ sudo ufw allow 80`

Activate firewall via `$ sudo ufw enable` This will turn on the firewall, and also activate it on system startup.

The good way is to verify, if the ports are really opened, or blocked. Go to your terminal shell on your localhost machine and install nmap program:
`$ sudo apt-get install nmap`

Then run command, which will scan all open ports on server  
`$ sudo nmap vps1.matho.sk`

You should see the similar response:
```
Starting Nmap 7.80 ( https://nmap.org ) at 2023-10-22 12:44 CEST
Nmap scan report for vps.matho.sk (109.123.251.98)
Host is up (0.024s latency).
rDNS record for 109.123.251.98: vmi1489968.contaboserver.net
Not shown: 996 filtered ports
PORT     STATE  SERVICE
53/tcp   open   domain
80/tcp   closed http
443/tcp  closed https
7777/tcp open   cbt
```

### 2.4 Activate swap
The great article about activating swap you can find on this [link](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04) It is fully working also for Ubuntu 22.04 LTS. The only change is, that we want to activate 8GB of swap space, instead of 1GB in tutorial.

## 3 Docker and Docker Compose
No we will install Docker. Prerequisites you can find at [https://docs.docker.com/engine/install/ubuntu/#prerequisites](https://docs.docker.com/engine/install/ubuntu/#prerequisites)

Reload system:  
`$ sudo apt-get update`

Install docker dependencies:
```bash
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

Add ppa (to install on amd64 architecture):  
`$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
```bash
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

Reload the system again, to fetch new ppa:  
`$ sudo apt-get update`

And install docker:  
`$ sudo apt-get install docker-ce docker-ce-cli containerd.io`

Test installation via running hello-world image:  
`$ sudo docker run hello-world`

The Docker Compose overview page is located at https://docs.docker.com/compose/ . Installation page is located at https://docs.docker.com/compose/install/

The easiest method is to install via apt-get  
`$ sudo apt-get install docker-compose`

Echo the version to detect, if it is installed correctly:  
`$ docker-compose --version`

## 4 Docker swarm
The Docker swarm tutorial you can find on [https://docs.docker.com/engine/swarm/swarm-tutorial/](https://docs.docker.com/engine/swarm/swarm-tutorial/)

The first VPS (`ubuntu1`) will be manager and the second (`ubuntu2`) will be worker. The manager can also host websites like the worker, but the manager node will be responsible for request distribution across the cluster. The ubuntu1 manager node has private IP 10.0.0.1 and the second node has private IP 10.0.0.2.

The following ports must be available for communication between Docker nodes in private network:

TCP port `2377` for cluster management communications
on ubuntu1:
```
$ sudo ufw allow from 10.0.0.2 to any port 2377
$ sudo ufw deny 2377
```

on ubuntu2:
``` 
$ sudo ufw allow from 10.0.0.1 to any port 2377
$ sudo ufw deny 2377
```

TCP and UDP port `7946` for communication among nodes
on ubuntu1:
``` 
$ sudo ufw allow from 10.0.0.2 to any port 7946
$ sudo ufw deny 7946
```

on ubuntu2:
``` 
$ sudo ufw allow from 10.0.0.1 to any port 7946
$ sudo ufw deny 7946
```

UDP port `4789` for overlay network traffic
on ubuntu1:
``` 
$ sudo ufw allow from 10.0.0.2 to any port 4789
$ sudo ufw deny 4789
```

on ubuntu2:
``` 
$ sudo ufw allow from 10.0.0.1 to any port 4789
$ sudo ufw deny 4789
```

Lets create manager node (run on ubuntu1):  
`$ sudo docker swarm init --advertise-addr 10.0.0.1`

You will see similar response to this one:
``` 
Swarm initialized: current node (ft3xaaa5wzsn62tn48y) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2xg6... 10.0.0.1:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
Note: the token will be different.

If you want to add worker node to swarm, execute the recommended command on ubuntu2 from the output - something like:  
`$ sudo docker swarm join --token WMTKN-1-2xg6... 10.0.0.1:2377`

Check, if worker was added. On manager node run:    
`$ sudo docker node ls`

Now, we have running Docker swarm.

## 5 Gluster
Gluster is free and open source software scalable network system. In short - it ensures, that data you write to Gluster "shared folder" on one node are automatically replicated to second node. So in case your first server die, you will have 1:1 copy on another node. I use it to store the uploaded files from website admin backends and for auto Postgres backups.

The great article I have found is located on [https://www.digitalocean.com/community/tutorials/how-to-create-a-redundant-storage-pool-using-glusterfs-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-create-a-redundant-storage-pool-using-glusterfs-on-ubuntu-20-04)
Our difference is, that we do not have 3 nodes, but only 2. If you have two nodes like me, follow my steps.

We have two nodes. First node with `ubuntu1` hostname will be our `gluster1`. The second node with `ubuntu2` hostname will be our `gluster2`
Because we have two nodes, all nodes will be both server and client.
```
gluster1	Server, client
gluster2	Server, client
```

On all nodes, run:  
`$ sudo vim /etc/hosts`

and insert this config (use your private IPs instead)
```
10.0.0.1 gluster1.example.com gluster1
10.0.0.2 gluster2.example.com gluster2
```
You can change the domains, but to be as similar to the DigitalOcean article as is possible, I decided to keep the same domain names here.

Then install the Gluster on both nodes:
``` 
sudo add-apt-repository ppa:gluster/glusterfs-9
sudo apt update
```

Because we have 2 Gluster servers, install it on both nodes:  
`$ sudo apt install glusterfs-server`

`Note:` although I have set `ppa:gluster/glusterfs-9`, it installed `v10` instead of `v9`. I'm not sure, if that is due to some bugs on Ubuntu 22.04, but it doesnt matter.

Then also on both nodes: start, activate on system start and show status of service via:
```
$ sudo systemctl start glusterd.service
$ sudo systemctl enable glusterd.service
$ sudo systemctl status glusterd.service
```

Then we want to set firewall. On `gluster1` node, execute
```
$ sudo ufw allow from 10.0.0.2 to any port 24007
$ sudo ufw allow from 10.0.0.2 to any port 24008
```

On `gluster2` node, execute
``` 
$ sudo ufw allow from 10.0.0.1 to any port 24007
$ sudo ufw allow from 10.0.0.1 to any port 24008
```

On both nodes, deny connection from public:
``` 
$ sudo ufw deny 24007
$ sudo ufw deny 24008
```

On both nodes:
```
$ sudo gluster peer status
```

On `gluster1`:
``` 
$ sudo gluster volume create volume1 replica 2 gluster1.example.com:/gluster-storage gluster2.example.com:/gluster-storage force
$ sudo gluster volume start volume1
$ sudo gluster volume status
```

In the latest versions, the port for inter-nodes communication is selected randomly. To be able deny/allow specific ports, it is good to hardcode it to Gluster config.

Navigate to Gluster config:
`$ sudo vim /etc/glusterfs/glusterd.vol`

and change the ports according to this:
``` 
 #   option base-port 49152
     option max-port  49153 # Initial value was port 60999
```

Change this port on both nodes and restart Gluster on both nodes:
```
$ sudo systemctl restart glusterd.service
```

Then allow the hardcoded port for inter-node communication:

On `gluster1`:
```
$ sudo ufw allow from 10.0.0.2 to any port 49152
$ sudo ufw allow from 10.0.0.2 to any port 49153
```

On `gluster2`:
```
$ sudo ufw allow from 10.0.0.1 to any port 49152
$ sudo ufw allow from 10.0.0.1 to any port 49153
```

On both nodes:
```
$ sudo ufw deny 49152
$ sudo ufw deny 49153
```

Because we have 2 clients, run on both nodes:  
`$ sudo apt install glusterfs-client`

On both nodes:  
`$ sudo mkdir /storage-pool`

On `gluster1`:  
`$ sudo mount -t glusterfs gluster1.example.com:/volume1 /storage-pool`

On `gluster2`:  
`$ sudo mount -t glusterfs gluster2.example.com:/volume1 /storage-pool`

On `gluster1`:
``` 
$ cd /storage-pool
$ sudo touch file_{0..9}.test
```

On `gluster1`:  
`$ sudo gluster volume set volume1 auth.allow 10.0.0.2`

On `gluster2`:  
`$ sudo gluster volume set volume1 auth.allow 10.0.0.1`

On both nodes:
``` 
$ sudo gluster volume info
$ sudo gluster peer status
```

On both nodes:
``` 
$ sudo gluster volume profile volume1 start
$ sudo gluster volume profile volume1 info
$ sudo gluster volume status
```
If you need explanation of some steps, check the [great article](https://www.digitalocean.com/community/tutorials/how-to-create-a-redundant-storage-pool-using-glusterfs-on-ubuntu-20-04) already linked at the beginning of this Gluster section.

### 5.1 Gluster - auto mounting

How to do auto mounting is mentioned in [this article](https://hexadix.com/setup-network-shared-folder-using-glusterfs-ubuntu-servers-backup-server-array-auto-mount/)

In theory adding the following line to the client’s fstab file should make the client mount the GlusterFS share at boot:
On the first node, write to fstab  
`$ sudo vim /etc/fstab`
this content:
`gluster1.example.com:/volume1 /storage-pool glusterfs defaults,_netdev 0 0`

On second node, write to fstab
`$ sudo vim /etc/fstab`
this content:
`gluster2.example.com:/volume1 /storage-pool glusterfs defaults,_netdev 0 0`

Contabo has specific `/etc/hosts` - after VPS restart, records are not persistent. See the following comment:
```
# Your system has configured 'manage_etc_hosts' as True.
2 # As a result, if you wish for changes to this file to persist
3 # then you will need to either
4 # a.) make changes to the master file in /etc/cloud/templates/hosts.debian.tmpl
5 # b.) change or remove the value of 'manage_etc_hosts' in
6 #     /etc/cloud/cloud.cfg or cloud-config from user-data
```

On all nodes, run:  
`$ sudo vim /etc/cloud/templates/hosts.debian.tmpl`

and insert this config (use your private IPs instead)
```
10.0.0.1 gluster1.example.com gluster1
10.0.0.2 gluster2.example.com gluster2
```

Normally the fstab record should work since the _netdev param should force the filesystem to wait for a network connection. If the fstab didn’t work for you because the GlusterFS client wasn’t running when the fstab file was processed, you can use following approach.

Delete the added lines from the /etc/fstab.

Add the mount command to `/etc/rc.local` file. We will not add it to /etc/fstab as rc.local is always executed after the network is up which is required for a network file system.

On `ubuntu1` node:  
`$ sudo vim /etc/rc.local`
add content:
``` 
#!/bin/sh -e

/usr/sbin/mount.glusterfs gluster1.example.com:/volume1 /storage-pool

exit 0
```
`$ sudo chmod u+x /etc/rc.local`

On ubuntu2 node:  
`$ sudo vim /etc/rc.local`
add content:
``` 
#!/bin/sh -e

/usr/sbin/mount.glusterfs gluster2.example.com:/volume1 /storage-pool

exit 0
```

`$ sudo chmod u+x /etc/rc.local`

This should work. Reboot both nodes, and it should auto mount correctly. It should work, but if you reboot the nodes both at the same time, the issues could persist.
Run `df -h` on boot to check `/storage-pool` is mounted. Also check `ls -la /storage-pool` on both nodes.

### 5.2 Gluster self healing
Gluster Self Heal helps to heal data on the Gluster bricks when there are some inconsistencies among the replica pairs in the volume. Pro-active self-heal daemon runs in the background, diagnoses issues and automatically initiates self-healing every 10 minutes on the files which require healing.

If you have issues with self-healing, check the following issue on Github [https://github.com/gluster/glusterfs/issues/3138](https://github.com/gluster/glusterfs/issues/3138)

Also, check the following commands in doc, to manually check or trigger self heal process

```
$ sudo gluster volume heal volume1
$ sudo gluster volume heal volume1 full
$ sudo gluster volume heal volume1 info
$ sudo gluster volume heal volume1 info summary

$ sudo tail -n 200 /var/log/glusterfs/glusterd.log
``` 

## 6 Portainer
If your infrastructure becomes big, it is usefull to have possibility to maintain the Docker stack via GUI, instead of linux commands. Portainer is GUI on top of Docker containers - Docker Compose and/or Docker Swarm. It enables you to manage your Docker Swarm via clicking in GUI, instead of linux commands. You can create stacks, up/down scaling, monitor your Docker containers or view the cluster overview - which container is running under which VPS. This is a must tool, you really want to have installed!

To be able run Portainer, allow this ports in firewall:
On ubuntu1:  
`$ sudo ufw allow 9443`

The 8000 port is optional:  
`$ sudo ufw allow 8000`

On ubuntu1:
``` 
sudo ufw allow from 10.0.0.2 to any port 9001
sudo ufw deny 9001
```

On ubuntu2:
``` 
sudo ufw allow from 10.0.0.1 to any port 9001
sudo ufw deny 9001
```

On ubuntu1:  
`$ sudo mkdir /storage-pool/portainer_data`

The installation steps are decribed [here](https://docs.portainer.io/start/install-ce/server/swarm/linux)
```
$ curl -L https://downloads.portainer.io/ce2-19/portainer-agent-stack.yml -o portainer-agent-stack.yml
$ sudo docker stack deploy -c portainer-agent-stack.yml portainer 
```

Now that the installation is complete, you can log into your Portainer Server instance by opening a web browser and going to `https://vps1.matho.sk:9443`

But how to mount the Portainer to some subdomain and cover by ssl cert? To achieve it, we will install Traefik.

`NOTE:` In the end after the end of tutorial, I recommend to do not exposed this 9443, 9001 and 8000 ports. Portainer will be communicating via private network and the http(s) domain will be server by Traefik - configured in the next chapter.

`Note 2:` I have found, that Docker swarm opened ports overwrite the behaviour of `ufw` firewall. Simply said, you cant block exposed ports from Docker swarm via `ufw` firewall. Take this into account, please and if not needed, do not open Docker swarm ports to public.

## 7 Traefik installation
Traefik is reverse-proxy sitting at the front of your Docker Swarm. It can forward requests to proper Docker containers and also can manage ssl certificates. It means, that you no more need to manually reissue Lets Encrypt certificates or setup some ssl renew process. It can handle cert renew process automatically. Traefik overview page is located at [https://doc.traefik.io/traefik/v2.2/providers/overview/](https://doc.traefik.io/traefik/v2.2/providers/overview/)

``` 
sudo ufw delete allow 8080
sudo ufw allow 8081
```

My Traefik instance will be sitting on `traefik.vps1.matho.sk`. If you did not setup "wildcard" DNS A record for any `vps1` subdomain, set it manually now.

Create Docker stack file for Traefik config:  
`vim ~/docker/traefik_stack.yml`

with following content: (replace your email and password in the config)
```yaml
version: '3.6'

networks:
  db-nw:
    driver: overlay
    attachable: true

services:
  reverse-proxy:
    image: traefik:v2.10
    command:
      - "--api.dashboard=true"
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=true"
      - "--providers.docker.network=db-nw"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web-secure.address=:443"
      - --certificatesresolvers.le.acme.email=EMAIL@YOURDOMAIN
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.le.acme.httpchallenge=true
      - --certificatesresolvers.le.acme.httpchallenge.entrypoint=web  
      # - --certificatesresolvers.le.acme.tlschallenge=true
      - --serverstransport.insecureskipverify=true  
      #- "--log.level=DEBUG"
      - --providers.docker=true
      - --providers.file.directory=/etc/traefik/dynamic_conf  
    ports:
      # The HTTP port
      - 80:80
      # The Web UI (enabled by --api.insecure=true)
      - 8081:8080
      - 443:443
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /root/letsencrypt:/letsencrypt  
      - /root/traefik/tools/certs:/tools/certs
      - /root/traefik/tools/traefik/config.yml:/etc/traefik/dynamic_conf/conf.yml:ro
    networks:
      - db-nw

    deploy:
      labels:
        - "traefik.http.routers.api.rule=Host(`traefik.vps1.matho.sk`)"
        - "traefik.http.routers.api.service=api@internal"
        - "traefik.http.routers.api.middlewares=auth"
        - "traefik.http.middlewares.auth.basicauth.users=YOUR-USER:YOUR-PASSWORD-FINGERPRINT"
        - "traefik.http.services.dummy-svc.loadbalancer.server.port=9999"
       
        - traefik.http.routers.api.tls.certresolver=le
        - traefik.http.routers.api.entrypoints=web-secure
        - traefik.http.services.api.loadbalancer.server.port=8080

      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
```

By default, your Traefik admin is publicly available - without some type of basic auth. You can generate admin:password for example on the page [https://hostingcanada.org/htpasswd-generator/](https://hostingcanada.org/htpasswd-generator/) . Insert username and password, select apache specific format and copy the credentials to Traefik label called `traefik.http.middlewares.auth.basicauth.users=`.

`Note:` escape dolar symbols. For each dolar, prepend another one dolar.

Then lets create some folders, where certificates will be stored for Traefik:
``` 
$ sudo mkdir /root/letsencrypt
$ sudo mkdir -p /root/traefik/tools/certs
$ sudo mkdir -p /root/traefik/tools/traefik
$ sudo touch /root/traefik/tools/traefik/config.yml
```

Deploy the prepared Docker stack via:  
`$ sudo docker stack deploy -c traefik_stack.yml traefik`

``` 
$ sudo ufw allow 8081
$ sudo ufw allow 80
```
On both nodes  
`$ sudo ufw allow 2375`

Now we have running Traefik on `traefik.vps1.matho.sk` subdomain. It is covered with ssl cert from Lets Encrypt and http basic auth via admin:password, you have generated.

We will move Portainer to subdomain in next chapter.

### 7.1 Setup Portainer to subdomain
Currently, we have Portainer running on  `https://vps1.matho.sk:9443`. But it would be nice to have own subdomain, like `https://portainer.vps1.matho.sk`, instead.

Modify the Portainer Docker stack this way - see the changed `deploy > labels`:

```yaml
version: '3.2'

services:
  agent:
    image: portainer/agent:2.19.1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    image: portainer/portainer-ce:2.19.1
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      #- "9443:9443"
      - "9000:9000"
      - "8000:8000"
    volumes:
      - portainer_data:/data
    networks:
      - traefik_db-nw
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      labels:
        - "traefik.http.routers.portainer.rule=Host(`portainer.vps1.matho.sk`)"
        - traefik.http.routers.portainer.tls.certresolver=le
        - traefik.http.routers.portainer.entrypoints=web-secure
        - traefik.http.services.portainer.loadbalancer.server.port=9000       
        - traefik.docker.network=traefik_db-nw 

networks:
  traefik_db-nw:
    external: true
  agent_network:
    driver: overlay
    attachable: true

volumes:
  portainer_data:
```

Ensure, you have wildcard domain for `*.vps1.matho.sk` or ensure you have setup DNS for `portainer` subdomain. In the Docker stack, change the Host rule for your domain

Deploy the stack:  
`$ sudo docker stack deploy -c portainer-agent-stack.yml portainer`

Give it few seconds to restart the containers and point `https://portainer.vps1.matho.sk` in your browser. If not reachable in few minutes, restart Traefik. Then you should see the Portainer on the given subdomain with the ssl certificate.

At the end, remove the exposed https port. We no longer need it, as Traefik is doing the redirection of the requests. Remove the ports section in Portainer stack.

## 8 Docker Registry
The most popular Docker hosting is [Docker hub](https://hub.docker.com/). You can host your pre-builded Docker images there for free, if the images are public. But I dont want to publish my images of my private projects. So there are few alternatives, like:
1) I will pay to Docker hub to host my private images there
2) I will setup Gitlab to my server and use the Gitlab's Docker registry
3) or I will install some open source registry project to my VPS

I selected the last one. I will install the `registry2` project to my server. It is for free. But there is very big disadvantage - if your server goes down, you will not have ability to push/pull to Docker registry. But I'm accepting the risk - in case of total server failure, I can rebuild the images in few minutes manually - the plain Dockerfiles (not docker images) I'm hosting in another repository on another server.

I did steps based on this article - [https://gabrieltanner.org/blog/docker-registry/](https://gabrieltanner.org/blog/docker-registry/)
On ubuntu1:  
``` 
$ sudo apt install apache2-utils
$ sudo mkdir -p /data/registry2/data
$ sudo mkdir -p /data/registry2/auth
```

Generate password for user `matho`  
`$ htpasswd -Bc registry.password matho`

Move the password file to Docker volume:  
`mv registry.password /data/registry2/auth/registry.password`

Create Docker stack recipe:  
`vim /root/docker/registry2-stack.yml`

with the following content:
```yaml
version: '3'

services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    environment:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.password
    volumes:
      - /data/registry2/data:/data
      - /data/registry2/auth:/auth
    networks:
      - traefik_db-nw
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      labels:
        - "traefik.http.routers.registry.rule=Host(`registry.vps1.matho.sk`)"
        - traefik.http.routers.registry.tls.certresolver=le
        - traefik.http.routers.registry.entrypoints=web-secure
        - traefik.http.services.registry.loadbalancer.server.port=5000       
        - traefik.docker.network=traefik_db-nw
networks:
  traefik_db-nw:
    external: true
```

This Docker registry will be running on `registry.vps1.matho.sk`. Again ensure, this subdomain has valid DNS A record pointing to the VPS IP address, or wildcard DNS record is set.

Deploy the Docker stack via:  
`$ sudo docker stack deploy -c registry2-stack.yml registry2`

Then, lets do some testing. Pull the Alpine project on ubuntu1:
```
$ docker pull alpine
$ docker tag alpine registry.vps1.matho.sk/my-alpine2
$ sudo docker images
```

Login to your selfhosted Docker registry and push the `my-alpine2` image:
```
$ docker login registry.vps1.matho.sk
$ docker push registry.vps1.matho.sk/my-alpine2
```

Try to pull the image on `ubuntu2` host:
```
$ docker login registry.vps1.matho.sk
$ sudo docker pull registry.vps1.matho.sk/my-alpine2
```

If login and pull passed correctly, we are done.

`Note`: I highly recommend to save this Docker stack recipes to some git repo different from this two servers. You can use public [https://gitlab.com](https://gitlab.com) service, where you can host some repositories for free.

## 9 Zabbix
Zabbis is infrastructure monitoring tool. It can send you email for example in case that one of your server is down, or when some problem occured.

Zabbix is provided as `docker-compose` recipe. That provided docker-compose recipe is slightly different from the Docker Stack recipe, I like to use and prefer.
The Zabbix Docker git repository you can find on [https://github.com/zabbix/zabbix-docker](https://github.com/zabbix/zabbix-docker) My Docker stack recipe is forked from `docker-compose_v3_ubuntu_pgsql_latest.yaml` file, which you can find in the git repository.

Create folder for Zabbix Docker stack file and files with ENV values:  
`$ sudo mkdir ~/docker-stacks/zabbix`  
`$ sudo cd ~/docker-stacks/zabbix`

Create file for the modified Docker stack recipe:  
`$ sudo vim docker-compose_v3_ubuntu_pgsql_latest.yaml`

Insert following Docker stack recipe:
```yaml
version: '3.8'
services:
 zabbix-server:
  image: zabbix/zabbix-server-pgsql:ubuntu-6.4-latest
  hostname: zabbix-server
  ports:
    - "10051-10051"
  volumes:
   - /etc/localtime:/etc/localtime:ro
   - /data/zabbix/zbx_env/usr/lib/zabbix/alertscripts:/usr/lib/zabbix/alertscripts:ro
   - /data/zabbix/zbx_env/usr/lib/zabbix/externalscripts:/usr/lib/zabbix/externalscripts:ro
   - /data/zabbix/zbx_env/var/lib/zabbix/dbscripts:/var/lib/zabbix/dbscripts:ro
   - /data/zabbix/zbx_env/var/lib/zabbix/export:/var/lib/zabbix/export:rw
   - /data/zabbix/zbx_env/var/lib/zabbix/modules:/var/lib/zabbix/modules:ro
   - /data/zabbix/zbx_env/var/lib/zabbix/enc:/var/lib/zabbix/enc:ro
   - /data/zabbix/zbx_env/var/lib/zabbix/ssh_keys:/var/lib/zabbix/ssh_keys:ro
   - /data/zabbix/zbx_env/var/lib/zabbix/mibs:/var/lib/zabbix/mibs:ro
   - snmptraps:/var/lib/zabbix/snmptraps:rw
  ulimits:
   nproc: 65535
   nofile:
    soft: 20000
    hard: 40000
  deploy: 
   mode: replicated
   replicas: 1
   placement:
    constraints: [node.role == manager]    
   resources:
    limits:
      cpus: '0.70'
      memory: 1G
    reservations:
      cpus: '0.5'
      memory: 512M
  env_file:
   - ./env_vars/.env_db_pgsql
   - ./env_vars/.env_srv
  secrets:
   - POSTGRES_USER
   - POSTGRES_PASSWORD
  depends_on:
   - postgres-server
  networks:
   zabbix_network:
     aliases:
       - zabbix-server
  stop_grace_period: 30s
  sysctls:
   - net.ipv4.ip_local_port_range=1024 64999
   - net.ipv4.conf.all.accept_redirects=0
   - net.ipv4.conf.all.secure_redirects=0
   - net.ipv4.conf.all.send_redirects=0
  labels:
   - "com.zabbix.description=Zabbix server with PostgreSQL database support"
   - "com.zabbix.company=Zabbix LLC"
   - "com.zabbix.component=zabbix-server"
   - "com.zabbix.dbtype=pgsql"
   - "com.zabbix.os=ubuntu"

 zabbix-web-nginx-pgsql:
  image: zabbix/zabbix-web-nginx-pgsql:ubuntu-6.4-latest
  #ports:
  # - "8085:8080"
  # - "4435:8443"
  volumes:
   - /etc/localtime:/etc/localtime:ro
   - /data/zabbix/zbx_env/etc/ssl/nginx:/etc/ssl/nginx:ro
   - /data/zabbix/zbx_env/usr/share/zabbix/modules/:/usr/share/zabbix/modules/:ro
  deploy:
   mode: replicated
   replicas: 1
   placement:
     constraints: [node.role == manager]
   labels:
    - "traefik.http.routers.zabbix.rule=Host(`zabbix.vps1.matho.sk`)"
    - "traefik.http.routers.zabbix.tls.certresolver=le"
    - "traefik.http.routers.zabbix.entrypoints=web-secure"
    - "traefik.http.services.zabbix.loadbalancer.server.port=8080"
    - "traefik.docker.network=traefik_db-nw"     
   resources:
    limits:
      cpus: '0.70'
      memory: 512M
    reservations:
      cpus: '0.5'
      memory: 256M
  env_file:
   - ./env_vars/.env_db_pgsql
   - ./env_vars/.env_web
  secrets:
   - POSTGRES_USER
   - POSTGRES_PASSWORD
  depends_on:
   - postgres-server
   - zabbix-server
  healthcheck:
   test: ["CMD", "curl", "-f", "http://localhost:8080/ping"]
   interval: 10s
   timeout: 5s
   retries: 3
   start_period: 30s
  networks:
   traefik_db-nw:
   zabbix_network:
  stop_grace_period: 10s
  sysctls:
   - net.core.somaxconn=65535
  labels:
   - "com.zabbix.description=Zabbix frontend on Nginx web-server with PostgreSQL database support"
   - "com.zabbix.company=Zabbix LLC"
   - "com.zabbix.component=zabbix-frontend"
   - "com.zabbix.webserver=nginx"
   - "com.zabbix.dbtype=pgsql"
   - "com.zabbix.os=ubuntu"  

 zabbix-agent:
   image: zabbix/zabbix-agent2:ubuntu-6.4-latest
   hostname: zabbix-agent
   user: 0:0
   ports:
     - "10050-10050"
   volumes:
     - /etc/localtime:/etc/localtime:ro
     - /data/zabbix/zbx_env/etc/zabbix/zabbix_agentd.d:/etc/zabbix/zabbix_agentd.d:ro
     - /data/zabbix/zbx_env/var/lib/zabbix/enc:/var/lib/zabbix/enc:ro
     - /data/zabbix/zbx_env/var/lib/zabbix/buffer:/var/lib/zabbix/buffer
     - /data/zabbix/zbx_env/var/lib/zabbix/ssh_keys:/var/lib/zabbix/ssh_keys:ro
     - /var/run/docker.sock:/var/run/docker.sock
   deploy:
     mode: global
     resources:
       limits:
         cpus: '0.2'
         memory: 128M
       reservations:
         cpus: '0.1'
         memory: 64M
   env_file:
     - ./env_vars/.env_agent
   privileged: true
   pid: "host"
   networks:
     zabbix_network:
       aliases:
         - zabbix-agent
   stop_grace_period: 5s
   labels:
     com.zabbix.description: "Zabbix agent"
     com.zabbix.company: "Zabbix LLC"
     com.zabbix.component: "zabbix-agentd"
     com.zabbix.os: "ubuntu"   

 postgres-server:
  image: postgres:15-alpine
#  command: -c ssl=on -c ssl_cert_file=/run/secrets/server-cert.pem -c ssl_key_file=/run/secrets/server-key.pem -c ssl_ca_file=/run/secrets/root-ca.pem
  volumes:
   - /data/zabbix/zbx_env/var/lib/postgresql/data:/var/lib/postgresql/data:rw
   - /data/zabbix/env_vars/.ZBX_DB_CA_FILE:/run/secrets/root-ca.pem:ro
   - /data/zabbix/env_vars/.ZBX_DB_CERT_FILE:/run/secrets/server-cert.pem:ro
   - /data/zabbix/env_vars/.ZBX_DB_KEY_FILE:/run/secrets/server-key.pem:ro
  deploy:
   mode: replicated
   replicas: 1
   placement:
     constraints: [node.role == manager] 
  env_file:
   - ./env_vars/.env_db_pgsql
  secrets:
   - POSTGRES_USER
   - POSTGRES_PASSWORD
  stop_grace_period: 1m
  networks:
    zabbix_network:
   

networks:
  zabbix_network:
    attachable: true
    internal: true
    ipam:
      driver: default
      config:
        - subnet: 172.16.239.0/24
  traefik_db-nw:
    external: true

volumes:
  snmptraps:

secrets:
  MYSQL_USER:
    file: ./env_vars/.MYSQL_USER
  MYSQL_PASSWORD:
    file: ./env_vars/.MYSQL_PASSWORD
  MYSQL_ROOT_USER:
    file: ./env_vars/.MYSQL_ROOT_USER
  MYSQL_ROOT_PASSWORD:
    file: ./env_vars/.MYSQL_ROOT_PASSWORD
  POSTGRES_USER:
    file: ./env_vars/.POSTGRES_USER
  POSTGRES_PASSWORD:
    file: ./env_vars/.POSTGRES_PASSWORD
```

Then create `env_vars` folder in the current folder and write or fetch ENV files from this repository folder: [https://github.com/zabbix/zabbix-docker/tree/6.4/env_vars](https://github.com/zabbix/zabbix-docker/tree/6.4/env_vars)

When comparing to the official `docker-compose_v3_ubuntu_pgsql_latest.yaml` file, I removed some services from the stack. Zabbix maintainers wrote to config also configuration for MySQL database, although you do not need that service, if you use PostgreSQL database. Also I removed `zabbix-proxy-sqlite3`, `zabbix-proxy-mysql`, `zabbix-web-apache-pgsql`, `zabbix-java-gateway`, `zabbix-snmptraps`, `zabbix-web-service`, `mysql-server`, `db_data_mysql`. I think I do not need to run this services, for now. So it will save some memory and cpu.

Also, I rewrote the network configuration, and modified the version of Zabbix agent to second version. The first version has issues with Docker monitoring, so I installed the second version. As you can see, I'm using `mode: global`. This means, that this agent is running both on `ubuntu1` and also on second worker with hostname `ubuntu2`. I attached Zabbix nginx service to `traefik_db-nw` network to be able use the service on the subdomain via Traefik.

I had issues with activating agent monitoring (agent communication with Zabbix server node). I fixed it with following configuration
```yaml
 networks:
   zabbix_network:
     aliases:
       - zabbix-agent
```
and with specification `hostname: zabbix-agent`. Thanks to this, I can use DNS service configuration `zabbix-agent` when activating agent communication in Zabbix.

The Zabbix agent runs under a user named `zabbix`. By default `zabbix` is not allowed to execute Docker commands. But that is necessary to check the health of single Docker containers. To be able monitor services, I set the `user: 0:0` setting. This gives the container the `root` permissions of the host operating system.

Create Docker volumes via:  
`sudo mkdir -p /data/zabbix`

You can install the Docker stack via:  
`$ sudo docker stack deploy -c docker-compose_v3_ubuntu_pgsql_latest.yaml zabbix`

Zabbix will be running on `zabbix.vps1.matho.sk` domain. Feel free to modify the domain config according your needs. Also ensure, you specified the DNS A record for this domain.
The default credentials for Zabbix backend are: `Admin:zabbix` I recommend to change password immediately via menu `Users > users > Admin`

You can setup email trigger based on article [https://www.zabbix.com/documentation/current/en/manual/config/notifications/media/email](https://www.zabbix.com/documentation/current/en/manual/config/notifications/media/email)


### 9.1 Zabbix agent configuration

We need to allow firewall for Zabbix server <-->  Zabbix agent communication. Follow this steps:   
On `ubuntu1` node:
``` 
sudo ufw allow from 10.0.0.2 to any port 10050
sudo ufw deny 10050

sudo ufw allow from 10.0.0.2 to any port 10051
sudo ufw deny 10051
```

On `ubuntu2` node:
```
sudo ufw allow from 10.0.0.1 to any port 10050
sudo ufw deny 10050

sudo ufw allow from 10.0.0.1 to any port 10051
sudo ufw deny 10051
```

After it, you need to add new server in Zabbix backend. You can see screenshots in article https://techexpert.tips/zabbix/zabbix-monitor-linux-using-agent/ in the section `Tutorial - Zabbix Monitor Linux`

Go to `Data collection > Hosts > Create host` and fill in settings. You will have to enter the following information:
- `Host Name` - enter a Hostname to identify the Linux server - `ubuntu2`
- `Visible Hostname` - repeat the hostname.
- `Templates` - select Templates / Operating systems > `Linux by Zabbix agent`
- `host group` - enter a name to identify a group of similar devices - `linux servers`
- `Agent Interface` - instead of IP, specify DNS name `zabbix-agent` and switch `Connect to` to DNS instead of IP value.

Submit the form. You will need to modify the Agent DNS name also for the default `Zabbix server` configuration. Open the modal and modify IP configuration the DNS configuration with the same values - set `zabbix-agent` as DNS name and Connect to to DNS setting. After you submit the form, you should see green `ZBX` label under the Availability column. It means, your Zabbix server with Zabbix agent connection works.

Try to reboot the second node. Zabbix should let you know about the problem on the dashboard. Do some advanced configuration of Zabbix - like set the email notification service.

## 10 Minio
Minio is a high performance distributed object storage server. You can use it to self-host some files, you do not want to publish on Google drive or Dropbox. The files you can share with your friends or colleagues.

Navigate to the folder with our Docker stacks recipies:  
`cd ~/docker-stacks`

Create the folder for minio recipe:
```
$ mkdir minio
$ cd minio
```

Create the file with ENV config:   
`vim minio.env`

with content:
```
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=YOUR-PASSWORD
MINIO_BROWSER_REDIRECT_URL=https://minio-console.vps1.matho.sk
MINIO_SERVER_URL=https://minio.vps1.matho.sk
```

Then create minio stack file:  
`vim minio-stack.yml`

with content:
```yaml
version: '3.2'

services:
  minio:
    image: minio/minio:RELEASE.2023-10-25T06-33-25Z
    env_file: /root/docker-stacks/minio/minio.env
    volumes:
     - /var/data/minio:/data
    networks:
      - traefik_db-nw
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      labels:
        - traefik.http.middlewares.redirect-https.redirectScheme.scheme=https
        - traefik.http.middlewares.redirect-https.redirectScheme.permanent=true

        - traefik.http.routers.minio-https.rule=Host(`minio.vps1.matho.sk`)
        - traefik.http.routers.minio-https.tls.certresolver=le
        - traefik.http.routers.minio-https.entrypoints=web-secure
        - traefik.http.routers.minio-https.service=minio
                
        - traefik.http.routers.minio-http.rule=Host(`minio.vps1.matho.sk`)
        - traefik.http.routers.minio-http.entrypoints=web
        - traefik.http.routers.minio-http.middlewares=redirect-https
        - traefik.http.routers.minio-http.service=minio
        - traefik.http.services.minio.loadbalancer.server.port=9000
        - traefik.docker.network=traefik_db-nw

        - traefik.http.routers.minio-console-https.rule=Host(`minio-console.vps1.matho.sk`)
        - traefik.http.routers.minio-console-https.tls.certresolver=le
        - traefik.http.routers.minio-console-https.entrypoints=web-secure
        - traefik.http.routers.minio-console-https.service=minio-console
        
        - traefik.http.routers.minio-console-http.rule=Host(`minio-console.vps1.matho.sk`)
        - traefik.http.routers.minio-console-http.entrypoints=web
        - traefik.http.routers.minio-console-http.middlewares=redirect-https
        - traefik.http.routers.minio-console-http.service=minio-console
        - traefik.http.services.minio-console.loadbalancer.server.port=9001
    command:  minio server /data --console-address ":9001"

networks:
  traefik_db-nw:
    external: true
```

Crete Docker volume:   
`$ sudo mkdir -p /var/data/minio`

Deploy the stack via:  
`$ sudo docker stack deploy minio -c /root/docker-stacks/minio/minio-stack.yml`

If you do not see the Minio running on the domain `minio.vps1.matho.sk`, do restart Traefik container and refresh the domain.

Log into the console running on `minio-console.vps1.matho.sk` and create first bucket. Then you can upload/download some files to test, that Minio works.

## 11 Postgres HA - Spilo, Haproxy, Pgbouncer, Pgadmin4
### 11.1 Spilo - build Docker image

We will use `Spilo` by Zalando. The Github repo you can find on [https://github.com/zalando/spilo](https://github.com/zalando/spilo)

You can use the image from official url [https://ghcr.io/zalando/spilo-15:3.0-p1](https://ghcr.io/zalando/spilo-15:3.0-p1) or you can build own image. We will build own Docker image.

Create folder for Docker builds:
```
$ mkdir ~/docker-builds
cd ~/docker-builds
```

At the time of writing this tutorial, the latest version on Github is `3.0.-p1`. Download and unzip the version:
``` 
$ wget https://github.com/zalando/spilo/archive/refs/tags/3.0-p1.zip
$ apt install unzip
$ unzip 3.0-p1.zip
```

We will follow official guide from [https://github.com/zalando/spilo/tree/3.0-p1](https://github.com/zalando/spilo/tree/3.0-p1)
```
$ cd postgres-appliance
```

The PostgreSQL version 15 is specified in the Dockerfile, so I prefer to use the v15, although at the time of writing this article is v16 available.
You can specify `PGOLDVERSIONS` ENV value. If  you do not specify, the older PostgreSQL versions will be also installed. We do not want this, we want only v15. So pass the empty ENV value for `PGOLDVERSIONS`

Build the Docker image:  
`$ sudo docker build --no-cache --build-arg PGVERSION=15 --build-arg PGOLDVERSIONS= -t registry.vps1.matho.sk/spilo:3.0-p1  .`

The build will be ready in 5 minutes. Push it to our Docker registry:
``` 
$ sudo docker login registry.vps1.matho.sk
$ sudo docker push registry.vps1.matho.sk/spilo:3.0-p1
```

Now we have Docker image and we can deploy Docker stack.

### 11.2 Spilo - deploy Docker stack

Navigate to our Docker stack folder:   
`$ cd ~/docker-stacks`

Create stack file:
`$ vim spilo-stack.yml`

```yaml
version: '3'

services:
  spilo_1:
    image: registry.vps1.matho.sk/spilo:3.0-p1
    user: 0:0
    hostname: spilo_1
    #ports:
    #  - "5432:5432"
    #  - "8008:8008"
    environment:
      - PATRONI_POSTGRESQL_LISTEN=spilo_1:5432
      - SCOPE=sftsscope
    volumes:      
      - "/data/spilo/spilo/pgdata:/home/postgres/pgdata"
      - "/data/spilo/spilo/postgres.yml:/home/postgres/postgres.yml"
    networks:
      traefik_db-nw:
        aliases:
          - spilo_1
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
        
  spilo_2:
    image: registry.vps1.matho.sk/spilo:3.0-p1
    user: 0:0
    hostname: spilo_2
    #ports:
    #  - "5433:5432"
    #  - "8009:8008"
    environment:
      - PATRONI_POSTGRESQL_LISTEN=spilo_2:5432
      - SCOPE=sftsscope
    volumes:
      - "/data/spilo/spilo/pgdata:/home/postgres/pgdata"
      - "/data/spilo/spilo/postgres.yml:/home/postgres/postgres.yml"
    networks:
      traefik_db-nw:
        aliases:
          - spilo_2
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == worker]        

networks:
  traefik_db-nw:
    external: true
```

Prepare Docker volumes on both nodes:
```
sudo mkdir -p /data/spilo/spilo/pgdata
```
For the first run, comment the Docker volumes path in the Docker stack recipe, to start containers without the volumes configuration.

Deploy the Docker stack via:  
`$ sudo docker stack deploy -c spilo-stack.yml spilo`

When container is started, log in to the Spilo container via Portainer under `root` user. Copy the `postgres.yml` file from the `/home/postgres` folder.
Insert the `postgres.yml` file under `/data/spilo/spilo/postgres.yml` and modify the configuration (user passwords) according to your need. Then uncomment Docker volumes in the Docker stack and repeat process also on second node. At the end, redeploy the Docker stack.

Then check, if the Spilo is runing. Open in broser `http://vps1.matho.sk:8008` Note - it is plain http protocol, not secured. If you see `state: running` in the JSON, it should be working.

Because we do not need the open ports 5432/8008, do redeploy of the Docker stack with commented port section. Then you should NOT see available port on `http://vps1.matho.sk:8008`

`Note:` When you will be reimporting old Postgres databases, it could be good to open 5432 port temporary. But better solution is to open ssh tunnel to the 5432 port.

### 11.3 Firewall rules for Docker Swarm exposed ports

Do you see open port for `http://vps1.matho.sk:8008` although you have deny it in ufw? Yes, this is known problem with UFW and Docker integration. I expected, that command `sudo ufw delete allow 8008` will work and I will not be able to see that port as opened.

The problem is in-depth described in this repo [https://github.com/chaifeng/ufw-docker](https://github.com/chaifeng/ufw-docker) I tried to install it and use, but then I found out, that after system reboot the `ufw-docker` records could not work, as expected. So instead hacking with `ufw-docker` I recommend to do not expose ports do host in Docker swarm, if not really needed.

### 11.4 Spilo - install PgAdmin

Create folder for Docker stack recipes on `ubuntu1`
``` 
$ mkdir /root/docker-stacks/pgadmin
$ cd /root/docker-stacks/pgadmin
$ vim /root/docker-stacks/pgadmin/pgadmin.env
```

with content (change it)
``` 
PGADMIN_DEFAULT_EMAIL=YOUR-EMAIL
PGADMIN_DEFAULT_PASSWORD=YOUR-PASSWORD
```

Then create Docker stack file:  
`$ vim /root/docker-stacks/pgadmin/pgadmin-stack.yml`

```yaml
version: '3'

services:
  pgadmin4_1:
    image: dpage/pgadmin4:7.8
    env_file: /root/docker-stacks/pgadmin/pgadmin.env 
    networks:
      - traefik_db-nw
    volumes:
      - "/data/pgadmin4_1:/pgadmin"   
    environment:
      - PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION=True 
      - PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only!" 
      - PGADMIN_CONFIG_CONSOLE_LOG_LEVEL=10 
    deploy:        
      labels:
        - "traefik.http.routers.pgadmin4_1.rule=Host(`pgadmin.vps1.matho.sk`)"
        - traefik.http.routers.pgadmin4_1.tls.certresolver=le
        - traefik.http.routers.pgadmin4_1.entrypoints=web-secure
        - traefik.http.services.pgadmin4_1.loadbalancer.server.port=80       
        - traefik.docker.network=traefik_db-nw
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager] 
        
  pgadmin4_2:
    image: dpage/pgadmin4:7.8
    env_file: /root/docker-stacks/pgadmin/pgadmin.env 
    networks:
      - traefik_db-nw
    volumes:
      - "/data/pgadmin4_2:/pgadmin"   
    environment:
      - PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION=True 
      - PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only!" 
      - PGADMIN_CONFIG_CONSOLE_LOG_LEVEL=10 
    deploy:        
      labels:
        - "traefik.http.routers.pgadmin4_2.rule=Host(`pgadmin2.vps1.matho.sk`)"
        - traefik.http.routers.pgadmin4_2.tls.certresolver=le
        - traefik.http.routers.pgadmin4_2.entrypoints=web-secure
        - traefik.http.services.pgadmin4_2.loadbalancer.server.port=80       
        - traefik.docker.network=traefik_db-nw
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == worker]         
      
networks:
  traefik_db-nw:
    external: true
```

ENV file is shared for both instances, so PgAdmin credentials for both nodes are equal.

Create Docker volume dir on `ubuntu1`:  
`$ sudo mkdir /data/pgadmin4_1`

On `ubuntu2`  
`$ sudo mkdir /data/pgadmin4_2`

Deploy the Docker stack via:  
`$ sudo docker stack deploy -c pgadmin-stack.yml pgadmin`

Visit `https://pgadmin.vps1.matho.sk` and/or `https://pgadmin2.vps1.matho.sk`  You should be able to login via the credentials you wrote to the `/root/docker-stacks/pgadmin/pgadmin.env ` file.

For the master node: in the Object explorer on the left, do right click and select `Register > Server`. In the modal fill in:
- in General tab:
    - fill in `Master PostgreSQL` to `Name` input
- in Connection tab
    - fill in `spilo_1` to `Host name/address`
    - fill in `5432` for `port`
        - fill in `postgres` for `Maintenance database`
    - fill in `postgres` for `Username`
    - fill in your password for `Password`

For the worker node: in the Object explorer on the left, do right click and select `Register > Server`. In the modal fill in:
- in General tab:
    - fill in `Worker PostgreSQL` to `Name` input
- in Connection tab
    - fill in `spilo_2` to `Host name/address`
    - fill in `5432` for `port`
    - fill in `postgres` for `Maintenance database`
    - fill in `postgres` for `Username`
    - fill in your password for `Password`

Create test database on master and/or worker. It should create. But for now, the databases are not synchronized between nodes yet. You need to install HAProxy to do it.

## 12 Haproxy and PgBouncer
PgBouncer is a connection pooler for PostgreSQL. It reduces performance overhead by rotating client connections to PostgreSQL databases. It supports PostgreSQL databases located on different hosts.

HAProxy is a free, open source high availability solution, providing load balancing and proxying for TCP and HTTP-based applications by spreading requests across multiple servers.

On `ubuntu1` node create folder for Docker stack recipe.
```
$ sudo mkdir /root/docker-stacks/pgbouncer_haproxy
$ cd /root/docker-stacks/pgbouncer_haproxy
$ vim /root/docker-stacks/pgbouncer_haproxy/pgbouncer1.env
```

with content:
```
PGBOUNCER_AUTH_TYPE=md5
POSTGRESQL_USERNAME=postgres
POSTGRESQL_PASSWORD=YOUR_PASSWORD
POSTGRESQL_DATABASE=postgres
POSTGRESQL_HOST=spilo_1
POSTGRESQL_PORT=5432
PGBOUNCER_PORT=7432
PGBOUNCER_BIND_ADDRESS=pgbouncer_1
PGBOUNCER_AUTH_USER=postgres
```
Create env file for second Pgbouncer:  
`$ vim /root/docker-stacks/pgbouncer_haproxy/pgbouncer2.env`

with content:
``` 
PGBOUNCER_AUTH_TYPE=md5
POSTGRESQL_USERNAME=postgres
POSTGRESQL_PASSWORD=YOUR_PASSWORD
POSTGRESQL_DATABASE=postgres
POSTGRESQL_HOST=spilo_2
POSTGRESQL_PORT=5432
PGBOUNCER_PORT=7434
PGBOUNCER_BIND_ADDRESS=pgbouncer_2
PGBOUNCER_AUTH_USER=postgres
```

Create folder for Docker Pgbouncer volume on `ubuntu1`  
`$ sudo mkdir /data/pgbouncer_1`

Create folder for Docker Pgbouncer volume on `ubuntu2`  
`$ sudo mkdir /data/pgbouncer_2`

Create Docker volume for Haproxy on `ubuntu1`   
`$ sudo mkdir /data/haproxy_1`

Create Docker volume for Haproxy on `ubuntu2`  
`$ sudo mkdir /data/haproxy_2`

Insert there following content on `ubuntu1`:  
`$ sudo vim /data/haproxy_1/haproxy.cfg`

```
global
    maxconn 1000

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

resolvers docker
    nameserver dns1 127.0.0.11:53
    resolve_retries 3
    timeout resolve 1s
    timeout retry   1s
    hold other      30s
    hold refused    30s
    hold nx         30s
    hold timeout    30s
    hold valid      10s
    hold obsolete   10s    
    
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres_write
    bind *:7432
    mode            tcp
    option httpchk
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions    
    server-template postgresql_1 1 spilo_1:5432 check port 8008 resolvers docker init-addr last,libc,none
    server-template postgresql_2 1 spilo_2:5432 check port 8008 resolvers docker init-addr last,libc,none 

#listen postgres_read
#    bind *:7433
#    mode            tcp
#    balance leastconn
#    option pgsql-check user postgres
#    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
#    server-template postgresql_1 1 spilo_1:5432 check port 8008 resolvers docker init-addr last,libc,none
#    server-template postgresql_2 1 spilo_2:5432 check port 8008 resolvers docker init-addr last,libc,none
```

Insert there following content on `ubuntu2`:  
`$ sudo vim /data/haproxy_2/haproxy.cfg`

``` 
global
    maxconn 1000

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

resolvers docker
    nameserver dns1 127.0.0.11:53
    resolve_retries 3
    timeout resolve 1s
    timeout retry   1s
    hold other      30s
    hold refused    30s
    hold nx         30s
    hold timeout    30s
    hold valid      10s
    hold obsolete   10s
listen stats
    mode http
    bind *:7001
    stats enable
    stats uri /
    
listen postgres_write
    bind *:7434
    mode            tcp
    option httpchk
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server-template postgresql_2 1 spilo_2:5432 check port 8008 resolvers docker init-addr last,libc,none
    server-template postgresql_1 1 spilo_1:5432 check port 8008 resolvers docker init-addr last,libc,none     

#listen postgres_read
#    bind *:7435
#    mode            tcp
#    balance leastconn
#    option pgsql-check user postgres
#    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
#    server postgresql_1 spilo_1:5432
#    server postgresql_2 spilo_2:5432
```


Create Docker stack file:  
`$ sudo vim ~/docker-stacks/pgbouncer_haproxy/pgbouncer_haproxy-stack.yml`

```yaml
version: '3'

services:
  pgbouncer_1:
    image: bitnami/pgbouncer:1.21.0
    hostname: pgbouncer_1
    env_file: /root/docker-stacks/pgbouncer_haproxy/pgbouncer1.env
    networks:
      - traefik_db-nw
    #ports:
    #  - "7432:5432" 
    volumes:
      - "/data/pgbouncer_1:/etc/pgbouncer" 
    deploy:        
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
        
  pgbouncer_2:
    image: bitnami/pgbouncer:1.21.0
    hostname: pgbouncer_2
    env_file: /root/docker-stacks/pgbouncer_haproxy/pgbouncer2.env
    networks:
      - traefik_db-nw
    #ports:
    #  - "7434:5432"     
    volumes:
      - "/data/pgbouncer_2:/etc/pgbouncer"
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == worker]        

  haproxy_1:
    image: haproxy:2.8.3-alpine 
    hostname: haproxy_1
    #ports:
    #  - "8432:7432"
    #  - "8433:7433" 
    networks:
      - traefik_db-nw
    volumes:
      - "/data/haproxy_1:/usr/local/etc/haproxy"
    deploy:
      labels:
        - "traefik.http.routers.haproxy1.rule=Host(`haproxy.vps1.matho.sk`)"
        - traefik.http.routers.haproxy1.tls.certresolver=le
        - traefik.http.routers.haproxy1.entrypoints=web-secure
        - "traefik.http.routers.haproxy1.middlewares=haproxy1-auth"
        - "traefik.http.middlewares.haproxy1-auth.basicauth.users=admin:PASSWORD_HASH"
        - traefik.http.services.haproxy1.loadbalancer.server.port=7000        
        - traefik.docker.network=traefik_db-nw        
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager] 
        
  haproxy_2:
    image: haproxy:2.8.3-alpine
    hostname: haproxy_2
    #ports:
    #  - "8434:7434"
    #  - "8435:7435" 
    networks:
      - traefik_db-nw
    volumes:
      - "/data/haproxy_2:/usr/local/etc/haproxy"
    deploy:
      labels:
        - "traefik.http.routers.haproxy2.rule=Host(`haproxy2.vps1.matho.sk`)"
        - traefik.http.routers.haproxy2.tls.certresolver=le
        - traefik.http.routers.haproxy2.entrypoints=web-secure
        - "traefik.http.routers.haproxy2.middlewares=haproxy2-auth"
        - "traefik.http.middlewares.haproxy2-auth.basicauth.users=admin:PASSWORD_HASH"
        - traefik.http.services.haproxy2.loadbalancer.server.port=7001
        - traefik.docker.network=traefik_db-nw
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == worker]

networks:
  traefik_db-nw:
    external: true
```

You need to generate custom password hash. Visit the page https://hostingcanada.org/htpasswd-generator/ . Insert username and password, select apache specific format and copy the credentials to Traefik label.

`Note:` escape dolar symbols. For each dolar, prepend another one dolar.

Deploy the Docker stack via:  
`$ sudo docker stack deploy -c ~/docker-stacks/pgbouncer_haproxy/pgbouncer_haproxy-stack.yml pgbouncer_haproxy`

Once you have migrated the RoR projects, you will have some project on ubuntu1 and also on ubuntu2 node. You can play with Spilo - turn off Spilo_1, check if websites on both nodes works, then turn on Spilo_1 and turn off Spilo_2, check if websites on both nodes works. Alternatively, you can see the status in the Haproxy stats website and you can approve, that Haproxy communication across our cluster works.

## 13 Ruby on Rails projects - reimport

I have few web projects written in Ruby on Rails framework. I want to migrate it to this new architecture.

Create Docker stack file:  
`$ sudo vim /root/docker-stacks/rails_projects/rails_projects-stack.yml`

```yaml
version: '3'

services:
  mathosk:
    image: registry.vps1.matho.sk/mathosk:latest
    hostname: mathosk
    env_file: /root/docker-stacks/rails_projects/mathosk.env
    networks:
      - traefik_db-nw 
    volumes:
      - "/storage-pool/data/mathosk:/app/shared" 
    deploy:
      labels:
        - traefik.http.middlewares.mathosk-redirect-web-secure.redirectscheme.scheme=https
        - traefik.http.routers.mathosk.middlewares=mathosk-redirect-web-secure
        - traefik.http.routers.mathosk.rule=Host(`contabo.matho.sk`)
        - traefik.http.routers.mathosk.entrypoints=web
        - traefik.http.routers.mathosk-secure.rule=Host(`contabo.matho.sk`)
        - traefik.http.routers.mathosk-secure.tls.certresolver=le
        - traefik.http.routers.mathosk-secure.entrypoints=web-secure
        - traefik.http.services.mathosk-secure.loadbalancer.server.port=80
        - traefik.docker.network=traefik_db-nw
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == worker]

networks:
  traefik_db-nw:
    external: true
```

Modify DNS A -> point to Contabo VPS IP.

Create ENV file:  
`$ sudo vim /root/docker-stacks/rails_projects/mathosk.env`

with content (provide your own ENV for password)
``` 
SMTP_ADDRESS=
SMTP_DOMAIN=matho.sk
SMTP_USERNAME=no-reply@matho.sk
SMTP_PASSWORD=
POSTGRES_HOST=haproxy_2
POSTGRES_DB=matho_production
POSTGRES_USER=postgres
POSTGRES_PASSWORD=
POSTGRES_PORT=7434
RAILS_WORKERS=2
RAILS_MIN_THREADS=5
RAILS_MAX_THREADS=5
RAILS_ENV=production
```

Create new user for `POSTGRES_USER` with strong password `POSTGRES_PASSWORD` via PgAdmin. Priviledges keep the default.

Create new Gluster folder for shared public assets for Refinery CMS.   
`$ sudo mkdir -p /storage-pool/data/mathosk`

Rewrite the base Docker image for the project. Previously I was using Raspberry PI witch `arm64` architecture, so I needed to rebuild Docker image to `amd64`
Do build via:  
`$ sudo docker build -t registry.vps1.matho.sk/rpi-ruby-2.2-ubuntu-amd64:latest  .`

Then login and push the image to registry:
```
$ docker login registry.vps1.matho.sk
$ docker push registry.vps1.matho.sk/rpi-ruby-2.2-ubuntu-amd64:latest
```

Also, you can build the docker image on your localhost:  
`$ sudo docker build --network=host --build-arg RAILS_ENV=production -t registry.vps1.matho.sk/mathosk:latest  .`
Note the `--network=host` argument. It allows connection to your local Postgres database, when doing asset precompilation.

After the docker push, login to the `ubuntu2` node and pull the image. Do restart container via Portainer.

If you need to migrate the old Postgres database from IP 10.0.2.4, create empty `matho_production` database. Then you can trigger the restore from old server to new server via:

``` 
pg_dump --no-owner --no-privileges --dbname=postgresql://postgres:PASSWORD@10.0.2.4:5432/matho_production | psql "postgresql://matho_production:PASSWORD@vps2.matho.sk:5433/matho_production"
```

You can deploy the new docker stack via:  
`$ sudo docker stack deploy -c rails_projects-stack.yml rails_projects`

One more step there - we need to synch uploaded images for the website. We will synch it via rsync from the old cluster to this new cluster.
```
$ sudo rsync -lptrucv -e "ssh -p 7777" --progress /storage-pool/data/mathosk/ root@vps2.matho.sk:/storage-pool/data/mathosk
```

### 13.1 Ruby on Rails projects - scram auth problem
To be able successfully authenticate to Spilo's v15 Postgres, you need at least v10 of Postgres client in the Rails project.
Because I had issues with building Ruby on the Ubuntu 16.04+ I decided to upgrade Postgres in the Rails project Docker image.
To do it, specify the changes in Dockerfile via this way:

```Dockerfile
#... beginning of Dockerfile 16.04 is here ...
 
# Update the package lists:
RUN apt-get update && apt-get install -y \
    wget \
    curl \
    vim \
    git \
    build-essential \
    libgmp-dev \
    locales \
    nginx \
    cron \
    bash \
    imagemagick \
    apt-transport-https \
    ca-certificates

# Create the file repository configuration:
RUN sh -c 'echo "deb https://apt-archive.postgresql.org/pub/repos/apt xenial-pgdg-archive main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key:
RUN wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -

RUN apt-get update && apt-get install -y \
    libpq-dev \
    postgresql-client-12
    
# ....    
```
With this change, we have added Postgres PPA to Ubuntu 16.04 based Docker image and we installed the Postgres v12 client packaget with libpq-dev package. After rebuild and re-push, you should be able to see your Rails project website.

## 14 Postgres backuping
We are running in HA, so if one of the node will became corrupted, we have second node with the current postgres snapshot. But, what if something breaks, and both nodes will be corrupted? What if we will need to restore some older version of Postgres backup?

So, I have implemented backuping on daily base. The script will backup the databases to separate file, and at the specified day. The backups will be rotated on weekly base.

Because my Postgres cluster size is small - the backups has ~ 4MB, so I decided to use `pg_dump` to take snapshots of the database. I'm doing the backup on both nodes, and files are then stored to Gluster, which could be geo-replicated to another server.

Check, if pgdump works
```
POSTGRES_BACKUP_DOCKER_CONTAINER_NAME=$(docker container ls -a | grep 'spilo_1' | grep '5432/tcp' | awk '{ print $1 }' | head -n 1)
docker exec -e PGPASSWORD=YOUR-PASSWORD $POSTGRES_BACKUP_DOCKER_CONTAINER_NAME pg_dumpall -h spilo_1 -U postgres > 2023_11_17_backup.sql
```

We are using docker solution here, because we do not want to expose 5432 ports to host OS. We are connecting to the spilo docker container instead of to exposed port.

Backuping was implemented base on this article [https://wiki.postgresql.org/wiki/Automated_Backup_on_Linux](https://wiki.postgresql.org/wiki/Automated_Backup_on_Linux)

On ubuntu1:
```
$ sudo mkdir /storage-pool/spilo_1_backups
$ sudo mkdir /storage-pool/spilo_2_backups
```

On both nodes
``` 
$ sudo mkdir ~/pg_backuping
$ sudo vim ~/pg_backuping/pg_backup.config
```

On ubuntu1 (for ubuntu2 specify different hostname and backup dir)
```bash
##############################
## POSTGRESQL BACKUP CONFIG ##
##############################

# Optional system user to run backups as.  If the user the script is running as doesn't match this
# the script terminates.  Leave blank to skip check.
BACKUP_USER=root

# Optional hostname to adhere to pg_hba policies.  Will default to "localhost" if none specified.
HOSTNAME=localhost

# Optional username to connect to database as.  Will default to "postgres" if none specified.
USERNAME=postgres

PASSWORD=YOUR-PASSWORD

# This dir will be created if it doesn't exist.  This must be writable by the user the script is
# running as.
BACKUP_DIR=/storage-pool/spilo_1_backups/

# List of strings to match against in database name, separated by space or comma, for which we only
# wish to keep a backup of the schema, not the data. Any database names which contain any of these
# values will be considered candidates. (e.g. "system_log" will match "dev_system_log_2010-01")
SCHEMA_ONLY_LIST=""

# Will produce a custom-format backup if set to "yes"
ENABLE_CUSTOM_BACKUPS=yes

# Will produce a gzipped plain-format backup if set to "yes"
ENABLE_PLAIN_BACKUPS=yes

# Will produce gzipped sql file containing the cluster globals, like users and passwords, if set to "yes"
ENABLE_GLOBALS_BACKUPS=yes


#### SETTINGS FOR ROTATED BACKUPS ####

# Which day to take the weekly backup from (1-7 = Monday-Sunday)
DAY_OF_WEEK_TO_KEEP=6

# Number of days to keep daily backups
DAYS_TO_KEEP=7

# How many weeks to keep weekly backups
WEEKS_TO_KEEP=5

######################################
```

`$ sudo vim  ~/pg_backuping/pg_backup_rotated.sh`

```bash
#!/bin/bash

###########################
####### LOAD CONFIG #######
###########################

while [ $# -gt 0 ]; do
        case $1 in
                -c)
                        CONFIG_FILE_PATH="$2"
                        shift 2
                        ;;
                *)
                        ${ECHO} "Unknown Option \"$1\"" 1>&2
                        exit 2
                        ;;
        esac
done

if [ -z $CONFIG_FILE_PATH ] ; then
        SCRIPTPATH=$(cd ${0%/*} && pwd -P)
        CONFIG_FILE_PATH="${SCRIPTPATH}/pg_backup.config"
fi

if [ ! -r ${CONFIG_FILE_PATH} ] ; then
        echo "Could not load config file from ${CONFIG_FILE_PATH}" 1>&2
        exit 1
fi

source "${CONFIG_FILE_PATH}"

###########################
#### PRE-BACKUP CHECKS ####
###########################

# Make sure we're running as the required backup user
if [ "$BACKUP_USER" != "" -a "$(id -un)" != "$BACKUP_USER" ] ; then
	echo "This script must be run as $BACKUP_USER. Exiting." 1>&2
	exit 1
fi


###########################
### INITIALISE DEFAULTS ###
###########################


if [ ! $HOSTNAME ]; then
	HOSTNAME="localhost"
fi;

if [ ! $USERNAME ]; then
	USERNAME="postgres"
fi;

###########################
#### START THE BACKUPS ####
###########################

function perform_backups()
{
	
	POSTGRES_BACKUP_DOCKER_CONTAINER_NAME=`docker container ls -a | grep 'spilo_1' | grep '5432/tcp' | awk '{ print $1 }' | head -n 1`
	SUFFIX=$1
	FINAL_BACKUP_DIR=$BACKUP_DIR"`date +\%Y-\%m-\%d`$SUFFIX/"

	echo "Making backup directory in $FINAL_BACKUP_DIR"

	if ! mkdir -p $FINAL_BACKUP_DIR; then
		echo "Cannot create backup directory in $FINAL_BACKUP_DIR. Go and fix it!" 1>&2
		exit 1;
	fi;
	
	#######################
	### GLOBALS BACKUPS ###
	#######################

	echo -e "\n\nPerforming globals backup"
	echo -e "--------------------------------------------\n"

	if [ $ENABLE_GLOBALS_BACKUPS = "yes" ]
	then
		    echo "Globals backup"

		    set -o pipefail
		    if ! docker exec -e PGPASSWORD=$PASSWORD $POSTGRES_BACKUP_DOCKER_CONTAINER_NAME pg_dumpall -g -h "$HOSTNAME" -U "$USERNAME" | gzip > $FINAL_BACKUP_DIR"globals".sql.gz.in_progress; then
		            echo "[!!ERROR!!] Failed to produce globals backup" 1>&2
		    else
		            mv $FINAL_BACKUP_DIR"globals".sql.gz.in_progress $FINAL_BACKUP_DIR"globals".sql.gz
		    fi
		    set +o pipefail
	else
		echo "None"
	fi


	###########################
	### SCHEMA-ONLY BACKUPS ###
	###########################
	
	for SCHEMA_ONLY_DB in ${SCHEMA_ONLY_LIST//,/ }
	do
	        SCHEMA_ONLY_CLAUSE="$SCHEMA_ONLY_CLAUSE or datname ~ '$SCHEMA_ONLY_DB'"
	done
	
	SCHEMA_ONLY_QUERY="select datname from pg_database where false $SCHEMA_ONLY_CLAUSE order by datname;"
	
	echo -e "\n\nPerforming schema-only backups"
	echo -e "--------------------------------------------\n"
	
	SCHEMA_ONLY_DB_LIST=`docker exec -e PGPASSWORD=$PASSWORD $POSTGRES_BACKUP_DOCKER_CONTAINER_NAME psql -h "$HOSTNAME" -U "$USERNAME" -At -c "$SCHEMA_ONLY_QUERY" postgres`
	
	echo -e "The following databases were matched for schema-only backup:\n${SCHEMA_ONLY_DB_LIST}\n"
	
	for DATABASE in $SCHEMA_ONLY_DB_LIST
	do
	        echo "Schema-only backup of $DATABASE"
		set -o pipefail
	        if ! docker exec -e PGPASSWORD=$PASSWORD $POSTGRES_BACKUP_DOCKER_CONTAINER_NAME pg_dump -Fp -s -h "$HOSTNAME" -U "$USERNAME" "$DATABASE" | gzip > $FINAL_BACKUP_DIR"$DATABASE"_SCHEMA.sql.gz.in_progress; then
	                echo "[!!ERROR!!] Failed to backup database schema of $DATABASE" 1>&2
	        else
	                mv $FINAL_BACKUP_DIR"$DATABASE"_SCHEMA.sql.gz.in_progress $FINAL_BACKUP_DIR"$DATABASE"_SCHEMA.sql.gz
	        fi
	        set +o pipefail
	done
	
	
	###########################
	###### FULL BACKUPS #######
	###########################

	for SCHEMA_ONLY_DB in ${SCHEMA_ONLY_LIST//,/ }
	do
		EXCLUDE_SCHEMA_ONLY_CLAUSE="$EXCLUDE_SCHEMA_ONLY_CLAUSE and datname !~ '$SCHEMA_ONLY_DB'"
	done

	FULL_BACKUP_QUERY="select datname from pg_database where not datistemplate and datallowconn $EXCLUDE_SCHEMA_ONLY_CLAUSE order by datname;"

	echo -e "\n\nPerforming full backups"
	echo -e "--------------------------------------------\n"

	for DATABASE in `docker exec -e PGPASSWORD=$PASSWORD $POSTGRES_BACKUP_DOCKER_CONTAINER_NAME psql -h "$HOSTNAME" -U "$USERNAME" -At -c "$FULL_BACKUP_QUERY" postgres`
	do
		if [ $ENABLE_PLAIN_BACKUPS = "yes" ]
		then
			echo "Plain backup of $DATABASE"
	 
			set -o pipefail
			if ! docker exec -e PGPASSWORD=$PASSWORD $POSTGRES_BACKUP_DOCKER_CONTAINER_NAME pg_dump -Fp -h "$HOSTNAME" -U "$USERNAME" "$DATABASE" | gzip > $FINAL_BACKUP_DIR"$DATABASE".sql.gz.in_progress; then
				echo "[!!ERROR!!] Failed to produce plain backup database $DATABASE" 1>&2
			else
				mv $FINAL_BACKUP_DIR"$DATABASE".sql.gz.in_progress $FINAL_BACKUP_DIR"$DATABASE".sql.gz
			fi
			set +o pipefail
                        
		fi

		if [ $ENABLE_CUSTOM_BACKUPS = "yes" ]
		then
			echo "Custom backup of $DATABASE"
	
			if ! docker exec -e PGPASSWORD=$PASSWORD $POSTGRES_BACKUP_DOCKER_CONTAINER_NAME pg_dump -Fc -h "$HOSTNAME" -U "$USERNAME" "$DATABASE" > $FINAL_BACKUP_DIR"$DATABASE".custom.in_progress; then
				echo "[!!ERROR!!] Failed to produce custom backup database $DATABASE"
			else
				mv $FINAL_BACKUP_DIR"$DATABASE".custom.in_progress $FINAL_BACKUP_DIR"$DATABASE".custom
			fi
		fi

	done

	echo -e "\nAll database backups complete!"
}

# MONTHLY BACKUPS

DAY_OF_MONTH=`date +%d`

if [ $DAY_OF_MONTH -eq 1 ];
then
	# Delete all expired monthly directories
	find $BACKUP_DIR -maxdepth 1 -name "*-monthly" -exec rm -rf '{}' ';'
	        	
	perform_backups "-monthly"
	
	exit 0;
fi

# WEEKLY BACKUPS

DAY_OF_WEEK=`date +%u` #1-7 (Monday-Sunday)
EXPIRED_DAYS=`expr $((($WEEKS_TO_KEEP * 7) + 1))`

if [ $DAY_OF_WEEK = $DAY_OF_WEEK_TO_KEEP ];
then
	# Delete all expired weekly directories
	find $BACKUP_DIR -maxdepth 1 -mtime +$EXPIRED_DAYS -name "*-weekly" -exec rm -rf '{}' ';'
	        	
	perform_backups "-weekly"
	
	exit 0;
fi

# DAILY BACKUPS

# Delete daily backups 7 days old or more
find $BACKUP_DIR -maxdepth 1 -mtime +$DAYS_TO_KEEP -name "*-daily" -exec rm -rf '{}' ';'

perform_backups "-daily"
```

Test the bash script via:
```
$ chmod +x pg_backup_rotated.sh
$ sudo ./pg_backup_rotated.sh
```

If works, setup crontab `$ sudo crontab -e` with content:
`0 2 * * * /bin/bash /root/pg_backuping/pg_backup_rotated.sh`

Repeat on `ubuntu2` with spilo_2 in my ENV var.

## 15 Issues after server restarts
Sometimes the server do not boot correctly.

If you have issues with Gluster and you do not see output of `ls -la /storage-pool` after reboot, you probably need to mount the volume.

If the problem is on `gluster1` node, use:  
`$ sudo mount -t glusterfs gluster1.example.com:/volume1 /storage-pool`

If the problem is on `gluster2` node, use:  
`$ sudo mount -t glusterfs gluster2.example.com:/volume1 /storage-pool`

If you have issues with Portainer, solution could be:  
`$ sudo docker service update portainer_agent --force`

## 16 Summary
In this tutorial we described how to install Docker Swarm, GlusterFS, Traefik, Postgres in HA, Haproxy, Portainer and Zabbix monitoring on two virtual private servers. 