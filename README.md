# Nginx Docker

A simplified automated reverse proxy with SSL support for docker containers.

A fork of [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) integrating [acme-companion](https://github.com/nginx-proxy/acme-companion) and simplifying the configuration of proxied containers.

- [Why you might want to use it](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/)
- detailed documentation at [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy)

**Projects using it**

- [mattermost-docker](https://github.com/ffrosch/mattermost-docker)

## Install

```shell
git clone https://github.com/ffrosch/nginx-docker ./nginx-docker
cd nginx-docker
cp env.example .env
```

Albeit **optional**, it is **recommended** to provide a valid default email address through the `DEFAULT_EMAIL` environment variable, so that Let's Encrypt can warn you about expiring certificates and allow you to recover your account.

## Usage

Run it:

```console
docker compose up -d
```

Verify success:

```console
cd ./vhost-templates
docker compose up -d
curl -H "Host: whoami.local" localhost
```

Example output:

```console
I'm 5b129ab83266
```

## Add service | container

Any docker container being proxied needs to copy `.env` and `docker-compose.yml` from `./vhost-templates`.

It is absolutely necessary to adjust these variables within the `.env`file:

- `VIRTUAL_HOST`: the full domain name that will be routed to this container
- `EXPOSE_PORT`: the port on which the service in the container listens; it will only be available to docker services but not to the host machine
- `CONTAINER_IMAGE`
- `CONTAINER_NAME`
- `LETSENCRYPT_EMAIL`: your email address
- `LETSENCRYPT_TEST`: set to `false` **before moving from testing to production**

The Nginx Proxy will forward all requests on a domain to the matching `VIRTUAL_HOST`. For example `curl -H "http://subdomain.yourdomain.com"` would be forwarded to the container with `VIRTUAL_HOST=subdomain.yourdomain.com`. This will only work if a service is running behind the exposed port of that container!

### SSL

[acme-companion](https://github.com/nginx-proxy/acme-companion) is integrated into the `docker-compose.yml` and the `vhost-templates` for automated SSL certificate generation and renewal.

**Note**: The `acme-companion` for SSL is set to use [test certificates](https://github.com/nginx-proxy/acme-companion/blob/main/docs/Let's-Encrypt-and-ACME.md#test-certificates) from the Let's Encrypt staging environment per default. This is a safer choice due to [rate-limits on production certificates](https://letsencrypt.org/docs/rate-limits/). When everything is tested and working it has to be set to production settings.

## Configure service | container

Configuration beyond environment variables.

Read [Per-VIRTUAL_HOST examples](https://github.com/nginx-proxy/nginx-proxy/discussions/1643) and [this summay](https://github.com/nginx-proxy/nginx-proxy/issues/1398#issuecomment-587717134) to get a sense on how this works.

### Mount config file on container startup

Configuration files specific to a `VIRTUAL_HOST` can be mounted into an existing volume named volume when the **Virtual Host** container is started.

This requires a **Named Volume** configured as `type: bind` and only works on **Linux** because it requires to set `propagation: rshared` which is not supported on other host systems.

### Global configuration

Add `*.conf` files at `/etc/nginx/conf.d`. The name should preferably come alphabetically after `default.conf` to make sure that `default.conf` is loaded first.

This can be used to define different `proxy_cache_paths`.

```shell
docker cp custom.conf nginx-proxy:/etc/nginx/vhost.d/servicename_proxy.conf
```

### VIRTUAL_HOST general configuration

Add a single file with `server` blocks for `VIRTUAL_HOST` at `/path/to/vhost.d/${VIRTUAL_HOST}`.

If you are using multiple hostnames for a single container (e.g. `VIRTUAL_HOST=example.com,www.example.com`), the virtual host configuration file must exist for each hostname.

If you would like to use the same configuration for multiple virtual host names, you can use a symlink:

```shell
ln -s /path/to/vhost.d/www.example.com /path/to/vhost.d/example.com
```

### VIRTUAL_HOST location configuration

Add a single file with `location` blocks for `VIRTUAL_HOST`.

If you are using **multiple hostnames** for the same container use the symlink strategy as described above.

**Augment**

File path: `/etc/nginx/vhost.d/${VIRTUAL_HOST}_location`.

**Override**

File path: `/etc/nginx/vhost.d/${VIRTUAL_HOST}_location_override`.

## Testing

See [dataminelab/docker-jenkins | local testing](https://github.com/dataminelab/docker-jenkins-nginx-letsencrypt#local-testing) and [ngrok.com](https://ngrok.com/).

To display information about existing certificates, use the following command:

```shell
docker exec nginx-proxy-acme-companion /app/cert_status
```

### Localhost NGROK

A neat way to test a proxied container locally with https.

Run **ngrok** as a docker container:

```shell
docker run --net=host -it -e NGROK_AUTHTOKEN=<your-token> ngrok/ngrok:latest http 80
```

From the **ngrok** output copy the domain-part of the `Forwarding` address (exclude `https://`). Keep **ngrok** running. Start a new shell, assign the `DOMAIN` variable and run the container:

```shell
export VIRTUAL_HOST=<ngrok-domain>; docker compose up -d
```

After a moment you will be able to access your service with `https` over the **ngrok**-address!

For testing `https` it would be good to connect locally and not via `ngrok`.

```shell
# Stop normal DNS-Server
sudo systemctl stop systemd-resolved

# Listen at standard address for normal DNS-Server
# use Google DNS (8.8.8.8)
# listen locally on all subdomains of localhost
# NOTE: DO NOT use the domain "local", it does not work (reserved or something)
sudo dnsmasq --listen-address 127.0.0.53 --server=8.8.8.8 --no-daemon --address=/*.localhost/127.0.0.1

# Restart normal DNS-Server
sudo systemctl stop systemd-resolved
sudo systemctl start systemd-resolved
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
docker exec nginx-proxy nginx -T
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
