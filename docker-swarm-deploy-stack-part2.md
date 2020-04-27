# Deploy a stack to a swarm

"docker stack deploy" command is used to deploy a complete application stack to the swarm.
The deploy command accepts a stack description in the form of a Compose file.
docker stack and docker service commands must be run from a manager node
The docker stack deploy command supports any Compose file of version “3.0” or above

## Pre-requisite
- Docker Engine of version 1.13.0 or later

````
[install-docker.sh](https://github.com/koolkravi/kubernetes-playground/blob/master/kubernetes-with-kind/install-docker.sh)
```

- Docker Compose version 1.10 or later

```
install-docker-compose.sh
```

```
- docker-compose version 1.25.5, build 8a1c60f6

uinstall 
sudo rm /usr/local/bin/docker-compose
```

- Setup swarm Manager and workder node
[Execute Step B and Step C] (https://github.com/koolkravi/docker-swarm/blob/master/docker-compose-part1.md)

## Step 1 : Set up a docker registry
swarm consists of multiple Docker Engines, a registry is required to distribute images to all of them.
we cam use dockerhub or others.

### create a throwaway registry 
* Start the registry as a service on your swarm

```
root@ubuntuvm01:/home/vagrant# docker service create --name registry --publish published=5000,target=5000 registry:2
```

```
root@ubuntuvm01:/home/vagrant# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
p0swvjy91gkp        registry            replicated          1/1                 registry:2          *:5000->5000/tcp

```

```
Check
root@ubuntuvm01:/home/vagrant# curl http://10.0.0.5:5000/v2
<a href="/v2/">Moved Permanently</a>.

```

## Step 2: Create a  example application - python Hit counter app 

### 1. 
```
mkdir stackdemo
cd stackdemo
```
### 2. create a file - app.py
Content of file as below

```
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host='redis', port=6379)

@app.route('/')
def hello():
    count = redis.incr('hits')
    return 'Hello World! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000, debug=True)
```

### 3. Create a file - requirements.txt
Content of file
```
flask
redis
```
### 4. Create a file - Dockerfile 

contet
```
FROM python:3.4-alpine
ADD . /code
WORKDIR /code
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

### 5. Create a file - docker-compose.yml
content
```
version: '3'

services:
  web:
    image: 127.0.0.1:5000/stackdemo
    build: .
    ports:
      - "8000:8000"
  redis:
    image: redis:alpine
``` 

- image for the web app is build usig docker file . It is tagged with 127.0.0.1:5000 - the address of the registry.


## Step 3: Test the app with docker compose

### 1. 
```
root@ubuntuvm01:/vagrant/stackdemo# docker-compose up -d
WARNING: The Docker Engine you're using is running in swarm mode.

Compose does not use swarm mode to deploy services to multiple nodes in a swarm. All containers will be scheduled on the current node.

To deploy your application across the swarm, use `docker stack deploy`.

Creating network "stackdemo_default" with the default driver
Building web
```
- You see a warning about the Engine being in swarm mode. This is because Compose doesn’t take advantage of swarm mode, and deploys everything to a single node

### 2. check if app is running
```
root@ubuntuvm01:/vagrant/stackdemo# docker-compose ps
      Name                     Command               State           Ports
-----------------------------------------------------------------------------------
stackdemo_redis_1   docker-entrypoint.sh redis ...   Up      6379/tcp
stackdemo_web_1     python app.py                    Up      0.0.0.0:8000->8000/tcp
```

Test
```
root@ubuntuvm01:/vagrant/stackdemo# curl http://10.0.0.5:8000
Hello World! I have been seen 1 times.
root@ubuntuvm01:/vagrant/stackdemo# curl http://10.0.0.5:8000
Hello World! I have been seen 2 times.
```

```
root@ubuntuvm01:/vagrant/stackdemo# docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                    NAMES
0ae465ec8696        127.0.0.1:5000/stackdemo   "python app.py"          10 seconds ago      Up 8 seconds        0.0.0.0:8000->8000/tcp   stackdemo_web_1
0998b1a0e465        redis:alpine               "docker-entrypoint.s…"   10 seconds ago      Up 8 seconds        6379/tcp                 stackdemo_redis_1
eb4f8bedf00c        registry:2                 "/entrypoint.sh /etc…"   41 minutes ago      Up 41 minutes       5000/tcp                 registry.1.uzjmcykf8vlo6bpzu8co9tnx9

```
### 3. bring app down
```
root@ubuntuvm01:/vagrant/stackdemo# docker-compose down --volumes
Stopping stackdemo_web_1   ... done
Stopping stackdemo_redis_1 ... done
Removing stackdemo_web_1   ... done
Removing stackdemo_redis_1 ... done
Removing network stackdemo_default
```
```
root@ubuntuvm01:/vagrant/stackdemo# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
eb4f8bedf00c        registry:2          "/entrypoint.sh /etc…"   42 minutes ago      Up 42 minutes       5000/tcp            registry.1.uzjmcykf8vlo6bpzu8co9tnx9

```

## Step 4: Push the generated image to the registry
```
root@ubuntuvm01:/vagrant/stackdemo# docker-compose push
Pushing web (127.0.0.1:5000/stackdemo:latest)...
```

## Step 5: Deploy the stack to the swarm
### 1. create the stack with docker stack deploy
```
root@ubuntuvm01:/vagrant/stackdemo# docker stack deploy --compose-file docker-compose.yml stackdemo
Ignoring unsupported options: build

Creating network stackdemo_default
Creating service stackdemo_web
Creating service stackdemo_redis
```
- last argument is name of stack. each network, volume and service is prefixed with the stack name.

### 2. check if service is running
```
root@ubuntuvm01:/vagrant/stackdemo# docker stack services stackdemo
ID                  NAME                MODE                REPLICAS            IMAGE                             PORTS
ht1agyel2ddw        stackdemo_web       replicated          1/1                 127.0.0.1:5000/stackdemo:latest   *:8000->8000/tcp
thl4m9jkpb6m        stackdemo_redis     replicated          1/1                 redis:alpine
```

```
root@ubuntuvm01:/vagrant/stackdemo# curl http://10.0.0.5:8000
Hello World! I have been seen 1 times.
root@ubuntuvm01:/vagrant/stackdemo# curl http://10.0.0.5:8000
Hello World! I have been seen 2 times.
```

```
-  you can access any node in swarm at port 8000
curl http://address-of-other-node:8000
```
```
root@ubuntuvm01:/vagrant/stackdemo# docker stack ps stackdemo
ID                  NAME                IMAGE                             NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
ulxvwpr8ec8h        stackdemo_web.1     127.0.0.1:5000/stackdemo:latest   ubuntuvm01          Running             Running 8 minutes ago
j6qwmg8oj735        stackdemo_redis.1   redis:alpine                      ubuntuvm01          Running             Running 10 minutes ago
```
### 3. Bring stack down
```
docker stack rm stackdemo
```

### 4. Bring registry down
```
docker service rm registry
```

### 5.  leave sawrm
```
docker swarm leave --force
```

## some commands
docker stats


- Ref : https://docs.docker.com/engine/swarm/stack-deploy/
- Install Docker compose : https://docs.docker.com/compose/install/

## Next :
- back up and revovery swarm : https://docs.docker.com/engine/swarm/admin_guide/ (Back up the entire /var/lib/docker/swarm directory.)
- https://docs.docker.com/engine/swarm/how-swarm-mode-works/pki/
- Docker Engine managed plugin system : https://docs.docker.com/engine/extend/
- storage : https://docs.docker.com/storage/

