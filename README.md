docker-gitea
============

Docker Gitea Service
--------------------

[Gitea](https://gitea.io) is a self-hosted git service written in Go that is comparable to other self-hosted git projects like [Gitlab](https://about.gitlab.com/install/?version=ce). It provides an interface that is similar to [Github](https://github.com) but a solution that you host yourself. While it does not currently have more complex features like built-in CI it is a lightweight and functional solution. This repository contains the necessary configuration to run a full Gitea service in [Docker](https://docs.docker.com) using [Docker Compose](https://docs.docker.com/compose) and the capability to auto renew SSL certificates with [Let's Encrypt](https://www.letsencrypt.org).

## Table of contents

* [Requirements](#requirements)
* [Quick start](#quick-start)
* [Additional steps](#additional-steps)
  - [Create git user](#create-git-user)
  - [SSH passthrough](#ssh-passthrough)
* [Security](#security-note)
  - [SSH root access](#ssh-root-access)
  - [External ports](#external-ports)
* [Configuration](#configuration)
  - [Environment](#environment)
  - [Images](#images)
  - [Containers](#containers)
  - [Volumes](#volumes)
  - [Advanced configuration](#advanced-configuration)
* [Documentation](#documentation)
* [Contributing](#contributing)

## Requirements

Here are the basic requirements:

* An internet connected server or VPS with a static IP address
  - SSH access to the server
  - Storage space on the server for the service and repository data
* A domain with an `A` record pointing to the server IP (Configured at DNS provider)

Name | TTL | Class | Type | Record
--- | --- | --- | --- | ---
`git.example.com` | `1200` | `IN` | `A` | `$IP`

* An email address (e.g. gitea@example.com) configured at your domain (If you want the Gitea service to be able to send email)
  - Make sure to note down the outgoing (SMTP) mail server information (e.g. smtp.example.com:465)

This guide assumes you are using Debian/Ubuntu but it can be adapted to other variations of linux. 

If you would like to add additional configuration options or help automate some of the setup see [contributing](#contributing) below.

## Quick start

Install docker and docker-compose.

```shell
# Install docker
sudo apt-get install docker

# Install docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Make docker-compose executable
sudo chmod +x /usr/local/bin/docker-compose
```

Create `docker` group and add current user to group (or add the user you would like to run docker).

```shell
# Create docker group
sudo groupadd docker

# Add user to docker group
sudo usermod -aG docker $USER
```

Create the gitea data directory.

```shell
sudo mkdir -p /var/lib/gitea
```

Check the docker service status and run a test container.

```shell
# Verify that docker service is running
sudo systemctl status docker

# Run a test container
docker run hello-world
```

Clone this repository and setup the [.env](#environment) file for your desired configuration.

```
# Clone this repository to your computer
git clone https://github.com/jwobith/docker-gitea && cd docker-gitea

# Create a `.env` file by copying and adjusting `env.sample` for configuration.
cp env.sample .env
```

Start the docker service

```shell
# Start docker containers
docker-compose up -d

# Verify containers are running
docker ps
```

## Addtional Steps

### Create git user

Create a new `git` user on the host machine with UID and GID matching the `git` user inside the Gitea container.

```shell
# Create git user
adduser git

# Make sure user has UID and GID 1000
usermod -u 1000 -g 1000 git
```

### SSH passthrough

Create the file `/app/gitea/gitea` with the following contents:

```shell
#!/bin/sh
ssh -p 2222 -o StrictHostKeyChecking=no git@127.0.0.1 "SSH_ORIGINAL_COMMAND=\"$SSH_ORIGINAL_COMMAND\" $0 $@"
```

Make the file `/app/gitea/gitea` excecutable.

`sudo chmod +x /app/gitea/gitea`

Generate an SSH key for the `git` user. When prompted for a password you can leave it empty.

To generate an RSA key:

```shell
sudo -u git ssh-keygen -t rsa -b 4096 -C "Gitea Host Key"
```

Alternately, to generate an ED25519 key:

```shell
sudo -u git ssh-keygen -t ed25519 -C "Gitea Host Key"
```

Create a symlink between container `authorized_keys` and host git user `authorized_keys.`

```shell
ln -s /var/lib/gitea/git/.ssh/authorized_keys /home/git/.ssh/authorized_keys
```

Echo the `git` user key into the `authorized_keys` file.

For an RSA key:

```shell
echo "no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty $(cat /home/git/.ssh/id_rsa.pub)" >> /var/lib/gitea/git/.ssh/authorized_keys
```

For an ED25519 key:

```shell
echo "no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty $(cat /home/git/.ssh/id_ed25519.pub)" >> /var/lib/gitea/git/.ssh/authorized_keys
```

### Installation

The first time you go to the site Gitea will guide you through the installation wizard.

* Create an administrator user with a strong password.
* Enter the email address and password for the Gitea server email account.
* Enter the correct mail server information.
* The remaining items should stay at the default setting.

## Security

On the host machine, make sure to use a strong user password and strong SSH keys. When you create the Gitea administrator for the first time use a strong password as well.

### SSH root access

Disable root SSH access on the host machine. Edit `/etc/ssh/sshd_config` by changing the following line:

```shell
# Old sshd_config
PermitRootLogin yes

# New sshd_config
PermitRootLogin no
```

NOTE: If you are currently remotely accessing the machine as root or have edited the `/etc/ssh/sshd_config` incorrectly, the next command may cause you to lose connection to the server.  Make sure you are connected via SSH as a non-root user.

Restart the ssh server with `sudo service ssh restart`.

### External ports

If a firewall is configured on the host the following external ports must be opened:
  
* 80/tcp for Web UI HTTP
* 443/tcp for Web UI HTTPS
* 22/tcp for SSH

On a Debian/Ubuntu server this can be configured using UFW:

```shell
# Install ufw
sudo apt-get install ufw

# Enable ufw service
sudo systemct enable ufw

# Set ufw default to deny all incoming
sudo ufw default deny incoming

# Set ufw default to allow all outgoing
sudo ufw default allow outgoing

# Set ufw to allow 80/tcp, 443/tcp, and 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp

# Display status of ufw service
sudo ufw status verbose
```

## Configuration

### Environment

The configuration is performed via environment variables contained in a `.env` file. You can copy the provided `env.sample` file as a reference.

Variable | Description | Example
--- | --- | ---
`APP_NAME` | Name to display on homepage and tab | Gitea: Git with a cup of tea
`PROTOCOL` | Protocol for Gitea server | (Default: https)
`DOMAIN` | Domain for the Gitea service | git.example.com
`VIRTUAL_HOST` | Virtual host for Gitea server | git.example.com
`VIRTUAL_PORT` | Virtual port for Gitea server to expose to proxy network | 3000
`LETSENCRYPT_DOMAIN` | Domain for which to generate the certificate | git.example.com
`LETSENCRYPT_EMAIL` | E-Mail for receiving important account notifications (mandatory) | admin@example.com
`DB_NAME` | Name for the database | gitea
`DB_USER` | User for the database | gitea
`DB_PASSWD` | Password for the database | gitea

### Images

* **nginx/nginx**: Nginx docker image on docker hub.
* **jwilder/docker-gen**: Docker-gen image on docker hub.
* **jrcs/letsencrypt-nginx-proxy-companion**: Proxy companion docker image on docker hub.
* **gitea/gitea**: Gitea docker image on docker hub.
* **postgres:9.6**: PostgreSQL docker image version 9.6 on docker hub.

### Containers

* **nginx**: Reverse proxy provided by nginx.
* **nginx-gen**: Container generation for nginx using docker-gen and template `nginx.tmpl`.
* **nginx-proxy-companion**: Companion to nginx for creating, renewing, and using Let's Encrypt SSL certificates.
* **gitea**: Gitea, a self-hosted git service written in Go.
* **db**: PostgreSQL, the database for the git server.

### Volumes

Local
* **/var/lib/gitea**: Persistent volume for Gitea data

Named
* **conf**: Persistent volume for nginx configuration
* **vhost**: Persistent volume for nginx virtual host configuration
* **html**: Persistent volume for nginx html data
* **certs**: Persistent volume for nginx certificate data
* **postgres**: Persistent volume for PostgreSQL database

### Advanced configuration

To make additional configuration changes first shut down the containers with `docker-compose down`

* Edit `docker-compose.yml` to update the Docker service
* Edit `/var/lib/gitea/gitea/conf/app.ini` to update the Gitea configuration
* Edit `nginx.tmpl` to update the Nginx configuration

Restart the containers with `docker-compose up -d`

## Documentation

* [Gitea Website](https://gitea.io)
* [Gitea Docker Installation](https://docs.gitea.io/en-us/install-with-docker)
* [Docker](https://docs.docker.com)
* [Docker Compose](https://docs.docker.com/compose)
* [Gitea Repo](https://github.com/go-gitea/gitea)
* [Gitea Image](https://hub.docker.com/r/gitea/gitea)
* [Nginx Repo](https://github.com/nginx/nginx)
* [Nginx Image](https://hub.docker.com/\_/nginx)
* [Docker Repo](https://github.com/jwilder/docker-gen)
* [docker-gen Repo](https://github.com/jwilder/docker-gen)
* [docker-gen Image](https://hub.docker.com/r/jwilder/docker-gen)
* [docker-letsencrypt-nginx-proxy-companion Repo](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion)
* [letsencrypt-nginx-proxy-companion Image](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion)
* If you find any problems please fill out an [issue](https://github.com/jwobith/docker-gitea/issues/new). Thank you!

## Contributing

Do you want to help contribute to this repository? Check out the [contributing documentation](CONTRIBUTING.md).

## License

This project is licensed under the MIT License.
See the [LICENSE](LICENSE) file for the full license text.
