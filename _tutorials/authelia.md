---
layout: default
title: authelia
tags: tutorial
---
# Authelia. Identity manager
 
  
## Contexto
Authelia es un servidor de authenticación and autorizarción, también conocido como identity serverrver*Hacemos uso de un 'identity manager' para gestionar la autenticación de usuarios de los nuestros servicios de forma centralizada. Los usuarios pueden provenir de un _identity provideer_ tales como LDAP o AD. Los mecanismos de autenticación son varios OpenID, SAML o Kerberos.

![[resources/img/authelia.png]]

## Objetivos
* Instalación y configuración de authelia
* Integración con LDAP
* SSO mediante Kerberos


## Instalación

```bash
$incus launch images:ubuntu/noble authelia
$incus exec authelia -- apt-get install -y  bash-completion wget aptitude curl unzip gnupg
$incus exec authelia curl -fsSL https://www.authelia.com/keys/authelia-security.gpg -o /usr/share/keyrings/authelia-security.gpg
$incus exec authelia -- bash -c 'echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/authelia-security.gpg] https://apt.authelia.com stable main" | \
  sudo tee /etc/apt/sources.list.d/authelia.list > /dev/null'
$incus exec authelia -- apt update && apt install authelia
```

## Configuración

> **note** Note
> El acceso a Authelia se realizam mediante **https**, por lo que es indispensable el uso de un certificado.
> Utilizaremos nuestra _pki_ para crear uno firmado por nuestra CA_root. 

```bash
$mkdir /etc/authelia/ssl
$step ca  certificate auth.local --san=10.149.165.115   --san=authelia --san=localhost --san=192.168.10.222       --san=127.0.0.1 --san=auth.ies.grao --san=auth.incus /etc/authelia/ssl/authelia.crt /etc/authelia/ssl/authelia.key
```

```bash
#/etc/authelia/configuration.yaml

---
server:
  tls:
    key: '/etc/authelia/ssl/authelia.key'
    certificate: '/etc/authelia/ssl/authelia.crt'
identity_validation:
  reset_password:
    jwt_secret: 'iesgrao_secret'
authentication_backend:
  file:
    path: '/etc/authelia/users_database.yml'
access_control:
  rules:
    - domain: '*.incus'
      policy: 'one_factor'
session:
  secret: 'iesgrao_session_secret'
  cookies:
     -
      name: 'authelia_session'
      domain: 'authelia.incus'
      authelia_url: 'https://authelia.incus'
storage:
  encryption_key: 'you_must_generate_a_random_string_of_more_than_twenty_chars_and_configure_this'
  local:
    path: '/etc/authelia/db.sqlite3'
notifier:
  filesystem:
    filename: '/etc/authelia/notification.txt'
...
```

```bash
#/etc/authelia/users_database.yml
users:
  admin:
    disabled: false
    displayname: 'admin'
    password: '$argon2id$v=19$m=65536,t=3,p=4$TMNmh4rUxxNaLu8CUtLWhw$UDriHhoSBBO2Y/Lsd/5yg+LhB08D5rWLWKNkxZJvJVk'
    #This corresponds to CorrectHorseBatteryStaple
    email: 'admin@ies.grao'
    groups:
      - 'admins'
  user1:
    disabled: false
    displayname: 'karen'
    password: '$argon2id$v=19$m=65536,t=3,p=4$TMNmh4rUxxNaLu8CUtLWhw$UDriHhoSBBO2Y/Lsd/5yg+LhB08D5rWLWKNkxZJvJVk'
    #This corresponds to karen
```

```bash
chown -R authelia:authelia /etc/authelia
```

> **note** Note
> Es recomentable hacer uso de la utilidad _yamllint_ para comprobar la sintaxis de los ficheros _yaml_ de configuración.
> Podemos eliminar los _warnings_ relacioados con la longitud de la linea y la intendación de los comentarios añadiendo las siguientes lineas al comienzo del fichero _yaml_ a comprobar
> ```#yamllint disable rule:comments-indentation```
> ```#yamllint disable rule:line-length```

**Systemd Unit**

Si configuramos _ldap_ como la fuente de usuarios, authelia depende de que el servidor ldap esté en funcionamiento para arrancar correctamente. Modificaremos la unidad de de arranque del _systemd_ para que en el caso de que _ldap_ no este funcionando cuando _authelia_ arranque vuelva a intentarlo durante 5 veces a intervalos de 5 segundos

