# Description
A simple Docker Swarm cluster made of: one manager and two workers using docker containers deployed in AWS EC2

## Desploy
Follow the next steps to deploy your docker swarm cluster in EC".

- **STEP1**: start all the nodes inside the minikube network where jenkins is deployed

```
$ docker run -d --privileged --name swarm-manager --network minikube docker:dind
$ docker run -d --privileged --name swarm-worker1 --network minikube docker:dind
$ docker run -d --privileged --name swarm-worker2 --network minikube docker:dind
```

- **STEP2**: discover ip address of the swarm manager node

```
$ docker exec swarm-manager ip addr show eth0
13: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:c0:a8:31:02 brd ff:ff:ff:ff:ff:ff
    inet 192.168.49.2/24 brd 192.168.49.255 scope global eth0
       valid_lft forever preferred_lft forever
```

- **STEP3**: start swarm mode in the manager node anfd get the join-token

```
$ docker exec -it swarm-manager docker swarm init --advertise-addr 192.168.49.2
Swarm initialized: current node (kbdldapqc6b65k18brblxmwzj) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3kdz2ea1w7bkl2t0oj16ctalw5avhjdh0ef2ole1la8v1q0k5o-01kr672az3uoq3ksrjtek7oi8 192.168.49.2:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

- **STEP4**: discover ip address of the swarm wrokers node
```
$ docker exec swarm-worker1 ip addr show eth0
15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:c0:a8:31:04 brd ff:ff:ff:ff:ff:ff
    inet 192.168.49.4/24 brd 192.168.49.255 scope global eth0
       valid_lft forever preferred_lft forever

$ docker exec swarm-worker2 ip addr show eth0       
17: eth0@if18: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:c0:a8:31:05 brd ff:ff:ff:ff:ff:ff
    inet 192.168.49.5/24 brd 192.168.49.255 scope global eth0
       valid_lft forever preferred_lft forever
```

- **STEP5**: join the workers nodes to the swarm cluster

Using the swarm manager IP: 192.168.49.2 we will join the other workers nodes to the swarm cluster

```
$ docker exec -it swarm-worker1 docker swarm join --token SWMTKN-1-3kdz2ea1w7bkl2t0oj16ctalw5avhjdh0ef2ole1la8v1q0k5o-01kr672az3uoq3ksrjtek7oi8 192.168.49.2
This node joined a swarm as a worker.

$ docker exec -it swarm-worker2 docker swarm join --token SWMTKN-1-3kdz2ea1w7bkl2t0oj16ctalw5avhjdh0ef2ole1la8v1q0k5o-01kr672az3uoq3ksrjtek7oi8 192.168.49.2
This node joined a swarm as a worker.
```

- **STEP6**: Verify the swarm cluster created
```
$ docker exec -it swarm-manager docker node ls
ID                            HOSTNAME       STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
ne8vnvejion08zt4ii1w4jds1     e5a5ef6cc0ad   Ready     Active                          29.0.4
y8i0by57s8tefkfr6iklmgxnp     e5866becc3cf   Ready     Active                          29.0.4
kbdldapqc6b65k18brblxmwzj *   f8290a07cd38   Ready     Active         Leader           29.0.4
```

- **STEP7**: To deploy one of our web-service deployed in docker hub
```
$ docker exec -it swarm-manager docker service create --name web-service --publish 8090:80 ofertoio/web-service:v20251130-174016

vlzr9bodlsjk3lnv612gocbes
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service vlzr9bodlsjk3lnv612gocbes converged 
```

- **STEP8**: To list or check the services status deployed in the cluster swarm
```
$ docker exec -it swarm-manager docker service ls
$ docker exec -it swarm-manager docker service ps test-web
```

- **STEP7**: To access to the service
To access to the service we must use the manager IP. In our case
```
http://192.168.49.2:8090/
```

# BONUS
We can use this docker compose to start and stop our swarm cluster:

```
version: "3.9"

services:
  manager:
    image: docker:dind
    container_name: swarm-manager
    privileged: true
    networks:
      - minikube
    environment:
      - DOCKER_TLS_CERTDIR=
    command: ["--hosts", "tcp://0.0.0.0:2375", "--hosts", "unix:///var/run/docker.sock"]
    ports:
      - "2375:2375"   # manager docker API
    volumes:
      - manager-data:/var/lib/docker

  worker1:
    image: docker:dind
    container_name: swarm-worker1
    privileged: true
    networks:
      - minikube
    environment:
      - DOCKER_TLS_CERTDIR=
    command: ["--hosts", "tcp://0.0.0.0:2375", "--hosts", "unix:///var/run/docker.sock"]
    volumes:
      - worker1-data:/var/lib/docker

  worker2:
    image: docker:dind
    container_name: swarm-worker2
    privileged: true
    networks:
      - minikube
    environment:
      - DOCKER_TLS_CERTDIR=
    command: ["--hosts", "tcp://0.0.0.0:2375", "--hosts", "unix:///var/run/docker.sock"]
    volumes:
      - worker2-data:/var/lib/docker

networks:
  minikube:

volumes:
  manager-data:
  worker1-data:
  worker2-data:
```

## Desploy Portianer
Follow the next steps to deploy portainer inside the cluster swarm DinD to have extra functionalities inside

- **STEP1**: Create a volume
Login into the manager node of the cluster and create the volume for portainer
```
$ docker volume create portainer_data
```

- **STEP1**: Deploy portainer
```
$ docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

After deploy configure the admin password as usual an login in the new portainer portal. In this case we will use the manager node ip for it

```
https://192.168.49.2:9443
```

Portainer in Swarm mode
![Portainer Swarm Mode](./images/portainer_swarm_mode.png "Portainer Swarm Mode")

Swarm Cluster Visualizer
![Portainer Swarm Cluster](./images/portainer_swarm_cluster.png "Portainer Swarm Cluster")