---
title: Install on a server
description: Install Cahut on a Linux server in one command, which checks the signature of the release, sets up the service and its updates, and ends with the link to the first start.
---

One command installs Cahut on a Linux server as the `cahut` systemd service. It checks the signature of the release, declares your first Cahut, opens the firewall, sets up the daily updates, runs `cahut doctor`, and ends with a link to the [first start](/install/first-start/), where you create your account and connect your AI in the browser.

## Before you start

You need:

- **A Linux server with systemd**, on amd64 or arm64, where you can run commands as root. Cahut is tested on AlmaLinux 9 and 10 and Ubuntu 22.04 and 24.04, from the provider's standard image. Plan 2 vCPU, 4 GB of RAM and 40 GB of disk, the size the project aims at for its launch: today's Cahut needs less, and the disk fills with the Windows you publish.
- **A domain or a subdomain**, such as `cahut.example.com`, with an A or AAAA record that points to the server. No wildcard record is needed. Without a domain, Cahut takes the public IPv4 address of the server, and moves to a domain later: see [Without a domain](#without-a-domain).
- **Ports 80 and 443 open** to the internet, in the provider's firewall too. Port 80 answers the challenges of Let's Encrypt and redirects to HTTPS. Port 443 serves everything else: the Windows, MCP, OAuth, and the console under `/console/`. Cahut holds both ports: to share them with another site of the server, put a proxy in front of it, as [Behind a proxy](/admin/service/#behind-a-proxy) explains.
- **curl** and **OpenSSL 3**, which the standard images of AlmaLinux 9 and Ubuntu 22.04 and later have. `install.sh` checks the Ed25519 signature of the release with it: OpenSSL 1.1, on Ubuntu 20.04 and AlmaLinux 8, cannot.

Create the DNS record first: the service asks Let's Encrypt for the certificate as soon as it starts.

## Run the installer

```bash
curl -fsSL https://cahut.cloud/install.sh | sudo sh -s -- --domain cahut.example.com
```

Add `--email you@example.com` for Let's Encrypt to write to you about your certificates. The script says what it does, runs [cahut doctor](/admin/doctor/), and prints the link to the first start:

```text
cahut: downloading Cahut 1.1.1 for linux/amd64
cahut: signature and checksum checked
cahut: creating the cahut system user
cahut: writing /etc/cahut/cahut.env
cahut: declaring the Cahut of cahut.example.com
cahut: opening ports 80 and 443 in firewalld
cahut: cahut 1.1.1 (commit 06fd933, 2026-10-31T17:44:32Z, go1.27.3 linux/amd64) is running
cahut: waiting for the certificate of cahut.example.com

ok    System             AlmaLinux 10.2 (Lavender Lion), amd64.
ok    Clock              Synchronized.
ok    Service            The cahut service runs, and starts at boot.
ok    Service            Cahut 1.1.1 answers, with 1 Cahut.
…
ok    cahut.example.com  Its certificate, from Let's Encrypt, is valid until 22 January 2027.
ok    cahut.example.com  https://cahut.example.com/ answers.
…
No problem.

  https://cahut.example.com/

Open this link to start your Cahut. It works once, until 2026-10-25, and is shown only now.

  https://cahut.example.com/console/setup#k7n3…

It creates your account, and that of the administrator of this server
while it has none, with the same password.
For a new link: sudo -u cahut /usr/local/bin/cahut setup
Then, the console of the server: https://cahut.example.com/server/
```

Open the link in your browser: [The first start](/install/first-start/) walks through it.

The script reads the releases of Cahut on GitHub, without an account. To install from a private fork, name it in `CAHUT_REPO` and give the script a token in `GITHUB_TOKEN`, on the command line of `sudo`: it keeps the token for the updates, readable by root only.

## Without a domain

To try Cahut before you have a domain, give the public IPv4 address of the server instead:

```bash
curl -fsSL https://cahut.cloud/install.sh | sudo sh -s -- --ip 203.0.113.7
```

With neither `--domain` nor `--ip`, the script takes the address of the server's default route, when it is public. Behind NAT, the server does not know its public address: give it with `--ip`.

The Cahut then answers at `https://203.0.113.7/`, with a certificate of Let's Encrypt for that address. Such certificates last six days, and the service renews them every few days: port 80 must stay open. IPv6 addresses are not supported, since the policy of a Window could not name them.

ChatGPT does not connect to a Cahut at an IP address: move the Cahut to a domain before you add it to ChatGPT.

When you have a domain, point it to the server, then move the Cahut there, in the [console of the server](/admin/console/#the-console-of-the-server) or on the command line:

```bash
sudo -u cahut /usr/local/bin/cahut domain rename 203.0.113.7 cahut.example.com
```

Everything in the Cahut stays. The links that name the address stop working, and AIs connect again at the new one.

## What the installer does

1. It downloads the release for the server's architecture from GitHub, the latest one unless you name another, with its checksum file and the signature of that file. It checks the signature against the keys of Cahut written in the script, then the checksum of the archive: a release that the project did not sign does not install. With `--binary`, it takes the binary you give instead.
2. It checks that the binary runs on the server.
3. It creates the `cahut` system user, with no shell, and the data folder `/var/lib/cahut`, which only that user and its group can read. When the server already has an account named `cahut` that a person signs in with, the script stops before anything else: the service would run with that person's rights. Sign in with another account, rename or delete that one, then run the script again.
4. It writes the settings file `/etc/cahut/cahut.env` when its content changes, keeping the values you do not give again.
5. It writes the systemd units of the service and of the updater, and installs the binary as `/usr/local/bin/cahut`. On a server with SELinux, it restores the labels of the files.
6. It declares the Cahut of `--domain` or `--ip`, unless the server holds it already.
7. It opens ports 80 and 443 in firewalld or ufw, when one of them is active.
8. It enables and restarts the service, checks that it runs, and enables the daily update.
9. It waits up to half a minute for the certificate of a new Cahut, runs `cahut doctor`, and prints the link to the first start when the Cahut has nobody yet.

The service runs as the `cahut` user, with the one capability it needs to listen on ports 80 and 443, and can write only to its data folder. See [The cahut service](/admin/service/).

## Options

| Option | What it does |
| --- | --- |
| `--domain DOMAIN` | Declares the Cahut of `DOMAIN`, such as `cahut.example.com`. It must point to this server. |
| `--ip ADDRESS` | Declares a Cahut at the public IPv4 address of the server, without a domain. |
| `--email EMAIL` | Sets the contact of the Let's Encrypt account. Optional. |
| `--staging` | Uses the staging environment of Let's Encrypt, for tests. Browsers do not trust its certificates. |
| `--version TAG` | Installs this release, such as `v0.4.0`, instead of the latest. Releases are signed from v0.4.0 on. |
| `--binary PATH` | Installs this `cahut` binary instead of downloading a release. |
| `--updates POLICY` | What the daily update does: `patch` (the default), `all`, `check` or `off`. See [Updates](/admin/updates/). |
| `--help` | Prints the options. |

Two environment variables choose where releases come from: `CAHUT_REPO`, the GitHub repository of the releases, `cahutcloud/cahut` by default, and `GITHUB_TOKEN`, a token that can read the releases of a private repository.

## Add a Cahut

Point the DNS record of the new domain to the server, then declare it:

```bash
sudo -u cahut /usr/local/bin/cahut domain add docs.example.org
sudo -u cahut /usr/local/bin/cahut setup --domain docs.example.org
```

The running service gets the certificate at the first HTTPS connection to that domain. The second command prints the link to the first start of the new Cahut; its first person becomes its administrator.

## Update Cahut

The server updates itself every day, as [Updates](/admin/updates/) explains. To update now:

```bash
sudo /usr/local/bin/cahut update
```

Running the script again installs the latest release too, without the health check and the rollback of `cahut update`: use it to change the settings.

## Check that it runs

```bash
sudo /usr/local/bin/cahut doctor
```

[cahut doctor](/admin/doctor/) checks the service, the firewall, SELinux, the address and the certificate of each Cahut, and says what to fix. The address of a Cahut answers with one line:

```text
This is cahut.example.com, a Cahut. AIs connect to it over MCP at /mcp.
```

If the service does not start, its logs say why: `journalctl -u cahut`.

## Remove Cahut

There is no uninstaller yet. To remove the service and keep the data:

```bash
sudo systemctl disable --now cahut cahut-update.timer cahut-update.path
sudo rm /etc/systemd/system/cahut.service /etc/systemd/system/cahut-update.*
sudo rm /usr/local/bin/cahut
sudo rm -r /etc/cahut /var/lib/cahut-update
sudo systemctl daemon-reload
```

`/var/lib/cahut` then still holds every Cahut, its Windows and its Logbook, and the `cahut` user still exists. Deleting that folder deletes them for good.
