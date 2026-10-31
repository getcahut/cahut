---
title: Administer from the command line
description: Declare Cahuts, create people and tokens, and read the Logbook with the cahut command, on the server.
sidebar:
  label: Command line
---

Most of the administration happens in [the console](/admin/console/). The `cahut` command, on the server, declares Cahuts, creates the first accounts, and gives a new link to anyone locked out. Each command runs an operation of the registry as the administrator, and its changes go to the Logbook.

## Run the commands

Run them as the `cahut` user, which owns the data, and give the full path of the binary:

```bash
sudo -u cahut /usr/local/bin/cahut domain list
```

On AlmaLinux, `sudo` does not search `/usr/local/bin`, hence the full path. The commands read the data in `/var/lib/cahut`: `--data`, or the variable `CAHUT_DATA`, points them to another folder.

When the server holds several Cahuts, the commands that act in one of them need `--domain`. With a single Cahut, they act in it.

The [command line reference](/reference/cli/) lists every command and flag.

## Declare a Cahut

Point the DNS record of the domain to the server first, then:

```bash
sudo -u cahut /usr/local/bin/cahut domain add docs.example.org
sudo -u cahut /usr/local/bin/cahut domain list
```

```text
Declared docs.example.org: its Cahut is ready. Point its DNS to this server, then serve it.
ID  DOMAIN             CREATED     REPORTS  STATE
1   cahut.example.com  2026-10-20  0        active
2   docs.example.org   2026-10-21  0        active
```

`REPORTS` counts the reports about the Windows of each Cahut that wait for its [moderators](/admin/moderation/). The running service serves the new Cahut at once, and gets its certificate at the first HTTPS connection to its domain. A domain is a host name such as `cahut.example.com`; a Cahut without a domain takes the public IPv4 address of the server instead, such as `203.0.113.7`. There is no command yet to remove a Cahut.

`cahut setup` prints the link to the [first start](/install/first-start/) of a Cahut that has nobody yet, where its first person chooses their account:

```bash
sudo -u cahut /usr/local/bin/cahut setup --domain docs.example.org
```

While the server has no administrator, the first person of any of its Cahuts also becomes the administrator of the server: the command says so, and the link is then for the person who looks after the whole server.

## Move a Cahut to another domain

```bash
sudo -u cahut /usr/local/bin/cahut domain rename 203.0.113.7 cahut.example.com
```

Point the DNS record of the new domain to the server first. Everything in the Cahut stays; the links that name its old address stop working, the sessions of administrators there end, and AIs connect again at the new address. The running service follows at once.

