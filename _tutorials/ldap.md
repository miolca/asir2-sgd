---
layout: default
title: ldap
---
# OpenLDAP



Solicitar el certificado para TLS
```bash
step ca  certificate ldap.local --san=10.149.165.198 \
	--san 192.168.10.221 --san=ldap \
	--san=localhost --san=127.0.0.1 \
	/etc/ldap/ldap.crt /etc/ldap/ldpa.key 
```





TODO

## Actividad I. Autenticación en un servidr ssh mediante kerberos
### Guión esquemático


```bash
$ incus network create krb-net \
      ipv4.address=10.144.144.1/24 \
      ipv6.address=none ipv4.nat=true \
      ipv4.dhcp.ranges=10.144.144.100-10.144.144.200 \
      dns.nameservers=10.144.144.2 \
      dns.search=asir2.grao
```

### DNS

> [!TIP]
> Podemos asignar de forma temporal un servidor de nombres para poder instalar los paquetes mediante el comando _resolvectl dns eth0 10.144.144.1_

Intala los paquetes necesarios.
```bash
$ incus exec dnssrv -- bash -c 'apt-get update && apt-get -y install  aptitude wget bind9 dnsutils bash-completion nano xsel vim'
```

**Responde:**


> [!NOTE]
> Las siguientes indicaciones están basada sen el [tutorial](https://documentation.ubuntu.com/server/how-to/networking/install-dns/#install-dns) de configuración del servicio DNS de ubuntu. Se recomienda consultarlas para cualquier duda o aclaración.


#### Comprobación y depuración



**Depuración**
En caso de que la práctica no funciona como fuera de esperar es conveniente consutar los mensajes de depuración tanto del cliente como del servidor.
Ejecuta en el servidor el servicio sshd en modo _debug_ (muestra mensajes adicionales en la salida estandar) en un puerto diferente.
```bash
ssh-server$ /usr/sbin/sshd -d -d -d -p 2223
```
Y en el cliente conectate activando tambien el modo _debug_ al nuevo puerto del servidor ssh.
```bash
krb-cli$ ssh -vv ubuntu@ssh-server.asir2.grao -p 2223
```

Observa los mensajes de ambos programas para obtener más información de por que _kerberos_ no es capaz de autenticarse de forma correcta.


## Entrega
* Entrega un documento **estructurado** con una breve explicación(función) de los comando usados para la configuración tanto del cliente como del servidor así como de los pasos llevados a cabo para la comprobación de correcto funcionamiento de la actividad.
* Completa el documento con un diagrama de red **hecho  a mano** de la red de la actividad.
* Completa el documento con un diagrama temporal **hecho a mano**, similar al estudiado en clase que explique de forma gráfica los pasos para autenticarse mediante el protocolo _kerberos_ a un servicio. 
	* La clave del usuario es de color naranja
	* La clave del servicio (MySql) es de color morado
	* Enumera los pasos llevados a cabo y una breve explicación de lo que hace cada uno y que se obtiene.
* Completa el documento con anexos de los ficheros de configuración (sin comentario ni espacios en blanco) usados.
* Responde a las preguntas propuestas durante la realización de la práctica.

Utiliza el formato _markdown_ adecuado para tener un documento estructurado y legible.

### Propuestas de mejora
Las siguientes propuestas de mejora de la práctica se plantean al alumno como reto para que mejore sus destrezas y conocimientos de las herramientas de administrdor de sistemas y mejore su nota en la asignatura.

* **Mejora I-Obtención del KDC a través de DNS queries** Añade las entradas correspondienes en el servidor DNS para obtener la ip del servidor KDC mediante consultas DNS
* **Mejora II-Kerberizando servicios** Mejora tus conocimientos sobre kerberos así como tu calificación en la asignatura llevando a cabo la _kerberización_ del servidor de un servicio, que pueden ser un servidor web, una base de datos(MySQL, Hive, Spark) un servicio de carpetas compartidas (NFS, Samba) u otro servicio a tu elección. Busca información reslacionada en internet con las siguienes palabra clave _<servicio> kerberized_.
* **Mejora III-Automatización** Automatiza la creación y configuración del escenario mediante un _taskfile.yaml_ mejorando notablemente tus conocimiento de _scripting_. Da un paso más allá y prueba a automatizar la configuración del servidor DNS así como la del servidor _ssh_



## Links
* [How to set up SSSD with LDAP and Kerberos](https://documentation.ubuntu.com/server/how-to/sssd/with-ldap-and-kerberos/)
* [How to Integrate LDAP and Kerberos](https://www.linuxtoday.com/blog/integrate-ldap-kerberos/)




## Anexo I. Glosario de términos
* _KDC_ : Key Distribution Center. Cerebro de Kerberos. Está dividido en tres partes ejecutados en un mismo proceso.
	* _Database_: Contiene los _principals_ y sus contraseñas asociadas. Las contraseñas no son usadas directamente, se genera una _key_ a partir de la contraseña (salt) que es usada para encriptar mensajes.
	* _Authentication Server (AS)_: Punto de entrada para el cliente cuando interactua con el protocolo de autenticación Kerberos. Emite _TGT_ 
	* _Ticket Granting Service (TGS)_: Responsable de emitir al cliente un tiquet que permite al cliente autenticarse en un servicio. Emite _Service Tickets_. El cliente envía al TGS el TGT obtenido del AS y solicita al TGS un tiquet para un servicio en concreto.
* _keytab_: fichero que contiene datos encriptados como las contraseñas para los _pricipals_ (usuarios o servicios)
* _principal_: Cualquier entidad dentro del dominio Kerberos (usuarios, hosts y servicios) posee un identificador único así como una contraseña. Se identifica mediante una cadena de texto para la cual han sido asignadas las credenciales (tiquet). El formato típico para un _principal_ suele ser _primary/instance@REALM_
	* _primary_: La primera parte es, en el caso de un usuario es su nombre de usuario, en caso de un servicio el nombre del servicio.
	* instance: La segunda parte da información sobre la primera parte, en caso de ser un usuario suele ser el rol del usuario (user/admin). En caso de un host suele ser su FQDN (webserver.example.com)
	* _Realm_: Podemos entender un _Kerberos realm_ como el domino  en el cual un Kerberos AS tiene la autoridad de autenticar usuarios, hosts, o servicios. Suele estar formada por una única base de datos Kerberos y uno o más KDC. Por convenio, el _realm_ suele nombrarse en mayúsculas para diferenciarlo de un domino de internet.
* _cliente_: Entidad que obtiene un tiquet, normalmente un usuario o un host
* _ticket_: Credenciales electrónicas temporales que verifican la identidad del cliente para un servicio en particular
* _TGT_: Ticket-Granting Ticket: Un tiquet especial que permite al cliente obtener tickets adicionales dentro del mismo _realm_. Este TGT permite al usuario acceder a varios servicios sin necesidad de introducir la contraseña estableciendo comunicaciones seguras y capacidades de single sign-on(SSO)

## Anexo II. Misc comands
```bash
incus network set incusbr0 dns.domain asir2.grao
getent hosts 10.149.165.99 
resolvectl dns <network interface> <dns_address>
resolvectl domain <network interfacee> ~<dns_domain>
```