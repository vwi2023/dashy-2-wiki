# Management

_The following article explains aspects of app management, and is useful to know for when self-hosting. It covers everything from keeping the app up-to-date, secure, backed up, to other topics like auto-starting, monitoring, log management, web server configuration and using custom environments. Most of it is aimed at running the Dashy (or any other app) in a container, but some of it also applies to bare metal setups too. It's like a top-15 list of need-to-know knowledge for self-hosting._

## Contents
- [Providing Assets](#providing-assets)
- [Running Commands](#running-commands)
- [Healthchecks](#healthchecks)
- [Logs and Performance](#logs-and-performance)
- [Auto-Starting at Boot](#auto-starting-at-system-boot)
- [Updating](#updating)
- [Backing Up](#backing-up)
- [Scheduling](#scheduling)
- [SSL Certificates](#ssl-certificates)
- [Authentication](#authentication)
- [Managing with Compose](#managing-containers-with-docker-compose)
- [Environmental Variables](#passing-in-environmental-variables)
- [Securing Containers](#container-security)
- [Web Server Configuration](#web-server-configuration)
- [Running a Modified Apps](#running-a-modified-version-of-the-app)

---

## Providing Assets
Although not essential, you will most likely want to provide several assets to your running app.

This is easy to do using [Docker Volumes](https://docs.docker.com/storage/volumes/), which lets you share a file or directory between your host system, and the container. Volumes are specified in the Docker run command, or Docker compose file, using the `--volume` or `-v` flags. The value of which consists of the path to the file / directory on your host system, followed by the destination path within the container. Fields are separated by a colon (`:`), and must be in the correct order. For example: `-v ~/alicia/my-local-conf.yml:/app/public/conf.yml`

In Dashy, commonly configured resources include:
- `./public/conf.yml` - Your main application config file
- `./public/item-icons` - A directory containing your own icons. This allows for offline access, and better performance than fetching from a CDN
- Also within `./public` you'll find standard website assets, including `favicon.ico`, `manifest.json`, `robots.txt`, etc. There's no need to pass these in, but you can do so if you wish
- `/src/styles/user-defined-themes.scss` - A stylesheet for applying custom CSS to your app. You can also write your own themes here.

**[⬆️ Back to Top](#management)**

---
## Running Commands

The project has a few commands that can be used for various tasks, you can find a list of these either in the [Developing Docs](/docs/developing.md#project-commands), or by looking at the [`package.json`](https://github.com/Lissy93/dashy/blob/master/package.json#L5). These can be used by running `yarn [command-name]`.

 If you're using Docker, then you'll need to execute them within the container. This can be done by preceding each command with `docker exec -it [container-id]`, where container ID can be found by running `docker ps`. For example `docker exec -it 26c156c467b4 yarn build`. You can also enter the container, with `docker exec -it [container-id] /bin/ash`, and navigate around it with normal Linux commands.

**[⬆️ Back to Top](#management)**

---
## Healthchecks

Healthchecks are configured to periodically check that Dashy is up and running correctly on the specified port. By default, the health script is called every 5 minutes, but this can be modified with the `--health-interval` option. You can check the current container health with: `docker inspect --format "{{json .State.Health }}" [container-id]`, and a summary of health status will show up under `docker ps`. You can also manually request the current application status by running `docker exec -it [container-id] yarn health-check`. You can disable healthchecks altogether by adding the `--no-healthcheck` flag to your Docker run command.

To restart unhealthy containers automatically, check out [Autoheal](https://hub.docker.com/r/willfarrell/autoheal/). This image watches for unhealthy containers, and automatically triggers a restart. (This is a stand in for Docker's `--exit-on-unhealthy` that was proposed, but [not merged](https://github.com/moby/moby/pull/22719)).

```
docker run -d \
    --name autoheal \
    --restart=always \
    -e AUTOHEAL_CONTAINER_LABEL=all \
    -v /var/run/docker.sock:/var/run/docker.sock \
    willfarrell/autoheal
```

**[⬆️ Back to Top](#management)**

---
## Logs and Performance

#### Container Logs
You can view logs for a given Docker container with `docker logs [container-id]`, add the `--follow` flag to stream the logs. For more info, see the [Logging Documentation](https://docs.docker.com/config/containers/logging/). There's also [Dozzle](https://dozzle.dev/), a useful tool, that provides a web interface where you can stream and query logs from all your running containers from a single web app.

#### Container Performance
You can check the resource usage for your running Docker containers with `docker stats` or `docker stats [container-id]`. For more info, see the [Stats Documentation](https://docs.docker.com/engine/reference/commandline/stats/). There's also [cAdvisor](https://github.com/google/cadvisor), a useful web app for viewing and analyzing resource usage and performance of all your running containers.

#### Management Apps
You can also view logs, resource usage and other info as well as manage your entire Docker workflow in third-party Docker management apps. For example [Portainer](https://github.com/portainer/portainer) an all-in-one open source management web UI  for Docker and Kubernetes, or [LazyDocker](https://github.com/jesseduffield/lazydocker) a terminal UI for Docker container management and monitoring.

#### Advanced Logging and Monitoring
Docker supports using [Prometheus](https://prometheus.io/) to collect logs, which can then be visualized using a platform like [Grafana](https://grafana.com/). For more info, see [this guide](https://docs.docker.com/config/daemon/prometheus/). If you need to route your logs to a remote syslog, then consider using [logspout](https://github.com/gliderlabs/logspout). For enterprise-grade instances, there are managed services, that make monitoring container logs and metrics very easy, such as [Sematext](https://sematext.com/blog/docker-container-monitoring-with-sematext/) with [Logagent](https://github.com/sematext/logagent-js).

**[⬆️ Back to Top](#management)**

---

## Auto-Starting at System Boot

You can use Docker's [restart policies](https://docs.docker.com/engine/reference/run/#restart-policies---restart) to instruct the container to start after a system reboot, or restart after a crash. Just add the `--restart=always` flag to your Docker compose script or Docker run command. For more information, see the docs on [Starting Containers Automatically](https://docs.docker.com/config/containers/start-containers-automatically/).

For Podman, you can use `systemd` to create a service that launches your container, [the docs](https://podman.io/blogs/2018/09/13/systemd.html) explains things further. A similar approach can be used with Docker, if you need to start containers after a reboot, but before any user interaction.

To restart the container after something within it has crashed, consider using [`docker-autoheal`](https://github.com/willfarrell/docker-autoheal) by @willfarrell, a service that monitors and restarts unhealthy containers. For more info, see the [Healthchecks](#healthchecks) section above.

**[⬆️ Back to Top](#management)**

---
## Updating

Dashy is under active development, so to take advantage of the latest features, you may need to update your instance every now and again.

### Updating Docker Container
1. Pull latest image: `docker pull lissy93/dashy:latest`
2. Kill off existing container
	- Find container ID: `docker ps`
	- Stop container: `docker stop [container_id]`
	- Remove container: `docker rm [container_id]`
3. Spin up new container: `docker run [params] lissy93/dashy`

### Automatic Docker Updates

You can automate the above process using [Watchtower](https://github.com/containrrr/watchtower).
Watchtower will watch for new versions of a given image on Docker Hub, pull down your new image, gracefully shut down your existing container and restart it with the same options that were used when it was deployed initially.

To get started, spin up the watchtower container:

```
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower
```

For more information, see the [Watchtower Docs](https://containrrr.dev/watchtower/)

### Updating Dashy from Source
Stop your current instance of Dashy, then navigate into the source directory. Pull down the latest code, with `git pull origin master`, then update dependencies with `yarn`, rebuild with `yarn build`, and start the server again with `yarn start`.

**[⬆️ Back to Top](#management)**

---

## Backing Up

### Backing Up Containers

You can make a backup of any running container really easily, using [`docker commit`](https://docs.docker.com/engine/reference/commandline/commit/) and save it with [`docker export`](https://docs.docker.com/engine/reference/commandline/export/), to do so:
- First find the container ID, you can do this with `docker container ls`
- Now to create the snapshot, just run `docker commit -p [container-id] my-backup`
- Finally, to save the backup locally, run `docker save -o ~/dashy-backup.tar my-backup`
- If you want to push this to a container registry, run  `docker push my-backup:latest`

Note that this will not include any data in docker volumes, and the process here is a bit different. Since these files exist on your host system, if you have an existing backup solution implemented, you can incorporate and volume files within that system.

### Backing Up Volumes
[offen/docker-volume-backup](https://github.com/offen/docker-volume-backup) is a useful tool for periodic Docker volume backups, to any S3-compatible storage provider. It's run as a light-weight Docker container, and is easy to setup, and also supports GPG-encryption, email notification, and routing away older backups. 

To get started, create a docker-compose similar to the example below, and then start the container. For more info, check out their [documentation](https://github.com/offen/docker-volume-backup), which is very clear.

```yaml
version: '3'
services:
  backup:
    image: offen/docker-volume-backup:latest
    environment:
      BACKUP_CRON_EXPRESSION: "0 * * * *"
      BACKUP_PRUNING_PREFIX: backup-
      BACKUP_RETENTION_DAYS: 7
      AWS_BUCKET_NAME: backup-bucket
      AWS_ACCESS_KEY_ID: AKIAIOSFODNN7EXAMPLE
      AWS_SECRET_ACCESS_KEY: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    volumes:
      - data:/backup/my-app-backup:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
volumes:
  data:
```

It's worth noting that this process can also be done manually, using the following commands:

Backup:
```
docker run --rm -v some_volume:/volume -v /tmp:/backup alpine tar -cjf /backup/some_archive.tar.bz2 -C /volume ./
```
Restore:
```
docker run --rm -v some_volume:/volume -v /tmp:/backup alpine sh -c "rm -rf /volume/* /volume/..?* /volume/.[!.]* ; tar -C /volume/ -xjf /backup/some_archive.tar.bz2"
```
### Dashy-Specific Backup
Since Dashy is open source, and freely available, providing you're configuration data is passed in as volumes, there shouldn't be any need to backup the main container. Your main config file, and any assets you're using should be kept backed up, preferably in at least two places, and you should ensure that you can easily restore from backup, if needed.

Dashy also has a built-in cloud backup feature, which is free for personal users, and will let you make and restore fully encrypted backups of your config directly through the UI. To learn more, see the [Cloud Backup Docs](/docs/backup-restore.md)

**[⬆️ Back to Top](#management)**

---

## Scheduling

If you need to periodically schedule the running of a given command on Dashy (or any other container), then a useful tool for doing so it [ofelia](https://github.com/mcuadros/ofelia). This runs as a Docker container and is really useful for things like backups, logging, updating, notifications, etc. Crons are specified using Go's crontab format, and a useful tool for visualizing this is [crontab.guru](https://crontab.guru/). This can also be done natively with Alpine: `docker run -it alpine ls /etc/periodic`.
I recommend combining this with [healthchecks](https://github.com/healthchecks/healthchecks) for easy monitoring of jobs, and failure notifications.

**[⬆️ Back to Top](#management)**

---

## SSL Certificates

Enabling HTTPS with an SSL certificate is recommended if you hare hosting Dashy anywhere other than your home. This will ensure that all traffic is encrypted in transit.

[Let's Encrypt](https://letsencrypt.org/docs/) is a global Certificate Authority, providing free SSL/TLS Domain Validation certificates in order to enable secure HTTPS access to your website. They have good browser/ OS [compatibility](https://letsencrypt.org/docs/certificate-compatibility/) with their ISRG X1 and DST CA X3 root certificates, support [Wildcard issuance](https://community.letsencrypt.org/t/acme-v2-production-environment-wildcards/55578) done via ACMEv2 using the DNS-01 and have [Multi-Perspective Validation](https://letsencrypt.org/2020/02/19/multi-perspective-validation.html). Let's Encrypt provide [CertBot](https://certbot.eff.org/) an easy app for generating and setting up an SSL certificate

[ZeroSSL](https://zerossl.com/) is another popular certificate issuer, they are free for personal use, and also provide easy-to-use tools for getting things setup.


If you're hosting Dashy behind Cloudflare, then they offer [free and easy SSL](https://www.cloudflare.com/en-gb/learning/ssl/what-is-an-ssl-certificate/).

If you're not so comfortable on the command line, then you can use a tool like [SSL For Free](https://www.sslforfree.com/) to generate your Let's Encrypt or ZeroSSL certificate, and support shared hosting servers. They also provide step-by-step tutorials on setting up your certificate on most common platforms. If you are using shared hosting, you may find [this tutorial](https://www.sitepoint.com/a-guide-to-setting-up-lets-encrypt-ssl-on-shared-hosting/) helpful.


**[⬆️ Back to Top](#management)**

---

## Authentication

Dashy natively supports secure authentication using KeyCloak. There is also a Simple Auth feature that doesn't require any additional setup. Setup instructions for which, and alternative auth methods, has now moved to the **[Authentication Docs](/docs/authentication.md)** page.


**[⬆️ Back to Top](#management)**

---

## Managing Containers with Docker Compose

When you have a lot of containers, it quickly becomes hard to manage with `docker run` commands. The solution to this is [docker compose](https://docs.docker.com/compose/), a handy tool for defining all a containers run settings in a single YAML file, and then spinning up that container with a single short command - `docker compose up`. A good example of which can be seen in [@abhilesh's docker compose collection](https://github.com/abhilesh/self-hosted_docker_setups).

You can use Dashy's default [`docker-compose.yml`](https://github.com/Lissy93/dashy/blob/master/docker-compose.yml) file as a template, and modify it according to your needs.

An example Docker compose, using the default base image from DockerHub, might look something like this:

```yaml
---
version: "3.8"
services:
  dashy:
    container_name: Dashy
    image: lissy93/dashy
    volumes:
      - /root/my-config.yml:/app/public/conf.yml
    ports:
      - 4000:80
    environment:
      - BASE_URL=/my-dashboard
    restart: unless-stopped
    healthcheck:
      test: ['CMD', 'node', '/app/services/healthcheck']
      interval: 1m30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

**[⬆️ Back to Top](#management)**

---

## Passing in Environmental Variables

With Docker, you can define environmental variables under the `environment` section of your Docker compose file. Environmental variables are used to configure high-level settings, usually before the config file has been read. For a list of all supported env vars in Dashy, see [the developing docs](/docs/developing.md#environmental-variables), or the default [`.env`](https://github.com/Lissy93/dashy/blob/master/.env) file.

A common use case, is to run Dashy under a sub-page, instead of at the root of a URL (e.g. `https://my-homelab.local/dashy` instead of `https://dashy.my-homelab.local`). In this use-case, you'd specify the `BASE_URL` variable in your compose file.

```yaml
environment:
  - BASE_URL=/dashy
```

You can also do the same thing with the docker run command, using the [`--env`](https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables--e---env---env-file) flag.
If you've got many environmental variables, you might find it useful to put them in a [`.env` file](https://docs.docker.com/compose/env-file/). Similarly, for Docker run you can use [`--env-file`](https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables--e---env---env-file) if you'd like to pass in a file containing all your environmental variables.

**[⬆️ Back to Top](#management)**

---

## Container Security

### Keep Docker Up-To-Date
To prevent known container escape vulnerabilities, which typically end in escalating to root/administrator privileges, patching Docker Engine and Docker Machine is crucial. For more info, see the [Docker Installation Docs](https://docs.docker.com/engine/install/).

### Set Resource Quotas
Docker enables you to limit resource consumption (CPU, memory, disk) on a per-container basis. This not only enhances system performance, but also prevents a compromised container from consuming a large amount of resources, in order to disrupt service or perform malicious activities. To learn more, see the [Resource Constraints Docs](https://docs.docker.com/config/containers/resource_constraints/)

For example, to run Dashy with max of 1GB ram, and max of 50% of 1 CP core:
`docker run -d -p 8080:80 --cpus=".5" --memory="1024m" lissy93/dashy:latest`

### Don't Run as Root
Running a container with admin privileges gives it more power than it needs, and can be abused. Dashy does not need any root privileges, and Docker by default doesn't run containers as root, so providing you don't specifically type sudo, you should be all good here.

Note that if you're facing permission issues on Debian-based systems, you may need to add your user to the Docker group. First create the group: `sudo groupadd docker`,  then add your (non-root) user: `sudo usermod −aG docker [my-username]`, finally `newgrp docker` to refresh.

### Specify a User
One of the best ways to prevent privilege escalation attacks, is to configure the container to use an unprivileged user. This also means that any files created by the container and mounted, will be owned by the specified user (and not root), which makes things much easier. 

You can specify a user, using the [`--user` param](https://docs.docker.com/engine/reference/run/#user), and should include the user ID (`UID`), which can be found by running `id -u`, and the and the group ID (`GID`), using `id -g`.

With Docker run, you specify it like:
`docker run --user 1000:1000 -p 8080:80 lissy93/dashy`

Of if you're using Docker-compose, you could use an environmental variable

```yaml
version: "3.8"
services:
  dashy:
    image: lissy93/dashy
    user: ${CURRENT_UID}
    ports: [ 4000:80 ]
```   

And then to set the variable, and start the container, run: `CURRENT_UID=$(id -u):$(id -g) docker-compose up`

###  Limit capabilities 
Docker containers run with a subset of [Linux Kernal's Capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html) by default. It's good practice to drop privilege permissions that are not needed for any given container.

With Docker run, you can use the `--cap-drop` flag to remove capabilities, you can also use `--cap-drop=all` and then define just the required permissions using the `--cap-add` option. For a list of available capabilities, see the [Privilege Capabilities Docs](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities).

Here's an example using docker-compose, removing privileges that are not required for Dashy to run:

```yaml
version: "3.8"
services:
  dashy:
    image: lissy93/dashy
    ports: [ 4000:80 ]
    cap_drop:
    - ALL
    cap_add:
    - CHOWN
    - SETGID
    - SETUID
    - DAC_OVERRIDE
    - NET_BIND_SERVICE
``` 

### Prevent new Privilages being Added
To prevent processes inside the container from getting additional privileges, pass in the `--security-opt=no-new-privileges:true` option to the Docker run command (see [docs](https://docs.docker.com/engine/reference/run/#security-configuration)).

Run Command:
`docker run --security-opt=no-new-privileges:true -p 8080:80 lissy93/dashy`

Docker Compose
```yaml
security_opt:
- no-new-privileges:true
```

### Disable Inter-Container Communication
By default Docker containers can talk to each other (using [`docker0` bridged network](https://docs.docker.com/config/containers/container-networking/)). If you don't need this capability, then it should be disabled. This can be done with the `--icc=false` in your run command. You can learn more about how to facilitate secure communication between containers in the [Compose Networking docs](https://docs.docker.com/compose/networking/).

### Don't Expose the Docker Daemon Socket
Docker socket `/var/run/docker.sock` is the UNIX socket that Docker is listening to. This is the primary entry point for the Docker API. The owner of this socket is root. Giving someone access to it is equivalent to giving unrestricted root access to your host.

You should **not** enable TCP Docker daemon socket (`-H tcp://0.0.0.0:XXX`), as doing so exposes un-encrypted and unauthenticated direct access to the Docker daemon, and if the host is connected to the internet, the daemon on your computer can be used by anyone from the public internet- which is bad. If you need TCP, you should [see the docs](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-socket-option) to understand how to do this more securely.
Similarly, never expose `/var/run/docker.sock` to other containers as a volume, as it can be exploited.

### Use Read-Only Volumes
You can specify that a specific volume should be read-only by appending `:ro` to the `-v` switch. For example, while running Dashy, if we want our config to be writable, but keep all other assets protected, we would do:
```
docker run -d \
  -p 8080:80 \
  -v ~/dashy-conf.yml:/app/public/conf.yml \
  -v ~/dashy-icons:/app/public/item-icons:ro \
  -v ~/dashy-theme.scss:/app/src/styles/user-defined-themes.scss:ro \
  lissy93/dashy:latest
```

You can also prevent a container from writing any changes to volumes on your host's disk, using the `--read-only` flag. Although, for Dashy, you will not be able to write config changes to disk, when edited through the UI with this method. You could make this work, by specifying the config directory as a temp write location, with `--tmpfs /app/public/conf.yml` - but  that this will not write the volume back to your host.

### Set the Logging Level
Logging is important, as it enables you to review events in the future, and in the case of a compromise this will let get an idea of what may have happened. The default log level is `INFO`, and this is also the recommendation, use `--log-level info` to ensure this is set.

### Verify Image before Pulling
Only use trusted images, from verified/ official sources. If an app is open source, it is more likely to be safe, as anyone can verify the code. There are also tools available for scanning containers, 

Unless otherwise configured, containers can communicate among each other, so running one bad image may lead to other areas of your setup being compromised. Docker images typically contain both original code, as well as up-stream packages, and even if that image has come from a trusted source, the up-stream packages it includes may not have.

### Specify the Tag
Using fixed tags (as opposed to `:latest` ) will ensure immutability, meaning the base image will not change between builds. Note that for Dashy, the app is being actively developed, new features, bug fixes and general improvements are merged each week, and if you use a fixed version you will not enjoy these benefits. So it's up to you weather you would prefer a stable and reproducible environment, or the latest features and enhancements.

### Container Security Scanning
It's helpful to be aware of any potential security issues in any of the Docker images you are using. You can run a quick scan using Snyk on any image to output known vulnerabilities using [Docker scan](https://docs.docker.com/engine/scan/), e.g: `docker scan lissy93/dashy:latest`.

A similar product is [Trivy](https://github.com/aquasecurity/trivy), which is free an open source. First install it (with your package manager), then to scan an image, just run: `trivy image lissy93/dashy:latest`

For larger systems, RedHat [Clair](https://www.redhat.com/en/topics/containers/what-is-clair) is an app for parsing image contents and reporting on any found vulnerabilities. You run it locally in a container, and configure it with YAML. It can be integrated with Red Hat Quay, to show results on a dashboard. Most of these use static analysis to find potential issues, and scan included packages for any known security vulnerabilities.

### Registry Security
Although over-kill for most users, you could run your own registry locally which would give you ultimate control over all images, see the [Deploying a Registry Docs](https://docs.docker.com/registry/deploying/) for more info. Another option is [Docker Trusted Registry](https://docker-docs.netlify.app/ee/dtr/), it's great for enterprise applications, it sits behind your firewall, running on a swarm managed by Docker Universal Control Plane, and lets you securely store and manage your Docker images, mitigating the risk of breaches from the internet.

### Security Modules
Docker supports several modules that let you write your own security profiles.

[AppArmor](https://www.apparmor.net/)is a kernel module that proactively protects the operating system and applications from external or internal threats, by enabling you to  restrict programs' capabilities with per-program profiles. You can specify either a security policy by name, or by file path with the `apparmor` flag in docker run. Learn more about writing profiles, [here](https://gitlab.com/apparmor/apparmor/-/wikis/QuickProfileLanguage).

[Seccomp](https://en.wikipedia.org/wiki/Seccomp) (Secure Computing Mode) is a sandboxing facility in the Linux kernel that acts like a firewall for system calls (syscalls). It uses Berkeley Packet Filter (BPF) rules to filter syscalls and control how they are handled. These filters can significantly limit a containers access to the Docker Host’s Linux kernel - especially for simple containers/applications. It requires a Linux-based Docker host, with secomp enabled, and you can check for this by running `docker info | grep seccomp`. A great resource for learning more about this is [DockerLabs](https://training.play-with-docker.com/security-seccomp/).


**[⬆️ Back to Top](#management)**

---

## Web Server Configuration

_The following section only applies if you are not using Docker, and would like to use your own web server_

Dashy ships with a pre-configured Node.js server, in [`server.js`](https://github.com/Lissy93/dashy/blob/master/server.js) which serves up the contents of the `./dist` directory on a given port. You can start the server by running `node server`. Note that the app must have been build (run `yarn build`), and you need [Node.js](https://nodejs.org) installed.

If you wish to run Dashy from a sub page (e.g. `example.com/dashy`), then just set the `BASE_URL` environmental variable to that page name (in this example, `/dashy`), before building the app, and the path to all assets will then resolve to the new path, instead of `./`.

However, since Dashy is just a static web application, it can be served with whatever server you like. The following section outlines how you can configure a web server.

Note, that if you choose not to use `server.js` to serve up the app, you will loose access to the following features:
- Loading page, while the app is building
- Writing config file to disk from the UI
- Website status indicators, and ping checks

### NGINX

Create a new file in `/etc/nginx/sites-enabled/dashy`

```
server {
	listen 80;
	listen [::]:80;

	root /var/www/dashy/html;
	index index.html;

	server_name your-domain.com www.your-domain.com;

	location / {
		try_files $uri $uri/ =404;
	}
}
```
Then upload the build contents of Dashy's dist directory to that location.
For example: `scp -r ./dist/* [username]@[server_ip]:/var/www/dashy/html`

### Apache

Copy Dashy's dist folder to your apache server, `sudo cp -r ./dashy/dist /var/www/html/dashy`.

In your Apache config, `/etc/apche2/apache2.conf` add:
```
<Directory /var/www/html>
	Options Indexes FollowSymLinks
	AllowOverride All
	Require all granted
</Directory>
```

Add a `.htaccess` file within `/var/www/html/dashy/.htaccess`, and add:
```
Options -MultiViews
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^ index.html [QSA,L]
```

Then restart Apache, with `sudo systemctl restart apache2`

### cPanel
1. Login to your WHM
2. Open 'Feature Manager' on the left sidebar
3. Under 'Manage feature list', click 'Edit'
4. Find 'Application manager' in the list, enable it and hit 'Save'
5. Log into your users cPanel account, and under 'Software' find 'Application Manager'
6. Click 'Register Application', fill in the form using the path that Dashy is located, and choose a domain, and hit 'Save'
7. The application should now show up in the list, click 'Ensure dependencies', and move the toggle switch to 'Enabled'
8. If you need to change the port, click 'Add environmental variable', give it the name 'PORT', choose a port number and press 'Save'.
9. Dashy should now be running at your selected path an on a given port

**[⬆️ Back to Top](#management)**

---

## Running a Modified Version of the App

If you'd like to make any code changes to the app, and deploy your modified version, this section briefly explains how.

The first step is to fork the project on GitHub, and clone it to your local system. Next, install the dependencies (`yarn`), and start the development server (`yarn dev`) and visit `localhost:8080` in your browser. You can then make changes to the codebase, and see the live app update in real-time. Once you've finished, running `yarn build` will build the app for production, and output the assets into `./dist` which can then be deployed using a web server, CDN or the built-in Node server with `yarn start`. For more info on all of this, take a look at the [Developing Docs](/docs/developing.md).

You probably want to deploy your app with Docker, and this can be done as follows:

To build and deploy locally, first build the app with: `docker build -t dashy .`, and then start the app with `docker run -p 8080:80 --name my-dashboard dashy`.  Or modify the `docker-compose.yml` file, replacing `image: lissy93/dashy` with `build: .` and run `docker compose up`.

Your container should now be running, and will appear in the list when you run `docker container ls –a`. If you'd like to enter the container, run `docker exec -it [container-id] /bin/ash`.

You may wish to upload your image to a container registry for easier access. Note that if you choose to do this on a public registry, please name your container something other than just 'dashy', to avoid confusion with the official image.
You can push your build image, by running: `docker push ghcr.io/OWNER/IMAGE_NAME:latest`. You will first need to authenticate, this can be done by running `echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin`, where `CR_PAT` is an environmental variable containing a token generated from your GitHub account. For more info, see the [Container Registry Docs](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).

**[⬆️ Back to Top](#management)**
