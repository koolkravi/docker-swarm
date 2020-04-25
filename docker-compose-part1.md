# Docker swarm

## A. Set up
you need following 
# Two Create Virtual Machine with linux host
# install docker 
# The IP address of the manager machine
# Open protocols and ports between the hosts

# Step1: 2 Create Virtual Machine with linux host

Run below command. it willcreate 2 VM
ubuntuvm01 - master node
ubuntuvm02 - worker node

```
vagrant up
```

# step 2: install docker 
run script from below link on bothe VM
https://github.com/koolkravi/kubernetes-playground/blob/master/kubernetes-with-kind/install-docker.sh

```
install-docker.sh
```
{
root@ubuntuvm01:/home/vagrant# docker --version
Docker version 19.03.8, build afacb8b7f0

root@ubuntuvm02:/home/vagrant# docker --version
Docker version 19.03.8, build afacb8b7f0

}

# Step3: The IP address of the manager machine
The IP address must be assigned to a network interface available to the host operating system

```
ifconfig 
```

- ubuntuvm01 - master node - 10.0.0.5
- ubuntuvm02 - worker node - 10.0.0.6

# Step4: Open protocols and ports between the hosts
The following ports must be available. On some systems, these ports are open by default

- TCP port 2377 for cluster management communications
- TCP and UDP port 7946 for communication among nodes
- UDP port 4789 for overlay network traffic
- If you plan on creating an overlay network with encryption (--opt encrypted), you also need to ensure ip protocol 50 (ESP) traffic is allowed.

- check is port is open
```

nc -z 127.0.0.0 2377; echo $?
nc -z 127.0.0.0 7946; echo $?
nc -z 127.0.0.0 4789; echo $?

this returns "0" if the port is open and "1" if the port is closed
```

- open port
```
sudo ufw allow 2377/tcp
sudo ufw allow 7946/tcp
sudo ufw allow 4789/tcp
```
- sudo ufw status
- sudo ufw enable

## B. Create swarm

# Step 1: manager node

```
vagrant ssh ubuntuvm01
```

# Step 2: Following command create swarm

```
docker swarm init --advertise-addr 10.0.0.5

output:
Swarm initialized: current node (jiabm1qni66jywectswob4mtc) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1vqdwkeqtgwd0q20stycqbhfsv0lyg6007je9w7ozvihzghtw8-a6psq3f1xxwmdjxs89qm0j1vy 10.0.0.5:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

# Step 3: 
```
docker info
output:
Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
.
.
Swarm: active
NodeID: jiabm1qni66jywectswob4mtc
Is Manager: true
ClusterID: rhifj0nn542imrerrz6ymvak5
Managers: 1
Nodes: 1
.
.

```

# Step 4: 

```
root@ubuntuvm01:/home/vagrant# docker node ls

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
jiabm1qni66jywectswob4mtc *   ubuntuvm01          Ready               Active              Leader              19.03.8

```

## C. Add one more node to swarm
# Step 1: from worker nodes

```
vagrant ssh ubuntuvm02
``
# Step 2: 

```
docker swarm join --token SWMTKN-1-1vqdwkeqtgwd0q20stycqbhfsv0lyg6007je9w7ozvihzghtw8-a6psq3f1xxwmdjxs89qm0j1vy 10.0.0.5:2377

This node joined a swarm as a worker.

```

- Retrieve Join command

```
docker swarm join-token worker
```

- Repeate above to join mmore worker node to cluster

## D. Deploy a Service to swarm
# Step 1: from manager node

```
 docker service create --replicas 1 --name hellowworld alpine ping docker.com
```
# Step 2: from manager node

```
root@ubuntuvm01:/home/vagrant# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
vfu7dcgcj0ld        hellowworld         replicated          1/1                 alpine:latest

```
# Step 3: inspect the service
```
root@ubuntuvm01:/home/vagrant# docker service inspect --pretty hellowworld

ID:             vfu7dcgcj0ldc6wfew0mj8pdz
Name:           hellowworld
Service Mode:   Replicated
 Replicas:      1
Placement:
UpdateConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         alpine:latest@sha256:9a839e63dad54c3a6d1834e29692c8492d93f90c59c978c1ed79109ea4fb9a54
 Args:          ping docker.com
 Init:          false
Resources:
Endpoint Mode:  vip
```

