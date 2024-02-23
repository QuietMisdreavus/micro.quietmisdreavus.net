---
layout: post
title: running mastodon on arch linux
description: notes on setting up a Mastodon server on Arch instead of Ubuntu
categories: code
---

Here's something i wanted to write a long time ago. I think it's more relevant now, given the
renewed popularity of Mastodon.

I've been on "the Fediverse" for a few years now. I think i originally signed up back in early 2017;
i don't remember the exact reason for the flood of users then, but i've had an account for about
that long. A couple years ago i decided to set up my own instance - initially so that i could be
sure i was somewhere that wouldn't be outside of my control, but also with the hope of having a
local instance for a group of friends. The latter didn't quite pan out, but i stuck around now that
i had my own little slice of the network.

Note that setting up your own Mastodon server is not without its issues! It's a fair amount of
upkeep just to make sure the thing stays running. That doesn't even get into inter-instance
moderation, especially if more people than just you are going to have accounts there. Frankly,
content moderation is a harder ongoing task in comparison to just administering a server! I like the
occasional recreational systems administration, though, so it's been interesting.

## what does it actually take

The specific stack of Mastodon consists of the following parts:

* A Rails server to run web pages and the API
* A PostgreSQL server to store user and post information
* A Redis server to handle users' feeds, the streaming API, and caching
* (optional) An ElasticSearch server to handle search functionality
* (optional) Some object storage, like S3 or Minio, to store media off the local filesystem

The [Mastodon repo itself][masto-repo] has configurations for docker-compose and Vagrant, as well as
ready-made configuration for Heroku and similar services. If you want a push-button setup, one of
these configurations will suffice. However, i wanted to avoid using Docker or other kinds of
virtualization, and i wanted to run it on a VPS that i could control.

[masto-repo]: https://github.com/mastodon/mastodon

The [official instructions][masto-install] for setting up Mastodon outside of Docker assume that
you're running Ubuntu or Debian, for the purposes of noting how to install dependencies or manage
services. I ran my server on Ubuntu for a while, but ultimately i got fed up when i couldn't easily
upgrade from one LTS release to the next without starting fresh. I have more experience managing
Arch Linux systems, and the lure of "not having to deal with OS versions upgrades" was strong. After
a while i decided to migrate the server to Arch for ease of administration.

[masto-install]: https://docs.joinmastodon.org/admin/install/

## so how did you do it

I'm kinda assuming here that you already know how to set up Arch and have gotten a reasonable system
running, with a personal user that can run `sudo` and either SSH access or other kind of terminal
access. I'm not quite here to evangelize Arch and/or teach it; just to talk about setting up
Mastodon on a semi-unsupported platform.

For the most part, the [installation from source instructions][masto-install] can be followed pretty
closely. Where the process diverges on a different distro usually comes down to different system
package availability and user management. What follows is a series of "notes to self", from what i
can remember about setting up the Arch system in contrast with the official instructions.

-----