```bash
#/usr/lib/systemd/system/authelia.service
[Unit]
Description=Authelia authentication and authorization server
Documentation=https://www.authelia.com
After=multi-user.target
StartLimitInterval=100
StartLimitBurst=5

[Service]
User=authelia
Group=authelia
UMask=027
Environment=AUTHELIA_SERVER_DISABLE_HEALTHCHECK=true
ExecStart=/usr/bin/authelia --config /etc/authelia/configuration.yml --config.experimental.filters template
SyslogIdentifier=authelia
CapabilityBoundingSet=
NoNewPrivileges=yes
RestrictNamespaces=yes
ProtectHome=true
PrivateDevices=yes
PrivateUsers=yes
ProtectControlGroups=yes
ProtectKernelModules=yes
ProtectKernelTunables=yes
SystemCallArchitectures=native
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **note** Note
> Es necesario crear el fichero _/etc/tmpfiles.d/authelia.tmpfiles.conf_ que establece los permisos de los ficheros en las unidades [_systemd_](https://www.authelia.com/reference/guides/systemd/)

```bash
Z     /etc/authelia/*     0640 authelia   authelia   -           -
z     /etc/authelia/ssl/  0740 authelia   authelia   -           -
e     /etc/authelia       0755 authelia   authelia   -           -
```

## Integración [[tutorials/lldap]]

Hasta ahora, los usuarios en authelia estaban en el fichero _users_database.yml_. Podemos indicar a authelia que obtenta los usuario (_user federation_) de un directory de información ldap/lldap.
Las partes relevantes del fichero de configuración se indican a continuación.


```bash
#/etc/authelia/configuration.yaml
authentication_backend:
  ldap:
    address: 'ldap://localhost:3890'
    implementation: 'lldap'
    base_dn: 'dc=example,dc=com'
    user: 'uid=authelia,ou=people,dc=example,dc=com'
    password: '12345678'
```

> **note** Note
> Deberemos crear en lldap un usuario authelia para poder establecer la conexión




TODO----
## Integración [[tutorials/ldap]]

Es necesario manejar los usuarios mediante una base de datos, usaremos postgresql. Una vez instalada hay que crear el usuario y la base de datos.
```bash
sudo -i -u postgres psql
CREATE DATABASE keycloak;
CREATE USER keycloak WITH ENCRYPTED PASSWORD 'my-db-password';
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;
\q
```

## Integración zammad

Para autenticar usuarios de zammad en authelia medianto OIDC, debemos crear el cliente _OIDC_ denominado _zammad_.

```bash
{% raw %}
#/etc/authelia/configuration.yaml
identity_providers:
  oidc:
    jwks:
      - key_id: 'main-key'
        algorithm: 'RS256'
        use: 'sig'
        key: {{ secret "/etc/authelia/ssl/private_pkcs8.pem" | mindent 10 "|" | msquote }}
    clients:
        - client_id: 'zammad'
          client_name: 'Zammad'
          public: true
          authorization_policy: 'one_factor'
          require_pkce: true
          pkce_challenge_method: 'S256'
          redirect_uris:
            - 'https://zammad.incus/auth/openid_connect/callback'
          scopes:
            - 'openid'
            - 'email'
            - 'profile'
          response_types:
            - 'code'
          grant_types:
            - 'authorization_code'
          access_token_signed_response_alg: 'none'
          userinfo_signed_response_alg: 'none'
          token_endpoint_auth_method: 'none'
{% endraw %}
```

> **warning** Warning
{% raw %}
> El fichero de configuración anterior añade una la línea `key: {{ secret "/etc/authelia/ssl/private_pkcs8.pem" | mindent 10 "|" | msquote }} ` que es una característica experimental.
{% endraw %}
  > Para que _authelia_ arranque de forma correcta debemos añadir a la linea de arranque del servicio authelia _/usr/lib/systemd/system/authelia.service_ la opción filter template
}
> `ExecStart=/usr/bin/authelia --config /etc/authelia/configuration.yml --config.experimental.filters template` 

## Links
* [Simple Self-Hosted Security with Authelia](https://www.apalrd.net/posts/2024/ultimate_authelia/)
* [How to Configure OIDC JWKS in Authelia: A Step-by-Step Guide](https://devdenvino.medium.com/how-to-configure-oidc-jwks-in-authelia-a-step-by-step-guide-8509861c847a)
* [Authelia OIDC](https://www.authelia.com/configuration/identity-providers/openid-connect/provider/)
* [Zammad OIDC integration](https://www.authelia.com/integration/openid-connect/clients/zammad/)
* [lldap integration](https://www.authelia.com/integration/ldap/lldap/)
* [ldap integration](https://www.authelia.com/integration/ldap/introduction/)