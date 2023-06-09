# SvelteKit systemd Guide

This repository contains a GitHub workflow with a `build` job that builds your SvelteKit into a very minimal systemd portable service and a `deploy` job that can upload and start the container on your server. The image built by the `build` job only contains your SvelteKit app, a Node.js executable and libc++. The total size of the final image is around 37 MB.

1. [How to use this](#how-to-use-this)
2. [The service unit file](#the-service-unit-file)
3. [Frequently used commands](#frequently-used-commands)
   - [portablectl](#portablectl)
   - [systemctl](#systemctl)
   - [journalctl](#journalctl)
4. [Frequently asked questions](#frequently-asked-questions)
   - [What is systemd?](#what-is-systemd)
   - [What is a systemd portable service?](#what-is-a-systemd-portable-service)
   - [Where can I get a VPS and how do I set it up?](#where-can-i-get-a-vps-and-how-do-i-set-it-up)
   - [How do I expose my app to the internet?](#how-do-i-expose-my-app-to-the-internet)

## How to use This

To use this repository, you have two options:

1. [Use it as a template](https://github.com/karimfromjordan/sveltekit-systemd/generate)
2. Start from scratch:
   - Create a new SvelteKit project using `pnpm create svelte@latest myapp`.
   - Install `@sveltejs/adapter-node` and add it to your `svelte.config.js`.
   - Copy and paste the `.github` directory from this repository into the root directory of your project.

In the `Settings` tab of your repository on GitHub add the following secrets.

- `SSH_HOST`: the IP address or a domain pointing to the IP address of the VPS you want to deploy to
- `SSH_USER`: the user to use for logging into the VPS
- `SSH_PRIVATE_KEY`: the private key to use for logging into the VPS

**Note:** The user needs to have permission to run the `portablectl` command. If you use the cloud init script provided further down below this is automatically setup for you. Otherwise make sure the user has at least `ALL=(ALL) NOPASSWD:/usr/bin/portablectl` set.

After setting up your repository, every time you push changes to the `main` branch, a `build` job will package your app as a systemd portable service followed by a `deploy` job that will upload the image to your VPS and start it using the systemd `portablectl` tool. You can also download the image from the `Actions` tab in your repository.

**Note:** This example uses `pnpm`. If you prefer `npm`, you will have to make some adjustments in `.github/workflows/deploy.yml`.

## The service unit file

The `build` job in this GitHub workflow creates a systemd service unit inside the image. The unit will be named `<reponame>.service` and by default looks like this:

```ini
[Unit]
Description=$GITHUB_REPOSITORY
StartLimitBurst=3
StartLimitIntervalSec=60

[Service]
DynamicUser=yes
StateDirectory=services/${GITHUB_REPOSITORY##*/}
Environment=PORT=5001 NODE_ENV=production
ExecStart=/opt/node /opt/app/build
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

This does the following:

- Your application will run under a dynamically allocated and restricted user. By default, systemd assigns the user the same name as your service file, which is set to your repository name by the workflow.
- The environment variables `PORT=5001` and `NODE_ENV=production` are passed to your application. You have the ability to override existing environment variables on your system or add additional ones.
- An environment variable named `STATE_DIRECTORY` will be provided to your application. It points to a directory where your SvelteKit app can write any data to. In the host system, this directory can be found at `/var/lib/private/services/<your-reponame>`. To back up the state of all your applications, simply copy the `/var/lib/private/services` directory.
- Your application is automatically restarted in the event of a crash. The service unit is configured to allow a maximum of 3 restart attempts within a 60-second timeframe.
- The logs for your app will be collected by the systemd journal. To view and filter the logs refer to the journalctl section further down below.

In general, there is no need to make any modifications to the service unit or workflow. However, if you are interested in exploring additional options provided by systemd, checkout the systemd documentation.

## Frequently used commands

Below is an explanation of all commands you need in order to manage applications deployed to a server using systemd. In most cases you need to prefix every command with `sudo` to gain higher priviliges and be able to execute them.

### portablectl

This command is used to manage systemd containers. It's part of the `systemd-container` package.

```bash
portablectl attach/detach/reattach ./container.raw
# Attach container, enable it on boot and start all units
portablectl attach \
  --enable \
  --now \
  --profile=trusted \
  ./container.raw
```

- `--enable`: This flag will setup the dependencies listed in the `[Install]` section of your service unit and thus make sure that your app is automatically started on boot.
- `--now`: This flag will immidiately start your app after the container has been attached.
- `--profile`: By default systemd containers inherit default options that further restrict resources for security reasons. The default profile however is too restrictive to run Node.js apps so we use the `trusted` profile instead.

### systemctl

This command is used to manage units.

```bash
# Show status of unit
systemctl status <reponame>.service
# Start/stop/restart/reload unit
systemctl start/stop/restart <reponame>.service
# Show, edit or revert unit configuration
systemctl cat/edit/revert <reponame>.service
```

`systemctl edit` will create a so called drop-in file and open it in an editor. Any systemd configuration you put into the drop-in will overwrite settings in the default service unit provided by the image. For example, to change the port your app runs on call `systemctl edit <reponame>.service` and type the following into the editor.

```ini
[Service]
Environment=PORT=3333
```

Make sure to restart your app after editing the configuration using `systemctl restart <reponame>.service`. Additional environment variables can be set by appending them separated with a space or by specifying the `Envrionment` option multiple times.

```ini
[Service]
Environment=PORT=3333 MY_VAR=123
Environment=ANOTHER_VAR=xyz
```

The `cat` command outputs all configuration files including your drop-in file and the portable profile used which are applied on top of your default unit file.

### journalctl

This command is used to view logs.

```bash
journalctl -u <reponame>.service -S yesterday -U today
```

Parameters:

- `-u/--unit=`: the unit you want to view logs for
- `-S/--since=`: start date
- `-U/--until=`: end date

Both `-S/--since=` and `-U/--until=` accept a wide range of values.

Example values for `-S/--since=` and `-U/--until=`:

- `today`, `yesterday`
- `-2h`, `-2d`

Example commands:

```bash
# Show all logs of the last two hours for `<reponame>.service`
journalctl -u <reponame>.service -S -2h
# Show all logs since yesterday
journalctl -u <reponame>.service -S today
```

## Frequently asked questions

### What is systemd?

Systemd is the native process manager used in most Linux distributions including Debian, Ubuntu, and Fedora. It is responsible for initializing the system, as well as executing and monitoring all installed applications. To manage applications and their dependencies, systemd utilizes a well-defined configuration system based on unit files. `.service` unit files are for executed apps. Another useful unit type are `.timer` units which allow you to schedule `.service` units based on various time parameters. They are similar to cron jobs but more flexible.

### What is a systemd portable service?

> systemd (since version 239) supports a concept of “Portable Services”. “Portable Services” are a delivery method for system services that uses two specific features of container management: 

> 1. Applications are bundled. I.e. multiple services, their binaries and all their dependencies are packaged in an image, and are run directly from it.

> 2. Stricter default security policies, i.e. sand-boxing of applications.

See the [official documentation](https://systemd.io/PORTABLE_SERVICES/) for more information. 

### Where can I get a VPS and how do I set it up?

There are several options available. Some popular VPS providers are:

- [Hetzner](https://www.hetzner.com/cloud?country=ot)
- [DigitalOcean](https://www.digitalocean.com/products/droplets)
- [Linode](https://www.linode.com/products/shared)
- [Vultr](https://www.vultr.com/products/cloud-compute)
- [Alwyzon](https://www.alwyzon.com/en/virtual-servers)
- [Contabo](https://contabo.com/en)

Most VPS providers, such as Hetzner and DigitalOcean, allow you to paste in a cloud init/user data script when creating a new VPS on their website. Cloud config is a YAML and shell script based way of setting up your server on first boot. You can find the configuration I use here.

**[VPS cloud config](https://gist.github.com/karimfromjordan/06dbe35f86468c74dfa89c75a3d00a8c)**

Make sure to replace the placeholders `<GITHUB_USERNAME>`, `<DOMAIN_NAME>` and `<HOSTNAME>` with your respective values:

- `<GITHUB_USERNAME>` will be used to import your public SSH key from GitHub so that you can log into your VPS straight away with your primary key.
- `<DOMAIN_NAME>` can be any domain or subdomain you want to use to identify your server.
- `<HOSTNAME>` is similar `<DOMAIN_NAME>` and can be any name of your choice to identify your VPS

### How do I expose my app to the internet?

Once you have deployed your SvelteKit app you still need to expose it so that it can send and receive requests from the internet. It is recommend to use a webserver/reverse proxy application for this. A webserver such as [Caddy](https://caddyserver.com/docs/) or [Nginx](https://nginx.org/en/docs/) not only allows you to host multiple apps on a single VPS but can also manage SSL certificates and other settings for you. Caddy is simpler to configure, provides automatic HTTPS by default and very fast in implementing new features such as support for HTTP3. If you use the [VPS cloud config](https://gist.github.com/karimfromjordan/06dbe35f86468c74dfa89c75a3d00a8c) mentioned in the previous section, Caddy will be installed automatically. You also need to point your domain(s) to the IP address of your VPS. The configuration file can be found at `/etc/caddy/Caddyfile`. To edit it run `nano /etc/caddy/Caddyfile`. A minimal `Caddyfile` for your SvelteKit app would look like this:

```Caddyfile
example.com {
  reverse_proxy :5001
}
```

After that reload Caddy using `systemctl reload caddy.service`

To run multiple applications on your VPS, assign each app a unique `PORT` number using the `systemctl edit <reponame>.service` command and add a server block for each app in your `Caddyfile`.
