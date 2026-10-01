---
layout: default
title: keycloak
---
# Keycloak. Identity manager
 
  
## Contexto
Hacemos uso de un 'identity manager' para gestionar la autenticación de usuarios de los nuestros servicios de forma centralizada. Los usuarios pueden provenir de otros _identity managers_ tales como LDAP o AD. Los mecanismos de autenticación son varios OpenID, SAML o Kerberos.
![arquitectura keycloak](https://miro.medium.com/v2/resize:fit:720/format:webp/0*eOnL_yZsbPWJkpHf.png)
![SSO](https://media2.dev.to/dynamic/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2F9wyjr3ij5uc6hap1bpcp.png)


## Objetivos
* Instalación y configuración de Keycloack
* Integración con LDAP
* SSO mediante Kerberos


## Instalación

**Requisitos**

```bash
sudo aptitude install default-jre posgresql
```

* Descarga _keycloak_, descomprimelo y copialo en _/opt_

Para poder hacer funcionar keycloak en modo producción es necesario un certificado digital.
Utilizaremos nuestra _pki_ para crear uno firmado por nuestra CA_root.
Los ficheros _keycloak.crt_ y _keycloak.key_ los copiaremos en la carpeta _/opt/keycloak/conf/ssl_

* Crea el usuario _keycloak_ para poder ejecutar _keycloak_ en systemd.
```bash
groupadd keycloak
useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
```
* Asignamos permisos a los ficheros


```bash
find /opt/keycloak -type d -exec chmod 755 {} \;
find /opt/keycloak -type f -exec chmod 644 {} \;
chown -R keycloak:keycloak /opt/keycloak
```

El fichero _/conf/keycloak.conf_ contiene la siguiente configuración
```bash
db=postgres
db-username=postgres
db-password=my-db-password
db-url=jdbc:postgresql://localhost/keycloak
https-certificate-file=/opt/keycloak/conf/ssl/keycloak.crt
https-certificate-key-file=/opt/keycloak/conf/ssl/keycloak.key
hostname=my-ip o dns
```

>[!NOTE]
Debemos solicitar los certificados a nuestra CA, si está instalada y configurada podemos hacerlo mediante _smallsteps cli_
```bash
step ca certificate keycloak.local --san=10.149.165.243 --san=keycloak \
	--san=localhost --san=127.0.0.1 \
	/opt/keycloak/conf/ssl/keycloak.crt \
	/opt/keycloak/conf/ssl/keycloak.key
```

Es necesario manejar los usuarios mediante una base de datos, usaremos postgresql. Una vez instalada hay que crear el usuario y la base de datos.
```bash
sudo -i -u postgres psql
CREATE DATABASE keycloak;
CREATE USER keycloak WITH ENCRYPTED PASSWORD 'my-db-password';
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;
\q
```

>[!WARNING]
> A falta de investigar, el código de arriba no asigna los permisos necesarios al usuario _keyclock_ para el correcto funcionamiento, por lo que de forma temporal se utilizará el usuario administrador _postgres_
```bash
sudo -i -u postgres psql
ALTER USER postgres WITH PASSWORD 'my-db-password'
\q
```

Integramos el arranque de keycloak en el sistema creando una unidad de _systemd_


```bash
#/etc/systemd/system/keycloack.service
[Unit]
Description=Keycloak Server
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=idle
User=keycloak
Group=keycloak
ExecStart=/opt/keycloak/bin/kc.sh start --optimized
SuccessExitStatus=143
TimeoutStopSec=10
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

Activamos el servicio

```bash
systemctl enable keycloak.service
systemctl start keycloak.service
```

Creamos el usuario de administrador de forma temporal
```bash
/opt/keycloak/bin/kc.sh bootstrap-admin user --username tmpadm
```

Keycloak está disponible en la URL https://keycloak-ip:8443/admin


* ParaEs necesario un certificado para poder hacer funcionar keycloak en modo producción


>[!WARNING]
> En cada una de las actividades, realizadas, deberás presentar un documento con una breve descripción de los pasos realizados en la tarea y las instrucciones. Utiliza un formato adecuado para el `codigo`

>[!TIP]
> Puedes aprovechar y aprender a utilizar *markdown* un lenguaje de marcas muy sencillo con el que podrás hacer documentos como este mismo.


## Links
[[How-To] Install Keycloak on Ubuntu 24 LTS VM](https://wiki.stretchpowered.com/books/keycloak/page/how-to-install-keycloak-on-ubuntu-24-lts-vm)
[How to Install and Configure Keycloak](https://oneuptime.com/blog/post/2026-02-02-keycloak-installation-configuration/view)