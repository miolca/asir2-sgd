# DNS .grao
**Used ip’s**

_Grao-net_ network 10.0.10.0/24

| Service   | ip | description
|---------  |-------| -----|
| ns1           | 10.0.10.11  | grao auth DNS 
| ns2           | 10.0.10.12  | ies.grao auth DNS |
| ns3           | 10.0.10.13  | asir.grao auth DNS |
| caddy         | 10.0.10.15  | Reverse proxy server |
| ldap          | 10.0.10.20  | |
| smallstep     | 10.0.10.25  | pki (certs) |
| authelia      | 10.0.10.30  | OIDC server | 
| zammad        | 10.0.10.35  | tic server |
| ansible       | 10.0.10.40  | automation |
| silverbullet  | 10.0.10.45  | notes |
| nextcloud     | 10.0.10.50  | cloud storage |
| incus-1       | 10.0.10.60  | cluster 1 |
| incus-2       | 10.0.10.61  | cluster 2 |
| incus-3       | 10.0.10.62  | cluster 3 |
| GW            | 10.0.10.100 | Gateway |



**Crear escenario**

```bash
incus network create grao-net \
    ipv4.address=10.0.10.100/24 \
    ipv6.address=none ipv4.nat=true
    dns.nameservers=10.0.10.11
    dns.domain=ies.grao
    

incus launch images:ubuntu/noble ns1 --network grao-net -d eth0,ipv4.address=10.0.10.11
incus launch images:ubuntu/noble ns2 --network grao-net -d eth0,ipv4.address=10.0.10.12
incus launch images:ubuntu/noble gcli --network grao-net -d eth0,ipv4.address=10.0.10.13
#incus config device set ns1 eth0 ipv4.address=192.168.10.223
#incus config device add <instance> eth0 nic network=grao-net

incus exec ns1 -- bash -c 'apt-get update && apt-get -y install  aptitude wget bash-completion gpg nano xsel vim curl dnsutils bind9'
incus exec ns2 -- bash -c 'apt-get update && apt-get -y install  aptitude wget bash-completion gpg nano xsel vim curl dnsutils bind9' 
incus exec gcli -- bash -c 'apt-get update && apt-get -y install  aptitude wget bash-completion gpg nano xsel vim curl dnsutils dnsmasq' 
```

**Configure ns1 como TLD grao**

```bash
#/etc/bind/named.conf.local
zone "grao" {
	type master;
	file "/etc/bind/db.grao";
};

zone "ies.grao" {
	type forward;
    forward only;
	forwarders { 10.0.10.12; };	
};

zone "asir.grao" {
	type forward;
    forward only;
	forwarders { 10.0.10.13; };	
};
```

```bash
#/etc/bind/named.conf.options
forwarders {
		192.168.10.100;
	};

```

```bash
#/etc/bind/db.grao
;
; BIND data file for ies.grao
;
$TTL    604800
$ORIGIN grao.
@       IN      SOA     ns1.grao. admin.grao. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

        IN      NS      ns1.grao.
ns1     IN      A       10.0.10.11

;subdoamin definitions in the same zone file
$ORIGIN ies.grao.
;name servers for the subdomain
ies.grao.	IN 	NS	ns2.ies.grao.
ns2.ies.grao.	IN	A	10.0.10.12

$ORIGIN asir.grao.
;name servers for the subdomain
asir.grao.	IN 	NS	ns2.asir.grao.
ns3.asir.grao.	IN	A	10.0.10.13
```

**Configure ns2 como subdomain ies.grao**

```bash
#/etc/bind/named.conf.local
zone "ies.grao" {
	type master;
	file "/etc/bind/db.ies.grao";
};
```

```bash
#/etc/bind/named.conf.options
forwarders {
		10.0.10.100;
	};
```


```bash
#/etc/bind/db.ies.grao
**Configure ns3 como subdomain asir.grao**

```bash
#/etc/bind/named.conf.local
zone "asir.grao" {
	type master;
	file "/etc/bind/db.asir.grao";
};
```

```bash
#/etc/bind/named.conf.options
forwarders {
		192.168.10.100;
	};
```


```bash
#/etc/bind/db.ies.grao
;
; BIND data file for ies.grao
;
$TTL    604800
$ORIGIN ies.grao.
@       IN      SOA     ns2.ies.grao. admin.ies.grao. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@               IN      NS      ns2.ies.grao.
ns1             IN      A       10.0.10.11
ns2             IN      A       10.0.10.12
ns3             IN      A       10.0.10.13
caddy			IN      A       10.0.10.15
ldap            IN      A       10.0.10.20
smallstep       IN      A       10.0.10.25
authelia        IN      A       10.0.10.30
zammad          IN      A       10.0.10.35
ansible         IN      A       10.0.10.40
silverbullet    IN      A       10.0.10.45
nextcloud       IN      A       10.0.10.50
incus-1         IN      A       10.0.10.60
incus-2         IN      A       10.0.10.61
incus-3         IN      A       10.0.10.62
auth            CNAME   authelia
dns             CNAME   ns1
notes           CNAME   silverbullet
pki             CNAME   smallstep
tic             CNAME   silverbullet
```


**Configure ns3 como subdomain asir.grao**

```bash
#/etc/bind/named.conf.local
zone "asir.grao" {
	type master;
	file "/etc/bind/db.asir.grao";
};
```

```bash
#/etc/bind/named.conf.options
forwarders {
		10.0.10.100;
	};
```


```bash
#/etc/bind/db.ies.grao
;
; BIND data file for asir.grao
;
$TTL    604800
$ORIGIN asir.grao.
@       IN      SOA     ns3.asir.grao. admin.ies.grao. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@        IN      NS      ns3.asir.grao.
ns3      IN      A       10.0.10.13
www      IN      A       10.144.144.2

```


**Configure router with dnsmasq to redirect /grao/ queries to ns1**

```bash
domain-needed
bogus-priv
server=/grao/192.168.10.200
```


**Configure client to route domain /grao/ to ns1**

```bash
#/etc/systemd/resolved.conf
[Resolve]
DNS=192.168.10.100
```
```bash
resolvectl dns eth0 192.168.10.200
resolvectl domain eth0 '~grao'
```
