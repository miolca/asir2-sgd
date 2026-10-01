---
layout: default
title: smallstep
---
# Silverbullet with Caddy

Silverbullet es una aplicación self-hosted que nos permite crear notas en formato markdown y que además incorpora numerosas mejoras para hacer más eficiente nuestro trabajo de documentación

## Servidor

```bash
incus launch images:ubuntu/noble silverbullet
incus exec smallstep -- apt-get install -y bash-completion wget aptitude curl unzip
```

Instalar y configurar Caddy

```bash
apt-get install caddy
systemctl enable caddy
mkdir /etc/caddy/ssl
```
Generar los certificados TLS necesarios con _step_

```bash
step ca  certificate silverbullet   \
     --san=silverbullet --san=localhost \
     --san=caddy.ies.grao --san=caddy.incus \
     /etc/caddy/ssl/caddy.crt  \
     /etc/caddy/ssl/caddy.key
```

```bash
#/etc/caddy/Caddyfile
:443 {
        # Set this path to your site's directory.
        root * /usr/share/caddy

        # Enable the static file server.
        file_server

        # Another common task is to set up a reverse proxy:
        reverse_proxy localhost:3000
        tls /etc/caddy/ssl/caddy.crt /etc/caddy/ssl/caddy.key
}
```

Descagar silverbullet e instalar como servicio
```bash
mkdir /opt/silverbullet && cd /opt/silverbullet
wget  https://github.com/silverbulletmd/silverbullet/releases/latest/download/silverbullet-server-linux-x86_64.zip
unzip silverbullet-server-linux-x86_64.zip
./silverbullet upgrade

```

Crear usuario no login silverbullet
```bash
groupadd silverbullet
useradd -r -g silverbullet -d /opt/silverbullet -s /sbin/nologin silverbullet
chown -R silverbullet:silverbullet /opt/silverbullet
```

Instalar como servicio systemctl
```bash
#/etc/systemd/system/silverbullet
[Unit]
Description=Silverbullety Server
#After=network.target postgresql.service
#Wants=postgresql.service

[Service]
Type=simple
User=silverbullet
Group=silverbullet
ExecStart=/opt/silverbullet/silverbullet -p3000 -L0.0.0.0 /opt/silverbullet/space

[Install]
WantedBy=default.target

```



## Links
