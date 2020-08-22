All credits go to joonty (https://github.com/joonty/systemd_mon) who is the author behind this tool.

To offer a complete experience, I have also incorporated the improvements made by:
- Asquera (https://github.com/Asquera/systemd_mon) => stdout, gelf and http notifier
- florczakraf (https://github.com/florczakraf/systemd_mon) => fields 'cc' and 'bcc' for emails
- kennep (https://github.com/kennep/systemd_mon) => addition of the parameters 'hostname' and first version Dockerfile
- Maistho (https://github.com/Maistho/systemd_mon) => corrections of various problems such as not starting during a missing systemd unit or resending the initial state at startup
- milk531 (https://github.com/milk531/systemd_mon) => dingbot notify
- p91 (https://github.com/p91/systemd_mon) => Desktop notification via DBus

# SystemdMon

Monitor systemd units and trigger alerts for failed states. The command line tool runs as a daemon, using dbus to get notifications of changes to systemd services. If a service enters a failed state, or returns from a failed state to an active state, notifications will be triggered.

Built-in notifications include email, slack, and hipchat, but more can be added via the ruby API.

It works by subscribing to DBus notifications from Systemd. This means that there is no polling, and no busy-loops. SystemdMon will sit in the background, happily waiting and using minimal processes.

## Requirements

* A linux server
* Ruby > 1.9.3 (2.7 for Dockerfile)
* Systemd (v204 was used in development)
* `ruby-dbus`
* `mail` gem (if email notifier is used)
* `slack-notifier` gem > 1.0 (if slack notifier is used)
* `hipchat` (if hipchat notifier is used)
* `dingbot` gem (if ding notifier is used)
* `gelf` gen (if gelf is used)

## Installation

Install the gem using:

    gem install systemd_mon

## Usage

To run the command line tool, you will first need to create a YAML configuration file to specify which systemd units you want to monitor, and which notifications you want to trigger. A full example looks like this:

```yaml
---
verbose: true # Default is off
hostname: hostname
notifiers:
  desktop:
   start_stop_message: false
   timeout: 2000
  ding:
    endpoint: https://oapi.dingtalk.com/robot/send
    access_token: accesstokenhere
  email:
    to: "team@mydomain.com"
    from: "systemdmon@mydomain.com"
    # cc and bcc are optional
    cc: "cc@mydomain.com"
    bcc: "bcc@mydomain.com"
    # These are options passed to the 'mail' gem
    smtp:
        address: smtp.gmail.com
        port: 587
        domain: mydomain.com
        user_name: "user@mydomain.com"
        password: "supersecr3t"
        authentication: "plain"
        enable_starttls_auto: true
  gelf:
    host: ip_of_gelf_host
    port: port_of_gelf_host
    level: INFO
    network: LAN
  hipchat:
    token: bigsecrettokenhere
    room: myroom
    username: doge
  http:
    bind_address: 0.0.0.0
    #with docker and docker-compose, remember to use the correct port if you modified it here
    bind_port: 9000
  slack:
    webhook_url: https://hooks.slack.com/services/super/secret/tokenthings
    channel: mychannel
    username: doge
    icon_emoji: ":computer"
    icon_url: "http://example.com/icon"
  stdout: {}
units:
- unicorn.service
- nginx.service
- sidekiq.service
```

Save that somewhere appropriate (e.g. `/etc/systemd_mon.yml`), then start the command line tool with:

    $ systemd_mon /etc/systemd_mon.yml

You'll probably want to run it via systemd, which you can do with this example service file (change file paths as appropriate):

```
[Unit]
Description=SystemdMon
After=network.target

[Service]
Type=simple
User=deploy
StandardInput=null
StandardOutput=syslog
StandardError=syslog
ExecStart=/usr/local/bin/systemd_mon /etc/systemd_mon.yml

[Install]
WantedBy=multi-user.target
```

## Behaviour

Systemd provides information about state changes in very fine detail. For example, if you start a service, it may go through the following states: activating (start-pre), activiating (start) and finally active (running). This will likely happen in less than a second, and you probably don't want 3 notifications. Therefore, SystemdMon queues up states until it comes across one that you think you should know about. In this case, it will notify you when the state reaches active (running), but the notification can show the history of how the state changed so you get the full picture.

SystemdMon does simple analysis on the history of state changes, so it can summarise with statuses like "recovered", "automatically restarted", "still failed", etc. It will also report with the host name of the server.

You'll also want to know if SystemdMon itself falls over, and when it starts back up again. It will attempt to send a final notification before it exits, and one to say it's starting. However, be aware that it might not send a notification in some conditions (e.g. in the case of a SIGKILL), or a network failure. The age-old question: who will watch the watcher?

## Get Image Container

You can either:
- Create your own image via the Dockerfile provides (with Docker or Buildah)

```
cd ./systemd_mon_directory/
docker build -t systemd_mon-image-name.
```

where "systemd_mon-image-name" is the name of the image you want to create

- Directly use the image created on my docker hub (https://hub.docker.com/r/gawindx/systemd_mon)

```
docker pull gawindx/systemd_mon:latest
```

Since systemd_mon relies on dbus, you need to mount the host dbus directory into your container. Besides that, the configuration filename is currently hardcoded to systemd_mon.yml. You have to mount the directory where the systemd_mon.yml file is located on your host system into your container as well. Below is a working example:

```
docker run --name "systemd_mon" \
            -v /var/run/dbus:/var/run/dbus \
            -v /path/to/systemd_mon/config/:/systemd_mon/ \
            -d \
            gawindx/systemd_mon
```

## Podman integration
Similar to using the systemdmon container with docker, you will need the image for use with podman (see "Get Image" above).

You can then launch the container as follows:
```
podman run --name "systemdmon"\
           --rm \
           --replace \
           --log-driver journald \
           --network host \
           --volume /var/run/dbus:/var/run/dbus:ro \
           --volume /var/Pod_Data/SystemdMon/config:/systemd_mon \
           -d \
           gawindx/systemd_mon:latest
```

## Docker/Systemd integration
If you want to run this image with systemd (very handy on CoreOS for example) you can use it as follows:

```
[Unit]
Description=systemd_mon
After=docker.service
Requires=docker.service

[Service]
Restart=always
RestartSec=60
ExecStartPre=-/usr/bin/docker kill systemd_mon
ExecStartPre=-/usr/bin/docker rm systemd_mon
ExecStart=/usr/bin/docker run --name "systemd_mon" \
                              -v /var/run/dbus:/var/run/dbus \
                              -v /path/to/systemd_mon/config/:/systemd_mon/ \
                              gawindx/systemd_mon:latest

[Install]
WantedBy=multi-user.target
```
## Podman/Systemd integration
Just like with Docker, podman can launch the container via Systemd.
However, the syntax is a bit more complex and requires a few additional parameters to comply with RedHat recommendations:
```
# %n == The full unit name, e.g. "nginx.service".
# %p == The unit's prefix, e.g. "nginx".

[Unit]
Description=SystemdMon in Container
After=network.target

[Service]
Type=forking
Restart=on-failure
KillMode=none

# Report variables.
WorkingDirectory=/Directory/where/the/systemd_mon/data/is/located
#Start
ExecStartPre=/usr/bin/rm -f /%t/%n-pid /%t/%n-cid
ExecStart=/usr/bin/podman run --conmon-pidfile /%t/%n-pid \
                                --cidfile /%t/%n-cid \
                                --name "%p"\
                                --rm \
                                --privileged true \
                                --replace \
                                --log-driver journald \
                                --health-cmd "ping -w 8 -4 -c 1 8.8.8.8 || exit 1" \
                                --health-interval 30s \
                                --health-retries 3 \
                                --health-start-period 120s \
                                --health-timeout 30s \
                                --network host \                                
                                --volume /var/run/dbus:/var/run/dbus:ro \
                                --volume /Directory/where/the/systemd_mon/data/is/located:/systemd_mon:Z \
                                -d \
                                gawindx/systemd_mon:latest

ExecStop=/usr/bin/sh -c "/usr/bin/podman rm -f `cat /%t/%n-cid`"

[Install]
WantedBy=multi-user.target
```
Podman does not understand the healthcheck syntax in the same way as Docker, it is better to tell it again.

## Docker-compose Integration

You can also integrate directly via docker compose (which in my preference compared to direct integration via docker).

Create a docker-compose.yml file:

```
version: '3.4'

services:
  docker-systemdmon:
    image: gawindx/systemd_mon
    restart: always
    container_name: systemd_mon
#optional
#    network_mode: "host"
    logging:
      driver: journald
#uncomment if you use http notifier
#use the same port here as in the configuration file
#    ports:
#      - 9000:9000/tcp
#If you use your own dns servers, it may be wise to indicate an external dns here 
#so that the container can reach the internet in difficulties (slack, dingbot, etc.)
#    dns:
#      - 8.8.8.8
    volumes:
      - /var/run/dbus:/var/run/dbus:ro
      - /path/to/systemd_mon/config/:/systemd_mon/
```

You can then run it via systemd:

```
[Unit]
Description=SystemdMon in Docker
Requires=docker.service

[Service]
WorkingDirectory=/path/to/docker-compose/directory
# docker stop/rm just in case the container has not been removed (e.g. the system crashed).
ExecStartPre=-/usr/bin/docker-compose down
ExecStartPre=-/usr/bin/docker-compose rm -f
#Always work with the most recent version of the image (optional)
ExecStartPre=-/usr/bin/docker-compose pull --ignore-pull-failures

ExecStart=/usr/bin/docker-compose up
ExecStop=/usr/bin/docker-compose down

# Restart 2 seconds after docker run exited with an error status.
Restart=always
RestartSec=2s
#In the case of a slow connection for downloading the image
#TimeoutStartSec=900

[Install]
WantedBy=multi-user.target

```

## Healthcheck

The Dockerfile provided incorporates a Healthcheck routine based on an external ping.

It is therefore easy to monitor his condition and his ability to communicate to the outside (willfarrell/autoheal by example)

## SeLinux
If Selinux is enabled on your system, you risk not being able to launch the container (similar problem when using systemd_mon directly).

To be able to run it without problem:
- Edit a new file named 'my-systemdmon.te' and insert the following text:
```
module my-systemdmon 1.0;

require {
        type init_t;
        type systemd_unit_file_t;
        type system_dbusd_t;
        type container_t;
        class dbus send_msg;
        class system status;
        class service status;
        class unix_stream_socket connectto;
}

#============= container_t ==============
allow container_t init_t:dbus send_msg;
allow container_t init_t:system status;
allow container_t system_dbusd_t:dbus send_msg;
allow container_t systemd_unit_file_t:service status;
allow container_t system_dbusd_t:unix_stream_socket connectto;
```
Pay attention to the context of your container which should normally be 'container_t', otherwise it will not work

- Then execute the commands in the following order:
```
checkmodule -M -m -o my-systemdmon.mod my-systemdmon.te
semodule_package -o my-systemdmon.pp -m my-systemdmon.mod
semodule_package -i my-systemdmon.pp
```