```
root@ubuntuvm01:/home/vagrant# docker service inspect hellowworld
```

# step 4: Check which node is running the service

```
root@ubuntuvm01:/home/vagrant# docker service ps hellowworld
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
z08pkp3lpyi8        hellowworld.1       alpine:latest       ubuntuvm01          Running             Running 10 minutes ago
```

# Step 4: To See details about the container ( run where task is running)

```
root@ubuntuvm01:/home/vagrant# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
6408e7d8a3a0        alpine:latest       "ping docker.com"   13 minutes ago      Up 13 minutes                           hellowworld.1.z08pkp3lpyi8gqww2dgq1ffex

```
# Step 5: scale

```
root@ubuntuvm01:/home/vagrant# docker service scale hellowworld=5
hellowworld scaled to 5
overall progress: 5 out of 5 tasks
1/5: running   [==================================================>]
2/5: running   [==================================================>]
3/5: running   [==================================================>]
4/5: running   [==================================================>]
5/5: running   [==================================================>]
verify: Service converged

```

- Containers running in a service are called “tasks.”

```
root@ubuntuvm01:/home/vagrant# docker service ps hellowworld
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
z08pkp3lpyi8        hellowworld.1       alpine:latest       ubuntuvm01          Running             Running 19 minutes ago
qy5ktskdi7vy        hellowworld.2       alpine:latest       ubuntuvm02          Running             Running about a minute ago
n8s2zw0s0b5b        hellowworld.3       alpine:latest       ubuntuvm02          Running             Running about a minute ago
kmi2xf6nare7        hellowworld.4       alpine:latest       ubuntuvm02          Running             Running about a minute ago
qw372ed93kk8        hellowworld.5       alpine:latest       ubuntuvm01          Running             Running about a minute ago
```

```
root@ubuntuvm01:/home/vagrant# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
95215a941cc8        alpine:latest       "ping docker.com"   3 minutes ago       Up 3 minutes                            hellowworld.5.qw372ed93kk8nitum6swtmsyj
6408e7d8a3a0        alpine:latest       "ping docker.com"   21 minutes ago      Up 21 minutes                           hellowworld.1.z08pkp3lpyi8gqww2dgq1ffex
```

```
root@ubuntuvm02:/home/vagrant# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
4d80fa37c1ba        alpine:latest       "ping docker.com"   3 minutes ago       Up 3 minutes                            hellowworld.3.n8s2zw0s0b5bro7gnld36tnuv
0fb02987e569        alpine:latest       "ping docker.com"   3 minutes ago       Up 3 minutes                            hellowworld.4.kmi2xf6nare795qfhxo5w2hsk
59b6ee594304        alpine:latest       "ping docker.com"   3 minutes ago       Up 3 minutes                            hellowworld.2.qy5ktskdi7vy5sda6lm9pqezo
```


# Step 5: delete the service running on swarm
```
root@ubuntuvm01:/home/vagrant# docker service rm hellowworld
hellowworld

```

```
verify 

root@ubuntuvm01:/home/vagrant# docker service inspect hellowworld
[]
Status: Error: no such service: hellowworld, Code: 1
```

```
containers take few seconds to clean up
docker ps
```

## Step 6: Rolling update

```
root@ubuntuvm01:/home/vagrant# docker service create --replicas 3 --name redis --update-delay 10s redis:3.0.6

root@ubuntuvm01:/home/vagrant# docker service inspect --pretty redis

ID:             6ncwb3ay2qig3snhptqi5f6p4
Name:           redis
Service Mode:   Replicated
 Replicas:      3
Placement:
UpdateConfig:
 Parallelism:   1
 Delay:         10s
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         redis:3.0.6@sha256:6a692a76c2081888b589e26e6ec835743119fe453d67ecf03df7de5b73d69842
 Init:          false
Resources:
Endpoint Mode:  vip
```

