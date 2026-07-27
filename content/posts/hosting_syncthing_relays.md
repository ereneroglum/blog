+++
date = '2026-07-27T15:40:14+03:00'
title = 'Hosting Syncthing Relays'
tags = [ 'syncthing', 'relay', 'nat', 'internet', 'podman', 'container', 'docker', 'systemd' ]
+++


## What is Syncthing

[Syncthing](https://syncthing.net/) is a great application to sync files between
your devices. It always worked for me without setting up any infrastructure
myself and I have been using it for a while to sync my music collection, books
and [KeePassXC](https://keepassxc.org/) database. While it does not require
me to setup any infrastructure it needs _some_ infrastructure to be set up.
Syncthing at bare minimum needs to discover the IP addresses of peers. If two
peers are in the same broadcast domain, syncthing can discover peers using local
discovery methods. But if peers are in different broadcast domains, they are
discovered using global discovery methods. This depend on the community hosted
[discovery servers])(https://docs.syncthing.net/users/stdiscosrv.html). Upon
discovering their IP addresses, peers try to connect directly. If not this is
where you have to abandon all hope.

## A little rant about Internet Services

Because the world is a cruel place, most ISP's in my country are incompetent at
best and do not provide IPv6 to their customers. Even my university
([METU](https://www.metu.edu.tr/)) has problems with IPv6 addresses. These
ISP's (which in my opinion very much fail to provide internet service) make sure
to assign minimal amount of IP addresses to their customers by using one or more
layers of NAT. In other parts of the world many ISP's can not provide addresses
because of IPv4 exhaustion but fear not, this isn't the case for me. In my
country, ISP's is actually did not assign most of their IP space to anybody and
they refuse it because of the monetary incentives. Why should they care about
the internet and the end to end connectivity and all the other things when they
can charge more to use some arbitrary numbers ?

## How Syncthing works when there is nothing to connect

Syncthing tries to overcome these limitations by trying multiple NAT traversal
techniques. These are mostly successful in my case but still there are some
cases where connectivity can not be established. When NAT traversal fails
syncthing tries to relay the data from community hosted
[relay servers](https://docs.syncthing.net/users/strelaysrv.html).

## So what ?

I have benefited from these relay servers and I want to contribute back. I have
realized my 3 VPS servers are mostly sitting idle and I will host syncthing
relays on these servers. Setting these servers are actually surprisingly easy.
I am using Debian 13, but you can follow along with any distribution. I will use
podman with rootless containers in separate users and I want to automatically
start and update the syncthing relay containers. Podman integrates nicely with
Systemd.

## The Setup

First I have installed the dependencies with:

```
sudo apt install podman systemd-container
```

Then I created a user account:

```
sudo useradd -F -r -s /usr/sbin/nologin -m -d /var/lib/syncthing syncthing
```

Since I want the service to be active all the times I enabled lingering which
keeps a systemd user session intact. Normally user sessions are created upon
PAM authentication they are active only when someone is actively using the
account. But in this case lingering is what I want:

```
sudo loginctl enable-linger syncthing
```

Now I want to open a shell as the new syncthing user but I want this shell to
inherit all the necessary environment variables from systemd user session. The
```systemd-container``` package provides ```machinectl``` utility. Normally this
is used for managing systemd containers, but we can also use it open a shell
with all the necessary variables for systemd:

```
sudo machinectl shell syncthing@ /bin/bash
```

Now the only thing remaining is to write the configuration for syncthing
container. I am using a feature of podman called podman quadlets. This feature
lets systemd recognize some files to create containers as systemd services on
the fly. You can find more information
[here](https://docs.podman.io/en/latest/markdown/podman-quadlet.1.html). The
first file defines a volume to hold persistent data. I have placed this
in `~syncthing/.config/containers/systemd/syncthing.volume`

```
[Volume]
```

the next file is for configuring the container. Here is my configuration
in ```~syncthing/.config/containers/systemd/syncthing.container```:

```
[Unit]
Description=Syncthing Relay Service

[Container]
Image=docker.io/syncthing/relaysrv:edge
AutoUpdate=registry
PublishPort=22067:22067
PublishPort=22070:22070
Volume=syncthing.volume:/var/strelaysrv

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Now the only thing remains is to instruct systemd to scan the file system
to detect newly created services and start the containers:

```
systemctl --user daemon-reload
systemctl --user start syncthing
```

We don't need to instruct it to start the services on boot since the
podman generator handles it. We need to enable the timer for auto updates
though:

```
systemctl --user enable podman-auto-update.timer
```

If everything is done correctly, the relay IP should be on
[relay statistics](https://relays.syncthing.net/) page. I have checked and I
can see my relay is up. You may need to alter your firewall rules to allow TCP
traffic with destination ports `22067` and `22070` but this entirely depends
on your setup so I did not include them in this post.
