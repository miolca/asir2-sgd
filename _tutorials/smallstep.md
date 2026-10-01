---
layout: default
title: smallstep
---
# SmallStep. CA

Smallsteps permite la creación y gestión de una CA en la que los clientes pueden solicitar certificados y renovarlos de forma automática mediante el protocolo ACME.

## Servidor

```bash
incus launch images:ubuntu/noble smallstep
incus exec smallstep -- apt-get install -y bash-completion wget aptitude curl
```

Installar small step-ca y step-cli
```bash
apt-get update && apt-get install -y --no-install-recommends curl gpg ca-certificates
curl -fsSL https://packages.smallstep.com/keys/apt/repo-signing-key.gpg -o /etc/apt/keyrings/smallstep.asc
cat << EOF > /etc/apt/sources.list.d/smallstep.sources
Types: deb
URIs: https://packages.smallstep.com/stable/debian
Suites: debs
Components: main
Signed-By: /etc/apt/keyrings/smallstep.asc
EOF
apt-get update && apt-get -y install step-cli step-ca
```
Creamos una CA con 10 años de duración
```bash
mkdir -p /etc/step
export STEPPATH=/etc/step
useradd step
passwd -l step 
step ca init
echo "p4ssword"> /etc/step/password
chmod 400 /etc/step/password
step ca init --name="iesgraoCA" \
		--dns="smallstep,smallstep.ies.grao" \
		--address=":443" --provisioner="admin" \
		--deployment-type standalone \
		--password-file /etc/step/password.txt \
            --provisioner-password-file /etc/step/password.txt
chown -R step:step /etc/step/
```

Comprobamos que arranca el servidor CA

```bash
sudo -u step step-ca /etc/step/config/ca.json --password-file /etc/step/password.txt
```

> [!NOTE]
Si deseamos crear una CA con más de 10 años de duración o utilizar nuestro ya existente certificado ca_root.crt debemos ejecutar los siguientes pasos:
```bash
$ step certificate create "My Root CA" root_ca.crt root_ca_key --profile root-ca --not-after 175200h
$ step ca init --root root_ca.crt --key root_ca_key
```

Añadimos step como servicio en systemd

```bash
# The service script
cat > /etc/systemd/system/step-ca.service  << EOF
[Unit]
Description=Smallstep Certificate Authority

[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
User=step
Group=step
Environment="STEPPATH=/etc/step"
ExecStart=/usr/bin/step-ca /etc/step/config/ca.json --password-file=/etc/step/password.txt

[Install]
WantedBy=multi-user.target
EOF
```

```bash
systemctl daemon-reload
systemctl enable --now step-ca
systemctl start --now step-ca
systemctl status --now step-ca
sudo journalctl --follow --unit=step-ca
```


Añadimo ACME provisioner al servidor

```bash
export STEPPATH=/etc/step
step ca provisioner add acme --type ACME --admin-name acme
systemctl restart step-ca
```

Cambiar la duración por defecto de los certificados emitidos
```bash
step ca provisioner update admin --ca-config=/etc/step/config/ca.json \
	--x509-default-dur=87600h
```

## Cliente

Debemos instalar step-cli y crear una relación de confianza con step-ca

```bash
step ca bootstrap --ca-url smallstep --fingerprint <fingerpring>
```
Solicitar un certificado

```bash
step ca certificate svc.example.com srv.crt srv.key
``` 
Solicitar un certificado con duración implícita.
```bash
step ca certificate srv.example.com srv.crt srv.key --not-after=5m
``` 

Solicitar un certificado con multiple SAN y rutas de almacenamiento
```bash
step ca  certificate apache.local --san=10.149.165.115 \
	--san=apache --san=localhost \
	--san=127.0.0.1 \
	--not-after=___m \
	/etc/apache2/ssl/srv.crt \
	/etc/apache2/ssl/private/srv.key 

``` 

Solicitar un certificado con tocken previamente generado para el dominio específico
```bash
TOKEN=$(step ca token foo.local)
step ca certificate foo.local foo.crt foo.key --token $TOKEN
``` 

Renovar un certificado
```bash
step ca certificate srv.example.com srv.crt srv.key --not-after=5m
``` 
Ver información del certificado
 ```bash
step certificate inspect srv.crt 
step certificate inspect srv.crt --bundle
``` 

Renovar un certificado automáticamene 
[link](https://smallstep.com/practical-zero-trust/nginx-tls)
```bash
TODO
```


### ACME Provider

Experimentar con el protocolo ACME y probar la renovación automática, challenge, etc.
[link](https://smallstep.com/practical-zero-trust/nginx-tls)
```bash
TODO
```


## Links
* [Self-Hosted TRUST with your own Certificate Authority!](https://www.apalrd.net/posts/2023/network_acme/#install-smallstep)
* [How to Create a Certificate Authority Server via SmallStepCA](https://medium.com/@ed-roof-maker/how-to-create-a-certificate-authority-server-via-smallstepca-786cbb201016)




## Anexo I. Glosario de términos
* _Provisioners_ : Provisioners son los métodos para demostar a la CA la identidad de quien solicita un certificado. La CA neecesita una prueba de que la entidad solicitante es quien dice ser. 



## Anexo II. smallstep commands
```bash
$ step certificate inspect <crt-file>
$ step ca certificate apache.example.com apache.crt apache.key
$ step ca bootstrap --ca-url [CA URL] --fingerprint [CA fingerprint]
$ step certificate fingerprint $(step path)/certs/root_ca.crt
$ step certificate install
$ step certificate create --csr foo.example.com foo.csr foo.key
$ step ca sign
$ step certificate install --firefox root-ca.pem
$ step ca root root.crt
```