```
update
root@ubuntuvm01:/home/vagrant# docker service update --image redis:3.0.7 redis

root@ubuntuvm01:/home/vagrant# docker service inspect --pretty redis

ID:             6ncwb3ay2qig3snhptqi5f6p4
Name:           redis
Service Mode:   Replicated
 Replicas:      3
UpdateStatus:
 State:         completed
 Started:       About a minute ago
 Completed:     28 seconds ago
 Message:       update completed
Placement:
UpdateConfig:
 Parallelism:   1
 Delay:         10s
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         redis:3.0.7@sha256:730b765df9fe96af414da64a2b67f3a5f70b8fd13a31e5096fee4807ed802e20
 Init:          false
Resources:
Endpoint Mode:  vip
```

```
to start paused service

docker service update redis
```

```
Watch rolling update

root@ubuntuvm01:/home/vagrant# docker service ps redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
m8xhskhous40        redis.1             redis:3.0.7         ubuntuvm01          Running             Running 3 minutes ago
nvijt2zjz14k         \_ redis.1         redis:3.0.6         ubuntuvm01          Shutdown            Shutdown 3 minutes ago
mxjdmdwi84n7        redis.2             redis:3.0.7         ubuntuvm02          Running             Running 3 minutes ago
e7k4blc5yov6         \_ redis.2         redis:3.0.6         ubuntuvm02          Shutdown            Shutdown 4 minutes ago
saibwosepijc        redis.3             redis:3.0.7         ubuntuvm01          Running             Running 3 minutes ago
vxsbvz0h8are         \_ redis.3         redis:3.0.6         ubuntuvm01          Shutdown            Shutdown 3 minutes ago

```

## E. Drain node on swarm

# Step 1: 
```
root@ubuntuvm01:/home/vagrant# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
jiabm1qni66jywectswob4mtc *   ubuntuvm01          Ready               Active              Leader              19.03.8
op0i0d7tbjo7vmx9vm8rwe71c     ubuntuvm02          Ready               Active                                  19.03.8

```

```
root@ubuntuvm01:/home/vagrant# docker service  ps redis | grep Running
m8xhskhous40        redis.1             redis:3.0.7         ubuntuvm01          Running             Running 17 minutes ago
mxjdmdwi84n7        redis.2             redis:3.0.7         ubuntuvm02          Running             Running 17 minutes ago
saibwosepijc        redis.3             redis:3.0.7         ubuntuvm01          Running             Running 16 minutes ago

```

```
Drain node 

root@ubuntuvm01:/home/vagrant# docker node update --availability drain ubuntuvm02
ubuntuvm02

```

```
root@ubuntuvm01:/home/vagrant# docker node inspect --pretty ubuntuvm02
ID:                     op0i0d7tbjo7vmx9vm8rwe71c
Hostname:               ubuntuvm02
Joined at:              2020-04-25 07:53:31.245406517 +0000 utc
Status:
 State:                 Ready
 Availability:          Drain
.
.

```

```
root@ubuntuvm01:/home/vagrant# docker service ps redis | grep Running
m8xhskhous40        redis.1             redis:3.0.7         ubuntuvm01          Running             Running 50 minutes ago
2dss7ioa4tg4        redis.2             redis:3.0.7         ubuntuvm01          Running             Running 10 minutes ago
saibwosepijc        redis.3             redis:3.0.7         ubuntuvm01          Running             Running 49 minutes ago
```

- The swarm manager maintains the desired state by ending the task on a node with Drain availability and creating a new task on a node with Active availability.

```
set the node back to Active availability

root@ubuntuvm01:/home/vagrant# docker node update --availability active ubuntuvm02
ubuntuvm02

root@ubuntuvm01:/home/vagrant# docker node inspect --pretty ubuntuvm02
ID:                     op0i0d7tbjo7vmx9vm8rwe71c
Hostname:               ubuntuvm02
Joined at:              2020-04-25 07:53:31.245406517 +0000 utc
Status:
 State:                 Ready
 Availability:          Active
.
.
```

