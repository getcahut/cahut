---
title: The cahut service
description: Where Cahut keeps its files, how the systemd service runs, its settings, logs, certificates and health check, and how to back it up.
---

`install.sh` installs Cahut as the `cahut` systemd service. The service runs `cahut serve`, which serves every Cahut of the server: their MCP endpoints and their Windows.

## Files

| Path | What it holds |
| --- | --- |
| `/usr/local/bin/cahut` | The binary |
| `/etc/cahut/cahut.env` | The settings of the service |
| `/etc/cahut/update.env` | The token that reads the releases of a private fork, when the installer got one, for root only |
| `/etc/systemd/system/cahut.service` | The systemd unit |
| `/etc/systemd/system/cahut-update.*` | The updater: its service, its daily timer, and the path unit that starts it when the console asks |
| `/var/lib/cahut/` | The data: the records of the server, every Cahut with its Windows, and the certificates |
| `/var/lib/cahut-update/` | The state of updates, the previous binary and the copy of the databases before the last update |

[Cahuts and servers](/concepts/cahuts/#on-disk) shows what the data folder holds.

## Settings

`/etc/cahut/cahut.env` sets the environment of the service:

```sh
CAHUT_DATA=/var/lib/cahut
CAHUT_ACME_EMAIL=you@example.com
CAHUT_ACME_CA=
# What the daily update does: patch, all, check or off.
CAHUT_UPDATES=patch
# The admin API, for the programs of this machine with a service token: off when empty.
CAHUT_ADMIN_API=
# The proxies in front of Cahut, such as 127.0.0.1, whose PROXY protocol it
# reads on HTTPS and on HTTP: off when empty. Set them in this file.
CAHUT_HTTPS_PROXY_PROTOCOL=
CAHUT_HTTP_PROXY_PROTOCOL=
```

| Variable | What it sets |
| --- | --- |
| `CAHUT_DATA` | The data folder |
| `CAHUT_ACME_EMAIL` | The contact of the Let's Encrypt account, optional |
| `CAHUT_ACME_CA` | Empty for Let's Encrypt, `staging` for its staging environment, or the address of another ACME directory |
| `CAHUT_UPDATES` | What the daily update does. See [Updates](/admin/updates/) |
| `CAHUT_ADMIN_API` | The address of the machine, such as `127.0.0.1:8090`, where programs with a service token call the [admin API](/reference/http-api/#admin-api); off when empty |
| `CAHUT_HTTPS_PROXY_PROTOCOL` | The IP addresses or prefixes of the proxies, such as `127.0.0.1` or `10.0.0.0/8`, separated by commas, whose PROXY protocol the HTTPS listener reads; off when empty. See [Behind a proxy](#behind-a-proxy) |
| `CAHUT_HTTP_PROXY_PROTOCOL` | The same, for the HTTP listener |

After a change to the file, restart the service: `sudo systemctl restart cahut`. `install.sh` writes the file again when you pass it `--email`, `--staging`, `--updates` or `--admin-api`, and keeps the values you do not give again.

## The service

```bash
sudo systemctl status cahut
sudo systemctl restart cahut
```

The service runs as the `cahut` user and restarts 5 seconds after a failure. It listens on port 80, which answers the challenges of Let's Encrypt and redirects to HTTPS, and on port 443, for the Windows, MCP, OAuth, and the console and its [API](/reference/http-api/) under `/console/`. When it stops, it lets the requests in progress finish, for up to 10 seconds.

The unit limits what the service can do: it keeps one capability, to listen on ports below 1024, cannot gain others, sees the system as read-only except its data folder, and has no access to home folders, devices or kernel settings.

## Behind a proxy

The service holds ports 80 and 443. When another site of the server needs them too, a proxy in front of Cahut can share them, as long as it leaves TLS to Cahut: on port 443, a proxy that passes each connection as it comes, such as nginx `stream`, HAProxy in TCP mode or the load balancer of a cloud provider. Cahut keeps its certificates, and gets them as before. Move the service to local ports in an override of its unit:

```ini
# /etc/systemd/system/cahut.service.d/behind-a-proxy.conf
[Service]
ExecStart=
ExecStart=/usr/local/bin/cahut serve --https 127.0.0.1:8443 --http 127.0.0.1:8080
```

Behind the proxy, every connection comes from the proxy. For Cahut to count its limits and write its logs by the address of each client, the proxy passes that address with the PROXY protocol, and Cahut trusts the proxy, in `/etc/cahut/cahut.env`:

```sh
CAHUT_HTTPS_PROXY_PROTOCOL=127.0.0.1
```

```nginx
# /etc/nginx/nginx.conf
stream {
    server {
        listen 443;
        listen [::]:443;
        proxy_pass 127.0.0.1:8443;
        proxy_protocol on;
    }
}
```

To share port 443 with another site, nginx routes each connection by the name that the client asks for, with `ssl_preread on` and a `map` of `$ssl_preread_server_name`.

- From a trusted address, every connection must start with a header of the PROXY protocol, version 1 or 2. Cahut closes the others, and logs why: change the proxy and Cahut together.
- From any other address, Cahut reads no header, and serves the connection under its own address.
- So trust only the addresses that reach Cahut through the proxy. With the proxy on the same machine, Cahut listens on 127.0.0.1, where only the programs of the machine connect.

Port 80 answers the challenges of Let's Encrypt and redirects to HTTPS. A proxy that routes it by the `Host` header passes the requests to Cahut over HTTP, and cannot send the PROXY protocol: leave `CAHUT_HTTP_PROXY_PROTOCOL` empty, and the clients of port 80 keep the address of the proxy, which counts nothing there. The [health check](#health-check) of the updater goes through that proxy too: let it pass from the machine only, since Cahut sees every client of port 80 at 127.0.0.1.

```nginx
server {
    listen 80;
    listen [::]:80;
    location = /-/healthz {
        allow 127.0.0.1;
        allow ::1;
        deny all;
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
    }
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
    }
}
```

When a proxy passes port 80 as it comes, with the PROXY protocol, as a load balancer can, set `CAHUT_HTTP_PROXY_PROTOCOL` too. Never trust 127.0.0.1 there while Cahut holds port 80 itself: the updater calls it without a header, and every update would roll back.

[cahut doctor](/admin/doctor/) names the proxies that the service trusts, and says when the last connection of one came without a valid header.

## Logs

The service writes to the journal:

```bash
journalctl -u cahut -f
```

At start, it names the Cahuts it serves, and the proxies it trusts. It logs the errors that it hides from the AIs, with the Cahut, the tool and the person, and its certificates, on lines named `certificates`. The actions of AIs and people are in the [Logbook](/concepts/knock-first-and-logbook/#the-logbook), not in the journal.

## Certificates

The service gets a certificate from Let's Encrypt for the domain of each Cahut, and renews it. A Cahut at the IP address of the server gets a certificate for that address, which lasts six days and is renewed every few days. It obtains those of the Cahuts it knows when it starts, and that of a Cahut declared while it runs at the first HTTPS connection to its domain. When Let's Encrypt fails, for instance because the DNS record does not point to the server yet, the service tries again in the background.

It asks for no certificate for a name that no Cahut declares. The certificates and the keys of the ACME account are in `/var/lib/cahut/certificates`.

## Health check

Every Cahut answers `ok` at `/-/healthz` while the service runs:

```bash
curl https://cahut.example.com/-/healthz
```

On the server itself, port 80 answers at a loopback address with the version and the state of every Cahut, which it opens with their migrations. The updater checks a new version so, and [cahut doctor](/admin/doctor/) reads it:

```bash
curl http://127.0.0.1/-/healthz
```

```json
{"version":"1.1.1","ok":true,"cahuts":[{"domain":"cahut.example.com","ok":true}]}
```

A [suspended Cahut](/concepts/cahuts/#suspension) still answers `ok` at `/-/healthz`, and opens as any other: the health of the server adds `"suspended":true` to it, and does not fail an update.

## Back up

Cahut has no backups of its own yet: they come later, to S3 or SFTP storage. Until then, take snapshots of the server at your provider, or copy the data folder while the service is stopped:

```bash
sudo systemctl stop cahut
sudo tar -C /var/lib -czf cahut-backup.tar.gz cahut
sudo systemctl start cahut
```

Stop the service first: a copy of a database in use may not open. The folder holds the data of each Window too, in `windows/<id>/data/`, with its copies.

## Update

The server updates itself every day, with a rollback when a new version fails: see [Updates](/admin/updates/).
