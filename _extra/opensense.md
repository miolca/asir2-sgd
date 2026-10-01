---
layout: default
title: "OpenSense"

---
# OpenSense

## VM Install
```bash


$incus profile create opnsense << EOF
config:
  limits.cpu: "2"
  limits.memory: 4GiB
  raw.qemu: |
    -cpu host
  raw.qemu.conf: |
   [device "dev-qemu_rng"]
  security.secureboot: "false"
devices: 
  eth0:
    name: lan
    nictype: bridged
    parent: ovs-br0    
    type: nic
  eth1:
    name: wan
    nictype: bridged
    parent: ovs-br1    
    type: nic
  eth2:
    name: mgnt
    network: grao-net    
    type: nic
  root:
    path: /
    pool: incus-os-zfs
    type: disk
EOF
$incus create opnsense --vm --empty -d root,size=40GiB --profile opnsense

$incus config device add opnsense iso disk     source=~/Downloads/OPNsense-26.1.6-dvd-amd64.iso   boot.priority=10

$incus start opnsense --console=vga #default password opnsense
incus config device remove opnsense iso


```

Configurar opensense desde consola, habilitando servidor dhcp en lan

```bash
incus launch images:ubuntu/noble lan1 --network ovs-br0
```


## Transparente firewall bridge

* Crear Management interface y configurar cortafuegos (HTTP/HTTPS/ICMP)
* Desactivar outbound NAT
* Cambiar system tuneables
* Crear interface bridge (lan,wan)
* Desactivar bogon/private network en WAN
* Desactivar IP en LAN/WAN
* Crear firewall rule bridge
* Activar HTTPS (Install CA/Server certificate)



## Links

[Opensense bridge mode](https://docs.opnsense.org/manual/how-tos/transparent_bridge.html)
[Opensense bridge mode](https://www.youtube.com/watch?v=ZCDXNxDhrIQ&t=1362s)