If you want to follow the docs' [recommendation for setting up a firewall][iptables] that blocks
everything but HTTP, HTTPS, and SSH, the equivalent files are `/etc/iptables/iptables.rules` and
`/etc/iptables/ip6tables.rules`, and the way to load the rules is to use the systemd units
`iptables.service` and `ip6tables.service`, which are one-shot services that load these saved rules
when they are started. See also, [ArchWiki's blurb about configuring iptables][arch-iptables].

[iptables]: https://docs.joinmastodon.org/admin/prerequisites/#install-a-firewall-and-only-allow-ssh-http-and-https-ports
[arch-iptables]: https://wiki.archlinux.org/title/Iptables#Configuration_and_usage

-----

When installing system packages:

- Arch has several different Node.js packages for various LTS versions. While the instructions have
  you `curl | bash` an installation script for node 20.x, Arch users can install the
  `nodejs-lts-iron` package instead, which tracks the same version.
- Arch keeps an extremely up-to-date version of PostgreSQL in its repos. This can [cause
  problems][postgres-upgrade] when new versions come out. I recommend installing both `postgresql`
  and `postgresql-old-upgrade` to have the old version on hand whenever a major release comes out.
  You should also add `postgresql` and `postgresql-libs` to `options.IgnorePkg` in
  `/etc/pacman.conf`. Watch very carefully when running system upgrades! If you get a warning about
  holding back these packages, you will probably run into problems if you continue to upgrade
  everything else. Something like libxml or OpenSSL updating will cause postgres to fail to launch,
  because the shared libraries changed on the system.

[postgres-upgrade]: https://wiki.archlinux.org/title/PostgreSQL#Upgrading_PostgreSQL

An equivalent `pacman` call to install the prerequisite packages looks like this:

```
sudo pacman -S base-devel imagemagick ffmpeg libpqxx libxml2 libxslt protobuf \
  openssl libyaml nginx certbot certbot-nginx redis postgresql \
  postgresql-old-upgrade libidn jemalloc yarn nodejs-lts-iron
```

-----

Remember that Ubuntu automatically starts associated systemd services after installing a package,
but Arch doesn't, so you'll need to `systemctl enable --now postgresql.service redis.service
nginx.service` before starting to configure the database, the web server, and Mastodon itself.
Remember to also follow the wiki instructions for setting up [Certbot][] and [PostgreSQL][]. You'll
want to enable/start `certbot-renew.timer` to set up automatic certificate renewal.

[Certbot]: https://wiki.archlinux.org/title/Certbot
[PostgreSQL]: https://wiki.archlinux.org/title/PostgreSQL

-----

Because of the way system libraries are upgraded, many native extensions to Ruby gems can fail to
link against them after an upgrade. Because of this, i tend to include an `rm -rf vendor/bundle
node_modules` before each `bundle install && yarn install` during an update.

-----

Because Arch's version of OpenSSL is relatively new compared to the Ubuntu LTS releases, there's an
issue with the hash used by webpack when precompiling assets. To counteract this, add the following
to your environment when running `assets:precompile` (i added it to the `mastodon` user's bashrc):

```
export NODE_OPTIONS=--openssl-legacy-provider
```

-----

Mastodon pins a Ruby version that it uses to run Rails and Sidekiq with. The official docs suggest
to use `rbenv` to set a Ruby version for Mastodon, and it can be used here as well. When Mastodon
changes its pinned Ruby version, you can quickly set it up with `rbenv install` (which will compile
Ruby from source) followed by `rbenv global (version)`. `rbenv` will likely complain about not
knowing about the version in question, but it will also show a `git pull` command you can use to
update its version list. (It's probably also worth running `rbenv versions` and `rbenv uninstall
(old version)` when this happens, too, so you can clean up old stuff.)

-----

Recent versions of Node.js include "[corepack][]", an alternate way to manage package managers in
the npm universe. This can create some headaches if you, like me, are unfamiliar with the world of
Node.js and npm. The one trick i can give you is to run `corepack enable` as root, because it will
otherwise try to mess with system directories.

[corepack]: https://yarnpkg.com/corepack

### the mastodon user

Arch's `useradd` command doesn't have a `--disabled-login` flag like the `adduser` command used in
the official docs. Instead, the wiki suggests setting the login shell to `/usr/bin/nologin` to
disallow login shells for service users. However, this means that the simple `su` command used in
the docs won't work - it will try to run `nologin`, which will then refuse the login.

When creating the `mastodon` user, this is roughly the command i used:

```
sudo useradd --system --user-group --create-home --shell /usr/bin/nologin mastodon
```

When switching to the `mastodon` user, you'll need to specify a shell as part of the command:

```
sudo su - -s /bin/bash mastodon
```

-----

Because i like to copy/paste commands from [the glitch-soc upgrade docs][glitch-docs] when
performing Mastodon upgrades, i wanted to find a way to get the `mastodon` user to reload the
Mastodon services without having to switch back to my primary user with `sudo` access. However,
thanks to [`polkit`][], it's possible to allow `mastodon` to reload the web services without needing
`sudo` access. Write the following file:

[glitch-docs]: https://glitch-soc.github.io/docs/
[`polkit`]: https://wiki.archlinux.org/title/Polkit

```javascript
// /etc/polkit-1/rules.d/10-mastodon.rules
polkit.addRule(function(action,subject) {
    var isSystemd = (action.id == "org.freedesktop.systemd1.manage-units");
    if (isSystemd) {
        var unit = action.lookup("unit");
        var user = subject.user;

        var mastoServices = ["mastodon-web.service", "mastodon-sidekiq.service", "mastodon-streaming.service"];
        var mastoUsers = ["mastodon"]; // add your personal user here if you want

        var isMastoService = (mastoServices.indexOf(unit) > -1);
        var isMastoUser = (mastoUsers.indexOf(user) > -1);
        if (isMastoService && isMastoUser) {
            return polkit.Result.YES;
        }
    }
});
```

### regular cleanup

The Mastodon docs suggest running a cleanup command every week to clean up the media caches.
However, they don't ship with scripts or timers that do this, so i forgot to set it up initially.
Add the following script, service, and timer to run this regularly:

```bash
# /home/mastodon/mastodon-cleanup.sh
# remember to `chmod +x ~/mastodon-cleanup.sh` so it can run properly
#!/usr/bin/env bash

set -e
set -o pipefail

export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"

cd $HOME/live
export RAILS_ENV=production
bin/tootctl media remove
bin/tootctl preview_cards remove
```

```
# /etc/systemd/system/mastodon-cleanup.service
[Unit]
Description=Weekly cleanup tasks for mastodon

[Service]
Type=oneshot
User=mastodon
ExecStart=/home/mastodon/mastodon-cleanup.sh
```

```
# /etc/systemd/system/mastodon-cleanup.timer
[Unit]
Description=Run mastodon cleanup tasks weekly

[Timer]
OnCalendar=weekly
RandomizedDelaySec=3600
Persistent=true

[Install]
WantedBy=timers.target
```

### full-text search

If you want to set up full-text search, the [official docs][masto-es] point you to Elasticsearch,
since that's what it's tested with. However, while Elasticsearch is available in the AUR and there's
[an ArchWiki page on setting it up][arch-es], i wanted to avoid using the AUR on a production
system. Instead, someone pointed me to [OpenSearch][], a compatible fork with a license that allows
it to be packaged in Arch's repos. After installing and configuring OpenSearch with the wiki's
instructions, i was able to follow the rest of the instructions in the Mastodon docs to configure
the search index and get full-text search up and running.

I would personally recommend following the wiki's advice of binding OpenSearch to localhost, unless
you're specifically setting up a multi-server installation (which is outside the scope of this post,
but there is [information in the Mastodon docs][masto-scaling] if you're curious).

[masto-es]: https://docs.joinmastodon.org/admin/elasticsearch/
[arch-es]: https://wiki.archlinux.org/title/Elasticsearch
[OpenSearch]: https://wiki.archlinux.org/title/OpenSearch
[masto-scaling]: https://docs.joinmastodon.org/admin/scaling/

## to be continued?

I may add more things here if i run into more weird edges in systems administration.

## changelog

Since i keep updating this post when i tweak my server, i'll try to keep a log of when/what i update
here:

- *2022-12-24*: Initial version.
- *2023-07-01*: Added blurb about the cleanup job.
- *2024-01-27*: Multiple updates:
  - Updated Node.js package to reflect current requirements.
  - Added blurb about corepack and Yarn.
  - Added blurb about configuring iptables.
  - Added section about configuring OpenSearch for full-text search.
- *2024-02-22*: Multiple updates:
  - Small tweak to the note about clearing installed Ruby/npm dependencies.
  - Added blurb about manually enabling and starting services.
