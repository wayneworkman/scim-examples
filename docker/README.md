# Deploying the 1Password SCIM Bridge using Docker

This example describes the methods of deploying the 1Password SCIM bridge using Docker. The Docker Compose and Docker Swarm managers are available and deployment using each manager is described below.

## Docker Compose

This is the simplest method of deploying the SCIM bridge. These instructions require a remote Docker host be set up and configured to be accessed by the Docker CLI. _Please refer to your cloud provider on how to setup a remote Docker host if you do not have one set up already or are experiencing difficulties doing so._

**Note that the Docker Compose strategy is very useful for testing, but it is not recommended for use in a production environment. The scimsession file is passed into the docker container via an environment variable, which is less secure than Docker Swarm secrets or Kubernetes secrets, both of which are supported, and recommended.**

## Docker Swarm

These instructions require a remote Docker Swarm cluster be set up and configured to be accessed by the Docker CLI. _Please refer to your cloud provider on how to setup a remote Docker Swarm Cluster if you do not have one set up already or are experiencing difficulties doing so._

## 1: Clone this repository

To make this process easier, it is recommended to clone this repository to have easy access to scripts and configuration files.

## 2: Install Docker locally

Install [Docker for Desktop](https://www.docker.com/products/docker-desktop) on your local machine and _start Docker_ before continuing, as it will be needed to run the setup process

## 3: Create your DNS record

The 1Password SCIM bridge requires SSL/TLS in order to communicate with your IdP. In order to use TLS, you must create a DNS record that points to your Docker node. _Do not attempt to perform a provisioning sync before the DNS records have been propogated_. The DNS record must exist and the SCIM bridge server must be running if you wish to have LetsEncrypt automatically issue a TLS certificate for your SCIM bridge. _Please refer to your cloud provider on how to setup a DNS record if you do not have one set up already or are experiencing difficulties doing so._

## 4: Create your scimsession file and Deploy SCIM bridge

1. Connect to your remote Docker host from your local machine
    - Either connect using [docker-machine](https://docs.docker.com/machine/), OR use SSH to access your remote maching and clone this repo.

2. In your terminal, use the bash script [./scim-setup.sh](../session/scim-setup.sh) to authenticate your account and generate a `scimsession` file : This script uses a Docker container to run the `op-scim setup` command and writes the scimsession file back to your local machine using a mounted volume. Your bearer token will be printed to the console. **Save your bearer token, as it will be needed to authenticate with your IdP**.

_The scimsession file is equivalent to your Master Password and Secret Key when combined with the bearer token, therefore they should never be stored in the same place._

Example:
```
> cd [location of cloned scim-examples folder]
> ./scim-setup.sh

[interactive script]
Bearer token: jafewnqrrupcnoiqj0829fe209fnsoudbf02efsdo

> ./docker/deploy.sh
```

3. Once your scimsession file has been created, use the bash script `./docker/deploy.sh` to deploy the SCIM bridge. _Have the domain name indicated by the DNS record created for the SCIM bridge ready_. This script will do the following :

    1. Ask if you want to deploy with Docker Swarm or Compose

    1. For `docker-compose`, it will generate a `scim.env` file that allows the scimsession file to be passed into the container without insecurely writing it to the container filesystem. For `docker-swarm`, it will create a secret called `scimsession`, which the op-scim container will then read from `/run/secrets`, as defined in docker-compose.yml.

    1. You will be prompted for your SCIM bridge domain name which will configure LetsEncrypt to automatically issue a certificate for your bridge.

    1. Lastly, it will deploy a container from the `1password/scim` image. A redis container will also be started automatically to be used by the SCIM bridge.

The logs from the SCIM bridge and redis containers will be streamed to your machine. When you are done, press ctrl+c to stop the logs, and the containers will remain running on the remote machine.

_After the DNS record has been propogated_, you can continue setting up your IdP with the SCIM bridge Administration Guide while monitoring the logs from the bridge on your local machine.

### Systemd

In order to automatically launch the 1Password SCIM bridge upon startup when using **docker-compose**, you will need to configure systemd to automatically start the Docker daemon and launch op-scim.

1. Install the service file for op-scim. A [sample](compose/op-scim.service) is provided and you'll need to change the path.
2. Reload systemd: `systemctl daemon-reload`
3. Enable the op-scim service: `systemctl enable op-scim`
