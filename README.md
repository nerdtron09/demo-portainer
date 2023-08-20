## Description
Files for demo portainer

Guide contains:

- Installation of Docker on Ubuntu 22.04 VM
- Setup Manager and Worker node for Docker Swarm cluster
- Install Portainer to manage the Docker Swarm cluster
- Deploying Stacks on portainer

## Pre-requisites
Before proceeding, you must have:

- Permissions on AWS to create VMs, Security Groups and Load balancer and ACM
- can SSH to VMs created in AWS EC2
- Create a Load balancer with HTTP and HTTPS target on AWS, create a corresponding SSL cert on ACM


## Create VMs in AWS - Docker Swarm

- Create 2 VMs on AWS: 1 as a swarm manager and 1 as a swarm worker
- Be sure the create them on a private subnet and allow communication between the two VM. 
- Create a public Load Balancer in front of the two VM so we can access the deployed docker stacks.

## Installation of Docker and setup Docker swarm nodes
Docker swarm is used to distribute the container workloads on multiple nodes.

Portainer is used as a web GUI to manage the stacks, services, and containers that are deploy the Docker swarm cluster. 
Portainer agent is run on multiple nodes while Portainer itself runs on the manager node only. 

As of the current limitation, only 1 manager node is configured while controlling mutliple worker nodes. 

Connect to all the docker VMs, upgrade the OS to install the latest patches:
```
# sudo apt update && sudo apt upgrade -y
```

Install the docker.io server packages:
```
# sudo apt install docker.io -y
```

Add ubuntu to the docker users:
```
# sudo usermod -aG docker ubuntu
```

Install the GCP logging agent to send docker logs to GCP. (If you use GCP logging)
```
# curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
# sudo bash add-google-cloud-ops-agent-repo.sh --also-install
```

On the swarm manager node, initialize the docker swarm. And generate a token for the worker to join.
```
# sudo docker swarm init
# sudo docker swarm join-token worker
```

On the worker nodes, join them to the swarm using the command generated from the previus step:
```
# sudo docker swarm join --token SWMTKN-1-6cleslotpaj3xextgua4sixxxxxx-xxxxxxx 172.16.1.10:2377
```

On the manager node, check for the node listing:
```
# sudo docker node ls 
ID                            HOSTNAME                         STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
nmcfvx9uhzipflrctbk1nqt6u *   swarm-manager-1   Ready     Active         Leader           20.10.21
qe4vnfff5ipv8folcghuzsm8p     swarm-worker-2    Ready     Active                          20.10.21
```

Due to a bug, we need to re-create the `ingress` docker network with the correct MTU based from the VM's network interface.

See this link for reference: https://portal.portainer.io/knowledge/why-cant-my-agents-communicate-with-portainer-on-swarm

Check the MTU:
```
# ip a show | grep ens | grep mtu
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc mq state UP group default qlen 1000
```

On the manager node, delete the `ingress` docker network and recreate with correct MTU. Use the `--ingress` option to ensure it is a network mesh. 
```
# sudo docker network ls | grep ingress
# sudo docker network rm ingress
# sudo docker network create --ingress -d overlay --opt com.docker.network.driver.mtu=1460 ingress
# sudo docker network inspect ingress | grep -i mtu
```

**Note: For all docker networks that will be created in the future including the default networks, ensure that proper MTU is set to match the ingress network MTU. Example in docker-compose.yml file.
```
networks:
  default:
    driver: overlay
    driver_opts:
      com.docker.network.driver.mtu: 1460
```

On the manager node, create the backend network interface to be used by portainer and the docker containers.
```
# sudo docker network create -d overlay --opt com.docker.network.driver.mtu=1460 backend
```


Next is to install the portainer service and the portainer agent. 

Reference link for installation: https://docs.portainer.io/start/install-ce/server/swarm/linux

Download the YML manifest file for portainer: (don't deploy this file)
```
curl -L https://downloads.portainer.io/ce2-18/portainer-agent-stack.yml -o portainer-agent-stack.yml
```

Modifications were made on the portainer YML file to add the MTU (if needed).

On the manager node, deploy the portainer stack. Only 1 manager is the leader at any given time.
```
docker stack deploy -c portainer-agent-stack.yml portainer
```

Add a label for the Swarm nodes. This label is used to allow/deny the node to schedule containers (optional)

```
# docker node ls
# docker node update --label-add acceptServices=1 swarm-manager-2 
# docker node update --label-add acceptServices=1 swarm-worker-2 
```

Once deployed, create a temporary DNS entry such as `portainer-demo.masterkenneth.com` pointed to the AWS load balancer to login to the Portainer Web UI.

## Deploy Docker stack using docker-compose files, or use github repo.

Go to Stacks > Add Stack > Select Repository. 

    Add a Name for the stack: demo-web-app
    Authentication: Enabled
    Username: nerdtron09 (username on bitbucket)
    Personal Access Token: Github personal access token
    Repository URL: https://github.com/nerdtron09/demo-portainer.git
    Repository reference: refs/heads/main
    Compose path: demo-web-app/docker-compose.yml
    Automatic Updates: (Optional, can be enabled)
    Environment varaibles: (Add if needed)

Click Deploy the stack to start the services and containers. 

Repeat this step until you have added all the required stack. Ensure proper branch and path for each stack that you create. 

Once all portainer stacks are running on the new Docker swarm cluster, you can update the Load Balancer Backends to point to the new instance groups.