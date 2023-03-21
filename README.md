# Nginx Docker

A fork of the official nginx-deployment deployment solution with automatically generated `conf` files for other Docker containers. For more detailed instructions refer to the [original repo](https://github.com/nginx-proxy/nginx-proxy)

nginx-proxy sets up a container running nginx and [docker-gen](https://github.com/nginx-proxy/docker-gen). docker-gen generates reverse proxy configs for nginx and reloads nginx when containers are started and stopped.

See [Automated Nginx Reverse Proxy for Docker](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/) for why you might want to use this.

## Overview

Projects using **nginx-docker**:

- [mattermost-docker](https://github.com/ffrosch/mattermost-docker)

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

### Test general functionality

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
- `LETSENCRYPT_EMAIL`: your email address
- `LETSENCRYPT_TEST`: set to `false` before moving from testing to production

The Nginx Proxy will forward all requests on a domain to the matching `VIRTUAL_HOST`. For example `curl -H "http://subdomain.yourdomain.com"` would be forwarded to the container with `VIRTUAL_HOST=subdomain.yourdomain.com`. This will only work if a service is running behind the exposed port of that container!

### SSL

When using wildcard SSL certificates like `*.my-domain.com` all subdomains will also we available with SSL.

[acme-companion](https://github.com/nginx-proxy/acme-companion) is integrated into the `docker-compose.yml` and the `vhost-templates` for automated SSL certificate generation and renewal.

**Note**: The `acme-companion` for SSL is set to use [test certificates](https://github.com/nginx-proxy/acme-companion/blob/main/docs/Let's-Encrypt-and-ACME.md#test-certificates) from the Let's Encrypt staging environment per default. This is a safer choice due to [rate-limits on production certificates](https://letsencrypt.org/docs/rate-limits/). When everything is tested and working it has to be set to production settings.

### Custom Configuration (per container)

If you need to configure Nginx beyond what is possible using environment variables, you can provide custom configuration files on either a proxy-wide or per-`VIRTUAL_HOST` basis.

Read [Per-VIRTUAL_HOST examples](https://github.com/nginx-proxy/nginx-proxy/discussions/1643) and [this summay](https://github.com/nginx-proxy/nginx-proxy/issues/1398#issuecomment-587717134) to get a sense on how this works.

#### Mounting file into volume

Configuration files specific to a `VIRTUAL_HOST` can be mounted into an existing volume. This way, everytime the container runs, it's specific configuration files are available.

TODO: check whether `nginx-proxy` or `acme-companion` modify any of these custom files in place. Although this shouldn't be a problem, because in this case the source file will be altered.

```yml
# docker-compose.yml
version: "3.8"
services:
  virtual-host-container:
    image: your-image
    volumes:
      - conf:/conf.d
      - vhost:/vhost.d
      - ./myconf.conf:/conf.d/myconf.conf
      - ./vhost.conf:/vhost.d/app.example.com
      - ./vhost.conf:/vhost.d/www.app.example.com
      - ./vhost_location.conf:/vhost.d/app.example.com_location

volumes:
  conf:
    external: true
  vhost:
    external: true
```

#### Modifying template

If nothing else works, modifying `nginx.tmpl` to include certain things like a customized upstream block could be a solution. It is likely that this would require to setup the `nginx-proxy` with seperate containers (see original repo).

#### Proxy-wide configuration

To add settings on a proxy-wide basis, add your configuration file under `/etc/nginx/conf.d` using a name ending in `.conf`. The name should preferably come alphabetically after `default.conf` to make sure that `default.conf` is loaded first.

This method is for example useful for custom `upstream` directives!

#### VIRTUAL_HOST General configuration

If you want to use custom nginx configuration for a specific container/virtual-host, you can copy the configuration file to the `vhost` volume. The file has to have the **exact same name** as the `VIRTUAL_HOST`. For example, if you have a virtual host named `app.example.com`, you could provide a custom configuration for that host at `/path/to/vhost.d/app.example.com`.

```shell
# This should work
docker cp custom.conf nginx-proxy:/etc/nginx/vhost.d/<virtual-host-name>
```

If you are using multiple hostnames for a single container (e.g. `VIRTUAL_HOST=example.com,www.example.com`), the virtual host configuration file must exist for each hostname. If you would like to use the same configuration for multiple virtual host names, you can use a symlink:

```shell
ln -s /path/to/vhost.d/www.example.com /path/to/vhost.d/example.com
```

#### VIRTUAL_HOST Location configuration

To add settings to the "location" block on a per-`VIRTUAL_HOST` basis, add your configuration file under `/etc/nginx/vhost.d` just like the previous section except with the suffix `_location` like so: `/path/to/vhost.d/app.example.com_location`.
This strategy will **augment** any generated `location` blocks. If you want to completely override the `location` block, use `_location_override` as suffix.

If you are using multiple hostnames for the same container use the symlink strategy as described in the **General configuration**.

### Testing

See [dataminelab/docker-jenkins | local testing](https://github.com/dataminelab/docker-jenkins-nginx-letsencrypt#local-testing) and [ngrok.com](https://ngrok.com/).

To display information about existing certificates, use the following command:

```shell
docker exec nginx-proxy-acme-companion /app/cert_status
```

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

## Troubleshooting

If you can't access your `VIRTUAL_HOST`, make sure the container with the virtual host address is running and inspect the generated nginx configuration:

```shell
docker exec <nginx-proxy-instance> nginx -T
```

Pay attention to the upstream definition blocks, which should look like this:

```ini
# foo.example.com
upstream foo.example.com {
	## Can be connected with "my_network" network
	# Exposed ports: [{   <exposed_port1>  tcp } {   <exposed_port2>  tcp } ...]
	# Default virtual port: <exposed_port|80>
	# VIRTUAL_PORT: <VIRTUAL_PORT>
	# foo
	server 172.18.0.9:<Port>;
	# Fallback entry
	server 127.0.0.1 down;
}
```

If you get an `error network not found` message from your virtual host container, try running it with:

```shell
docker compose up --force-recreate
```

Further ideas for troubleshooting can be found in the [wiki](https://github.com/nginx-proxy/nginx-proxy/wiki/Troubleshooting).