## F. routing mesh (ingress network in the swarm)
Docker Engine swarm mode makes it easy to publish ports for services to make them available to resources outside the swarm

To use the ingress network in the swarm, following port should be open between swarm nodes
* Port 7946 TCP/UDP for container network discovery.
* Port 4789 UDP for the container ingress network.

# Step 1: Publish a port for a service

```
root@ubuntuvm01:/home/vagrant# docker service create --name my-web --replicas 2 --publish published=8080,target=80 nginx
```

```
docker service update --publish-add published=<PUBLISHED-PORT>,target=<CONTAINER-PORT> <SERVICE>

root@ubuntuvm01:/home/vagrant# docker service inspect --format="{{json .Endpoint.Spec.Ports}}" my-web
[{"Protocol":"tcp","TargetPort":80,"PublishedPort":8080,"PublishMode":"ingress"}]

```

```
publ tcp and udp only // Specify protocol
docker service create --name dns-cache --publish published=53,target=53 --publish published=53,target=53,protocol=udp dns-cache
 or
docker service create --name dns-cache -p 53:53 -p 53:53/udp dns-cache 

publish udp only // 
docker service create --name dns-cache --publish published=53,target=53,protocol=udp  dns-cache
or
docker service create --name dns-cache   -p 53:53/udp dns-cache
```

## G. Bypass the routing mesh
- You can bypass the routing mesh, so that when you access the bound port on a given node, you are always accessing the instance of the service running on that node. This is referred to as host mode
- use host mode
- only a single instance of the service runs on a given node, by using a global service rather than a replicated one, or by using placement constraints.
- To bypass the routing mesh, you must use the long --publish service and set mode to host. If you omit the mode key or set it to ingress, the routing mesh is used

```
docker service create --name redis --publish published=8081,target=6379,protocol=udp,mode=host --mode global redis:3.0.7
```

```
root@ubuntuvm01:/home/vagrant# docker ps | grep redis
19fc729246c0        redis:3.0.7         "docker-entrypoint.s…"   About a minute ago   Up About a minute   6379/tcp, 0.0.0.0:8081->6379/udp   redis.jiabm1qni66jywectswob4mtc.jl8szeuz0c08kk4amnyymg0um

root@ubuntuvm02:/home/vagrant# docker ps | grep redis
564577b64a65        redis:3.0.7         "docker-entrypoint.s…"   About a minute ago   Up About a minute   6379/tcp, 0.0.0.0:8081->6379/udp   redis.op0i0d7tbjo7vmx9vm8rwe71c.yvf11habgyvbmfpgwtvt6rwrt
```
```
root@ubuntuvm01:/home/vagrant# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
gikathg5vx1w        my-web              replicated          4/4                 nginx:latest        *:8080->80/tcp
h06on8q7m0gd        redis               global              2/2                 redis:3.0.7
```

## H. Configure an external load balancer

External load balancer for swarm services can be configured either in combination with the routing mesh or without using the routing mesh at all.

## Step 1: Using the routing mesh

HAProxy Read : https://cbonte.github.io/haproxy-dconv/

## Step 2: Without the routing mesh
To use an external load balancer without the routing mesh, set --endpoint-mode to dnsrr instead of the default value of vip
- Configure service discovery.: https://docs.docker.com/engine/swarm/networking/#configure-service-discovery

## command Summary 
```
docker node ls
docker service ls
docker service ps redis
docker inspect redis
docker inspect --pretty redis
docker ps
```


- ref 
- https://docs.docker.com/engine/swarm/swarm-tutorial/
- docker engine overview : https://docs.docker.com/engine/
- How to Open/Allow incoming firewall port on Ubuntu 18.04 Bionic Beaver Linux
- https://linuxconfig.org/how-to-open-allow-incoming-firewall-port-on-ubuntu-18-04-bionic-beaver-linux
- https://www.cyberciti.biz/faq/how-to-open-firewall-port-on-ubuntu-linux-12-04-14-04-lts/

# Further reading in detail - Deploy services to a swarm
* https://docs.docker.com/engine/swarm/services/

