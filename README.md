---
theme: default
_class: lead
paginate: true
backgroundColor: #fff
marp: true
---

# Assignment 1
## Vagrant and Docker task

###### 2020-2021

> [INTERNET OF THINGS MASTER DEGREE](https://masteriot.etsist.upm.es)

![bg right:65% 100%](https://cdn.educba.com/academy/wp-content/uploads/2020/02/Vagrant-vs-Docker.jpg)

---

# Requirements summary

- MacOS, Windows and Linux (Recommended)
    - Install [Virtual Box](https://www.virtualbox.org) and [Vagrant](https://www.vagrantup.com)
    - Use [Docker Playground](https://labs.play-with-docker.com) for Docker to avoid *Windows world* (Hyper-V, WSL, etc...)
    - Registration in [Docker Hub](https://hub.docker.com/signup)

> MacOS and Linux users can install Docker with no problems, but Docker Playground is fine enough.   

All needed file could be download from Moodle or via [GitHub](https://github.com/ETSISI-CCforIoT/assignment1)

---

# Vagrant assignment
### Assignment 1 (Part I)

![bg left](https://cdn.thenewstack.io/media/2015/11/vagrant_header_background-482a12a7-1024x476.png)

---

# Software Requirements

- [Virtual Box](https://www.virtualbox.org)
- [Vagrant](https://www.vagrantup.com)
    - for Windows 
        - **disable** Hyper-V
    - for Mac
    - for Linux

---

- Initialize a directory for usage with Vagrant 

```sh
vagrant init
```

- Store the box hashicorp/precise64

```sh
vagrant box add hashicorp/precise64
```

---

- Edit the `vagrant` file

```sh
Vagrant.configure("2") do |config|
  config.vm.box = " hashicorp/precise64"
  config.vm.provision :shell, path: "bootstrap.sh"
end
```

- Edit the `bootstrap.sh` file (create it if needed)

```sh
#!/usr/bin/env bash
apt-get update
apt-get install -y apache2
if ! [ -L /var/www ]; then
rm -rf /var/www
ln -fs /vagrant /var/www
fi
``` 

---

- Start the virtual machine
```sh
vagrant up
``` 

- SSH into guest machine

```sh
vagrant ssh
``` 

- Run the following command in the guest machine

```sh
vagrant@precise64:~$ wget -qO- 127.0.0.1
````


---

# Docker assignment

### Assignment 1 (Part II)

![bg left 90%](https://deploybot.com/assets/blog/Using-Docker-Containersposting.png)

---

# Software Requirements

- [Docker Playground](https://labs.play-with-docker.com) (Recommended -- via web browser)
- [Docker Desktop](https://www.docker.com)
    - for Windows 
        - **enable** WSL 2 (Recommended)
        - or enable Hyper-V
    - for Mac
    - for Linux

> Account in Docker is required!

---

# Part I: Docker Images

---

- Define a container with Dockerfile (file `Dockerfile`)

```docker
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt
EXPOSE 80

# Define environment variable
ENV NAME lfmingo

# Run app.py when the container launches
CMD ["python", "app.py"]
```

---

- Create the app (file `app.py`)

```python
from flask import Flask
from redis import Redis, RedisError 
import os
import socket
   # Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2) 
app = Flask(__name__)
@app.route("/")
def hello():
  try:
    visits = redis.incr("counter")
  except RedisError:
    visits = "<i>cannot connect to Redis, counter disabled</i>"
  html = "<h3>Hello {name}!</h3>" \
      "<b>Hostname:</b> {hostname}<br/>" \    
      "<b>Visits:</b> {visits}"
  return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)
if __name__ == "__main__": 
  app.run(host='0.0.0.0', port=80)
```
---

- Create the file `requirements.txt`

```txt
Flask
Redis
```

---

- Build the app = Create the Docker image

```sh
docker image build --tag lfmingo:1.0 .
```
---

- Run the container

```sh
docker run -d -p 8080:80 lfmingo:1.0 -n lfmingo
```
---

- Stop and Run again

```sh
docker stop lfmingo
docker start lfmingo
```
---

- Share the image into Docker Hub 

```sh
docker login
docker image tag lfmingo:1.0 lfmingo/lfmingo:1.0
docker push lfmingo/lfmingo:1.0 
```

Note that the tag must be in format `user/image:number`

---

- Push the image

```sh
docker pull lfmingo/lfmingo:1.0
docker run -d -p 8080:80 lfmingo/lfmingo:1.0 -n lfmingo
```

---

# Part II: Docker Compose Services on Swarm

---

- Define `docker-compose-service.yml` file

```yaml
version: "3" 
services:
    web:    
        image: lfmingo/lfmingo:1.0
        deploy:
            replicas: 5 
            resources:
                limits:
                    cpus: "0.1" 
                    memory: 50M
            restart_policy: 
                condition: on-failure
        ports:
            - "8080:80"
        networks: 
            - webnet

networks: 
    webnet:
```

---

- Run the app as a service on a swarm

```sh
docker swarm init
docker stack deploy -c docker-compose-services.yaml webserver
```

To delete the stack `docker stack rm webserver`

To leave the swarm `docker swarm leave --force`

---
# Part II: Docker Compose Stacks on Swarm

---

- Define `docker-compose-stack.yml` file
```yaml
version: "3" 
services:
    web:    
        image: lfmingo/lfmingo:1.0
        deploy:
            replicas: 5 
            resources:
                limits:
                    cpus: "0.1" 
                    memory: 50M
            restart_policy: 
                condition: on-failure
        ports:
            - "8080:80"
        networks: 
            - webnet
    visualizer:
        image: dockersamples/visualizer:stable 
        ports:
            - "8090:8080" 
        volumes:
            - "/var/run/docker.sock:/var/run/docker.sock" 
        deploy:
            placement:
                constraints: [node.role == manager]
        networks: 
            - webnet
    redis:
        image: redis:4.0.5-alpine
        command: ["redis", "--appendonly", "yes"]
        hostname: redis
        networks:
            - webnet 
        volumes:
            - redis-data:/data 
networks:
    webnet: 
volumes:
    redis-data:
```

---

- Run the app as a stack on a swarm

```sh
docker swarm init
docker stack deploy -c docker-compose-stack.yaml webapp
```

To remove the stack `docker stack rm webapp`


---

# Part III: Docker Swarm

---
## Install `docker-machine`


> OSX
```sh
base=https://github.com/docker/machine/releases/download/v0.16.0 &&
  curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/usr/local/bin/docker-machine &&
  chmod +x /usr/local/bin/docker-machine
```

> Windows with GitBash
```sh
base=https://github.com/docker/machine/releases/download/v0.16.0 &&
  mkdir -p "$HOME/bin" &&
  curl -L $base/docker-machine-Windows-x86_64.exe > "$HOME/bin/docker-machine.exe" &&
  chmod +x "$HOME/bin/docker-machine.exe"
```

See https://docs.docker.com/machine/install-machine/


---


## MacOS / Linux


- Set up the swarm
    - 1 master (Virtual Machines with docker engine)
    - 2 workers (Virtual Machines with docker engine)

```sh
docker-machine create master
docker-machine create worker-1
docker-machine create worker-2
````


You can ssh to worker with: `docker-machine ssh <name>`

You can discover IP address with: `docker-machine ip <name>`


---

## Windows 10

- Set up the swarm
    - 1 master (Virtual Machines with docker engine)
    - 2 workers (Virtual Machines with docker engine)

Launch Hyper-V Manager
- Click Virtual Switch Manager in the right-hand menu
- Click Create Virtual Switch of type External
- Give it the name `myswitch`, and check the box to share your host machine’s active network adapter
```sh
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" master
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" worker-1
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" worker-2
```

---

## Windows, MacOS and Linux

--- 

- Create a cluster

In the manager/master machine run:

```sh
docker swarm init <IP>
```

To reobtain the token run (in the master node)

```sh
docker swarm join-token worker
```


---

- Initialize the swarm and add nodes 

In the workers run 
```sh
docker swarm join \
    --token <token-obtained-from-master-machine> \
    <ip-master-machine>:2377`
```

To check the nodes and roles `docker node ls`


---

- Deploy the app on the swarm 



In the host machine: `docker-machine env <nombre master>` and all following `docker` commands will be run in `>nombre master>` machine (to reset `docker-machine env --unset`)

 `docker stack deploy -c docker-compose-stack.yaml webapp`

Run:

```sh
docker node ls
docker stack ps webapp  
docker stack services webapp
docker service ls
docker ps
docker-machine ls
```

---

- Re-scaling

Check services with `docker service ls`

```sh
docker service scale webapp_visualizer=2
```

---

- Cleanup and reboot

In master machine:

```sh
docker stack rm webapp
```

In workers:

```sh
docker swarm leave --force
```
