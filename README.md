# Nginx Docker

A fork of the official nginx-deployment deployment solution with automatically generated `conf` files for other Docker containers. For more detailed instructions refer to the [original repo](https://github.com/nginx-proxy/nginx-proxy)

nginx-proxy sets up a container running nginx and [docker-gen](https://github.com/nginx-proxy/docker-gen). docker-gen generates reverse proxy configs for nginx and reloads nginx when containers are started and stopped.

See [Automated Nginx Reverse Proxy for Docker](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/) for why you might want to use this.

## Install

```shell
git clone https://github.com/ffrosch/nginx-docker ./nginx-docker
cd nginx-docker
```

## Usage

To run it with docker compose:

```console
docker compose up -d
```

### Test functionality

Change to subdirectory `./vhost-templates` and run:

```console
docker compose up -d
curl -H "Host: whoami.local" localhost
```

Example output:

```console
I'm 5b129ab83266
```

## Adding containers

Any docker container being proxied needs to copy `.env` and `docker-compose.yml` from `./vhost-templates`.

It is absolutely necessary to adjust these variables within the `.env`file:

- `VIRTUAL_HOST`: the full domain name that will be routed to this container
- `EXPOSE_PORT`: the port on which the service in the container listens; it will only be available to docker services but not to the host machine
- `CONTAINER_IMAGE`
- `CONTAINER_NAME`

The Nginx Proxy will forward all requests on a domain to the matching `VIRTUAL_HOST`. For example `curl -H "http://subdomain.yourdomain.com"` would be forwarded to the container with `VIRTUAL_HOST=subdomain.yourdomain.com`. This will only work if a service is running behind the exposed port of that container!

## Update

Add the original repo as an upstream repo

```shell
git remote add upstream https://github.com/nginx-proxy/nginx-proxy.git
```

The easiest way to synchronise the github repo AND the local copy is using the github cli

```shell
# use the -force flag if sync is not possible due to conflicts
gh repo sync ffrosch/nginx-docker -b main
```

Alternatively vanilla git can be used

```shell
git fetch upstream
git checkout main
git merge upstream/main
git push
```