`--admin-handle` and `--admin-name` create the first administrator of the new Cahut at the same time, a person of it, and print their activation link, as [`user create`](#create-a-person) does:

```bash
sudo -u cahut /usr/local/bin/cahut domain add docs.example.org --admin-handle ana --admin-name "Ana Lima"
```

## Set the kind and limits of a Cahut

A Cahut is collaborative and has no limit until you say otherwise. `settings set` changes the kind or the limits you name and leaves the rest as it is; `none` lifts a limit:

```bash
sudo -u cahut /usr/local/bin/cahut settings set --domain docs.example.org --people 20 --disk 10GB
sudo -u cahut /usr/local/bin/cahut settings set --domain docs.example.org --disk none
sudo -u cahut /usr/local/bin/cahut settings show --domain docs.example.org
```

```text
Settings of docs.example.org:

Kind: collaborative

                  HOLDS   LIMIT
people            3       20
administrators    1       none
team Rooms        2       none
Windows           14      none
Windows per Room          none
disk              2.4 MB  none

Legal pages:

              ADDRESS  IN FRENCH
legal notice  none
terms         none
privacy       none
support       none
```

The disk counts in units of 1000 bytes, as hosts sell it: `10GB` is 10 billion bytes. [Limits](/concepts/cahuts/#limits) says what each limit counts.

`--kind individual` makes the Cahut [individual](/concepts/cahuts/#collaborative-or-individual), once no Room has more than one member and no Window opens to the whole Cahut. `--rooms-per-person`, `--windows-per-person` and `--disk-per-person` then bound each person; going back to `--kind collaborative` lifts them:

```bash
sudo -u cahut /usr/local/bin/cahut settings set --domain docs.example.org --kind individual --windows-per-person 50 --disk-per-person 1GB
```

## Link the legal pages of a Cahut

The [legal pages](/concepts/cahuts/#legal-pages) of a Cahut are the addresses of its legal notice, terms, privacy policy and support, which the console and the pages for readers link at their foot. `settings links` sets those you name and leaves the rest as it is; `none` removes a link, with its address in French:

```bash
sudo -u cahut /usr/local/bin/cahut settings links --domain docs.example.org \
  --legal-notice https://example.org/legal --terms https://example.org/terms --privacy https://example.org/privacy
sudo -u cahut /usr/local/bin/cahut settings links --domain docs.example.org --legal-notice-fr https://example.org/fr/mentions-legales
sudo -u cahut /usr/local/bin/cahut settings links --domain docs.example.org --privacy none
```

```text
Set the legal pages of docs.example.org:

              ADDRESS                    IN FRENCH
legal notice  https://example.org/legal  https://example.org/fr/mentions-legales
terms         https://example.org/terms
privacy       none
support       none
```

Each address starts with `https://`. An address in French is for a page whose French version differs: the console links it when it speaks French, and needs the address of the same page for the other languages. Without a flag, `settings links` prints the links; `settings show` prints them too.

## Set the address of moderation of a Cahut

Each report about a Window of a Cahut goes, when it sends e-mail, to its administrators who have an address, and to its address of moderation, such as an abuse mailbox of the host: the administrators of the server have no address of their own. See [Moderation](/admin/moderation/). `settings moderation-address` sets it, prints it without a flag, and `none` removes it:

```bash
sudo -u cahut /usr/local/bin/cahut settings moderation-address --domain docs.example.org --email abuse@example.org
```

```text
The reports of docs.example.org now go to abuse@example.org too, when it sends e-mail.
```

## Suspend a Cahut

After a decision of moderation, or when its organization stops paying, the administrators of the server [suspend a Cahut](/concepts/cahuts/#suspension), with a reason, and resume it later:

```bash
sudo -u cahut /usr/local/bin/cahut domain suspend docs.example.org --reason "Invoice of October unpaid"
sudo -u cahut /usr/local/bin/cahut domain list
sudo -u cahut /usr/local/bin/cahut domain resume docs.example.org
```

```text
The Cahut of docs.example.org is suspended since 2026-10-29 10:42: it keeps everything, and serves nothing but to the administrators of the server.
Its administrators hear of it by e-mail when it has a relay. Resume it with cahut domain resume docs.example.org.
ID  DOMAIN             CREATED     REPORTS  STATE
1   cahut.example.com  2026-10-20  2        active
2   docs.example.org   2026-10-21  0        suspended since 2026-10-29: Invoice of October unpaid
The Cahut of docs.example.org is open again: its Windows, its people and their AIs find everything as it was.
```

The reason holds 500 characters at most. Suspending a suspended Cahut again changes its reason only. The running service follows at once, and both commands go to the Logbook of the Cahut, with the reason.

## Create a person

```bash
sudo -u cahut /usr/local/bin/cahut user create --domain cahut.example.com --handle ana --name "Ana Lima"
```

```text
Created ana (Ana Lima), member of cahut.example.com, with a personal Room at cahut.example.com/ana.

Send ana this link, where they choose their password. It works once, until 2026-10-29, and is shown only now.

https://cahut.example.com/console/activate#...
```

The handle names the person and their personal Room: 2 to 32 lowercase letters, digits and hyphens, starting with a letter and ending with a letter or a digit. It must be free, as the name of a person and of a Room, and cannot be `mcp`, `api`, `admin`, `console`, `help` or `oauth`. `--admin` makes the person an administrator of the Cahut.

The link opens the console of the Cahut, under `/console/`, where the person chooses a password of 15 characters at least, then is signed in. It works once, for 7 days. `--email` gives the person an address, where the link goes too when the Cahut [sends e-mail](/admin/mail/). An administrator also creates people in the console, under **People**.

## List people and give a new link

```bash
sudo -u cahut /usr/local/bin/cahut user list --domain cahut.example.com
sudo -u cahut /usr/local/bin/cahut user link --domain cahut.example.com --handle ana
```

```text
HANDLE   NAME            ROLE    STATUS
ana      Ana Lima        member  pending
olivier  Olivier Durand  admin   active
```

A person is `pending` until they choose a password, then `active`, or `disabled`. `user link` gives a new link to a person who lost theirs or forgot their password: it replaces their previous link, and their password, if they have one, works until they use the new one. It also works for administrators, so that the command line is the way back in when nobody can sign in.

`user email --handle ana --email ana@example.com` gives a person an address, or removes it with `--email ""`; their old address gets a notice. Changing a person's role, and disabling or enabling a person, happen in the console, under **People**.

## Set the e-mail relay

```bash
printf '%s' "$SMTP_PASSWORD" | sudo -u cahut /usr/local/bin/cahut mail set --host smtp.example.com \
  --user cahut@example.com --password-stdin --from cahut@example.com
sudo -u cahut /usr/local/bin/cahut mail test --to you@example.com
```

Only a Cahut has a relay: without one, it sends no e-mail. On a server that holds several Cahuts, `--domain` names the Cahut. `mail show` prints the relay, `mail remove` removes it. See [E-mail](/admin/mail/).

## Administrators of the server

The administrators of the server have accounts of their own, apart from the people of the Cahuts. In the [console of the server](/admin/console/#the-console-of-the-server), under `/server/` of any domain of the server, they declare Cahuts, set what the server sets for each, moderate them, and read the Logbook of the server. From the domain where they signed in, they reach every Cahut. The people and the Rooms of each Cahut are for its own administrators.

```bash
sudo -u cahut /usr/local/bin/cahut admin create --handle olivier --name "Olivier Durand"
sudo -u cahut /usr/local/bin/cahut admin list
sudo -u cahut /usr/local/bin/cahut admin link --handle olivier
```

```text
Created olivier (Olivier Durand), administrator of this server.

Send olivier this link, where they choose their password. It works once, until 2026-10-29, and is shown only now.

https://cahut.example.com/server/activate#...
```

The link opens the console of the server, at the domain of the first Cahut declared, or of the one `--domain` names. `admin link` gives a new link to an administrator who lost theirs or forgot their password. The command line itself needs no account: whoever runs it on the server acts as an administrator of the server, everywhere.

An administrator of the server holds every Cahut of the server. To make someone an administrator of one Cahut only, create them as a person of it with [`user create --admin`](#create-a-person), or make them an administrator in its console, under **People**.

`admin disable` disables an administrator of the server: their sessions and their link end, and they cannot sign in until `admin enable` enables them again. The server keeps at least one administrator who is not disabled, and an administrator cannot disable themselves in the console.

```bash
sudo -u cahut /usr/local/bin/cahut admin disable --handle camille
sudo -u cahut /usr/local/bin/cahut admin enable --handle camille
```

`admin rename` changes the name of an administrator of the server. Their handle stays: they sign in with it. In the console, each administrator changes their own name, or another's, under **Manage**.

```bash
sudo -u cahut /usr/local/bin/cahut admin rename --handle camille --name "Camille Martin"
```

## Manage tokens

```bash
sudo -u cahut /usr/local/bin/cahut token create --domain cahut.example.com --user ana --name "Claude Code"
sudo -u cahut /usr/local/bin/cahut token list --domain cahut.example.com
sudo -u cahut /usr/local/bin/cahut token revoke --domain cahut.example.com --id 3
```

[Personal tokens](/connect/tokens/) says what a token allows and how to keep it.

## Read the Logbook

```bash
sudo -u cahut /usr/local/bin/cahut logbook --domain cahut.example.com --limit 50
sudo -u cahut /usr/local/bin/cahut logbook --server
```

The first command prints the latest entries of a Cahut's Logbook, the second those of the server. See [The Logbook](/concepts/knock-first-and-logbook/#the-logbook).

## Print the version

```bash
/usr/local/bin/cahut version
```

```text
cahut 1.1.1 (commit 06fd933, 2026-10-31T17:44:32Z, go1.27.3 linux/amd64)
```

## Update and diagnose

`sudo /usr/local/bin/cahut update` installs a new release, as root: see [Updates](/admin/updates/). `sudo /usr/local/bin/cahut doctor` checks the server: see [cahut doctor](/admin/doctor/). These two run as root, and never open the databases; the other commands run as the `cahut` user.

## When a command fails

A command that fails says why on its error output, and exits with code 1; a wrong flag or argument exits with code 2. When the input is at fault, the message says what to change:

```text
cahut: invalid input: handle "Ana": use 2 to 32 lowercase letters, digits and hyphens, starting with a letter and ending with a letter or a digit
```
