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

### SSL

When using wildcard SSL certificates like `*.my-domain.com` all subdomains will also we available with SSL.

[acme-companion](https://github.com/nginx-proxy/acme-companion) is integrated into the `docker-compose.yml` and the `vhost-templates` for automated SSL certificate generation and renewal.

To display information about existing certificates, use the following command:

```shell
docker exec nginx-proxy-acme-companion /app/cert_status
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

**Note**: The `acme-companion` for SSL is set to use [test certificates](https://github.com/nginx-proxy/acme-companion/blob/main/docs/Let's-Encrypt-and-ACME.md#test-certificates) from the Let's Encrypt staging environment per default. This is a safer choice due to [rate-limits on production certificates](https://letsencrypt.org/docs/rate-limits/). When everything is tested and working it has to be set to production settings.

### Per-VIRTUAL_HOST configuration

To add settings on a per-`VIRTUAL_HOST` basis, add your configuration file under `/etc/nginx/vhost.d`. Unlike in the proxy-wide case, which allows multiple config files with any name ending in `.conf`, the per-`VIRTUAL_HOST` file must be named exactly after the `VIRTUAL_HOST`.

In order to allow virtual hosts to be dynamically configured as backends are added and removed, it makes the most sense to mount an external directory as `/etc/nginx/vhost.d` as opposed to using derived images or mounting individual configuration files.

For example, if you have a virtual host named `app.example.com`, you could provide a custom configuration for that host as follows:

```console
docker run -d -p 80:80 -p 443:443 -v /path/to/vhost.d:/etc/nginx/vhost.d:ro -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
{ echo 'server_tokens off;'; echo 'client_max_body_size 100m;'; } > /path/to/vhost.d/app.example.com
```

If you are using multiple hostnames for a single container (e.g. `VIRTUAL_HOST=example.com,www.example.com`), the virtual host configuration file must exist for each hostname. If you would like to use the same configuration for multiple virtual host names, you can use a symlink:

```console
{ echo 'server_tokens off;'; echo 'client_max_body_size 100m;'; } > /path/to/vhost.d/www.example.com
ln -s /path/to/vhost.d/www.example.com /path/to/vhost.d/example.com
```

### Per-VIRTUAL_HOST location configuration

To add settings to the "location" block on a per-`VIRTUAL_HOST` basis, add your configuration file under `/etc/nginx/vhost.d` just like the previous section except with the suffix `_location`.

For example, if you have a virtual host named `app.example.com` and you have configured a proxy_cache `my-cache` in another custom file, you could tell it to use a proxy cache as follows:

```console
docker run -d -p 80:80 -p 443:443 -v /path/to/vhost.d:/etc/nginx/vhost.d:ro -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
{ echo 'proxy_cache my-cache;'; echo 'proxy_cache_valid  200 302  60m;'; echo 'proxy_cache_valid  404 1m;' } > /path/to/vhost.d/app.example.com_location
```

If you are using multiple hostnames for a single container (e.g. `VIRTUAL_HOST=example.com,www.example.com`), the virtual host configuration file must exist for each hostname. If you would like to use the same configuration for multiple virtual host names, you can use a symlink:

```console
{ echo 'proxy_cache my-cache;'; echo 'proxy_cache_valid  200 302  60m;'; echo 'proxy_cache_valid  404 1m;' } > /path/to/vhost.d/app.example.com_location
ln -s /path/to/vhost.d/www.example.com /path/to/vhost.d/example.com
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
