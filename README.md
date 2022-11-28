# ProyectoApache

## Imagen necesaria

Para montar el  contenedor de nuestro servidor **Apache** con soporte de **PHP** debemos ir a la documentación de docker donde podemos conseguir el susodicho contenedor de [Servidor Apache + PHP](https://hub.docker.com/_/php/).


## Configuración Docker-compose

Para levantarlo debemos configurar el fichero ***docker-compose.yml*** donde anotaremos lo siguiente:


* **Version**

Indicamos la version del VSC

```
version: '3.9'
```

* **Services**

Nombre de nuestro servicio

```
services:
asir_apache:
```

* **Image & Container_name**

Nombre del contenedor y el nombre de su imagen (incluyendo su versión)

```
image: php:7.4-apache
container_name: asir-apache
```

* **Ports**
Los puertos para realizar la conexión
```
ports:
- '80:80'
- '8000:8000'
```
## Carpetas de sitios

Para poder usar los documentos.html/.php tememos que crear 2 directorios, 1 para cada uno:

_site1>index.html_

_site2>index.php_

## Mapeos

Para mapear nuestro sitio y usarlo en el server de Apache, tenemos que indicar esto en el **docker-compose.yml//volumes:**

* **Volumes**

Indicamos que nuestro directorio Sitio1 se corresponde con el /var/www/html de Apache.

``` 
volumes:
- ./html:/var/www/html
- ./confApache:/etc/apache2
```
## Conexión mediante Localhost

La forma que vamos a usar para conectarnos al arhivo html y php va a ser mediante uso de _Localhost:80_    y   _Localhost:8000_

* **Configuración de los puertos**

Es una actividad sencilla, debemos modificar el archivo **000-default.conf** y en **DocumentRoot** añadir el directorio del archivo que queremos que se muestre al realizar la conexón _Localhost_ con un puerto:

```
<VirtualHost *:80>

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html/site1
```
Con esta configuración, si nos dirigimos a nuestro navegador y escribimos _localhost:80_, encontraremos el mensaje escrito en _index.html_  **"Hola mundo"**.

Ahora si deseamos conseguir lo mismo pero con el mensaje de _index.php_ debemos copiar el contenido de **000-default.conf** pero cambiando el directorio root y el puerto y dejarlo en la misma carpeta con un nombre diferente, en mi caso se llama **002-default.conf**

```
<VirtualHost *:8000>
	
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html/site2
```

**_IMPORTANTE_** 

Estos dos archivos los podemos encontrar en **confApache->sites-avaliable**, pero deben estar en **sites-enabled** también,para hacer esto debemos hacer click derecho sobre el contenedor, y elegir _attach shell_, nos saltará una terminal del docker donde deberemos escribir _a2ensite(nombre del fichero)_ para que este se inserte. Por último, bajaremos y volveremos a subir el docker para que se apliquen los cambios.

# Integración servidor DNS

Primeramente para levantar el contenedor con un servidor de DNS y personalizarlo para nuestro caso particular, utilizaremos nuestro anterior proyecto [```Servidor DNS en Docker```].

Por lo que antes de nada deberemos mover todas las carpetas y archivos de dicho proyecto a [```proyectoApache```], es decir, la carpeta [```conf```] y la carpeta [```zonas```].

Antes de centrarnos en modificar los ficheros de configuración de fichero apache vamos a implementarlo en nuestro [```docker-compose.yml```].

Para ello deberemos crear una nueva red y asignar a cada contenedor una IP diferente.

* **Contenedor bind9 con los sitios (Servidor DNS)**

*Añadimos el apartado ```networks``` y asignamos la IP 10.1.0.254 creada previamente denominada ```bind9_subnet``` al igual que mapearemos los directorios del servidor DNS* .

```yml
 bind9:
    container_name: asir1_bind9
    image: internetsystemsconsortium/bind9:9.16
    ports:
      - 5300:53/udp
      - 5300:53/tcp
    networks:
      bind9_subnet:
        ipv4_address: 10.1.0.254
    volumes:
      - ./conf:/etc/bind
      - ./zonas:/var/lib/bind

```

* **Contenedor servidor Apache**

*En este caso añadimos nuevamente como en el caso del servidor DNS la misma network pero en este caso con la IP ```10.1.0.253```*

```yml
services:
  apache:
    container_name: apachePHP
    image: php:7.2-apache
    networks:
      bind9_subnet:
        ipv4_address: 10.1.0.253
    ports:
      - '80:80'
      - '8000:8000'
  
    volumes:
      - ./html:/var/www/html
      - ./confApache:/etc/apache2

```

Posteriormente nos centramos en los ficheros de configuración, vamos al apartado de [```zonas```] y renombramos el archivo que hay en su interior a [```db.fabulas.com```]

```yml
$TTL    3600
@       IN      SOA     ns.fabulas.com. root.fabulas.com. (
                   2007010401           ; Serial
                         3600           ; Refresh [1h]
                          600           ; Retry   [10m]
                        86400           ; Expire  [1d]
                          600 )         ; Negative Cache TTL [1h]
;
@       IN      NS      ns.fabulas.com.
ns       IN      A       10.1.0.254

maravillosas     IN      A       10.1.0.253
oscuras     IN      CNAME       maravillosas

```

En el especificaremos en el ```ns``` la IP ```10.1.0.254``` que se corresponde con la IP de nuestro servidor DNS, que es quien va a resolver.

```yml
ns       IN      A       10.1.0.254

```

Posteriormente especificamos que ```maravillosas``` se corresponda con la IP del servidor apache ```10.1.0.253```.

```yml
maravillosas     IN      A       10.1.0.253

```
Indicaremos un CNAME de manera que el subdominio ```maravillosas.fabulas.com``` resuelva a ```oscuras.fabulas.com```.

```yml
oscuras     IN      CNAME       maravillosas

```

Antes de nada debemos volver a nuestros ficheros de configuración en [```confApache```] y editaremos:

* [**sites-available**]

En ambos casos debemos descomentar la línea 9 correspontiente al ```ServerName``` y cambiar su contendio al subdominio que se corresponda.

Ejemplificamos con el caso del **sitio1** que denominaremos como ```maravillosas.fabulas.com``` en el fichero [```000-default.conf```] y lo mismo sería con el **sitio2** que lo denominaríamos como ```oscuras.fabulas.com``` y se corresponde con [```002-default.conf```]

```xml
<VirtualHost *:80>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	ServerName maravillosas.fabulas.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html/sitio1

	# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
	# error, crit, alert, emerg.
	# It is also possible to configure the loglevel for particular
	# modules, e.g.
	#LogLevel info ssl:warn

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

	# For most configuration files from conf-available/, which are
	# enabled or disabled at a global level, it is possible to
	# include a line for only one particular virtual host. For example the
	# following line enables the CGI configuration for this host only
	# after it has been globally disabled with "a2disconf".
	#Include conf-available/serve-cgi-bin.conf
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet


```

* [**```sites-enabled```**]

Replicamos la modificación de los ficheros anteriores con los correspondientes con este directorio, tanto para el [```000-default.conf```] como con [```002-default.conf```]

# Añadir firefox para comprobar funcionamiento

Para añadir firefox como un contenedor y poder comprobar el funcionamiento de que nuestro Servidor DNS resuelve correctamente.

Para ello debemos añadirlo a nuestro [```docker-compose.yml```]
```yml

 firefox:
    container_name: firefox
    image: jlesage/firefox
    ports: 
      - '5800:5800'
    volumes:
      - ./firefox:/config:rw
    dns:
      - 10.1.0.254
    networks:
      bind9_subnet:
        ipv4_address: 10.1.0.252

```
Donde le asignaremos una nueva IP dentro de la misma red ```10.1.0.252``` y mapearemos su contenido de configuración donde indicamos con:

```yml
    volumes:
      - ./firefox:/config:rw

```
No recomendable si no vamos a configurarlo y no queremos +1000 ficheros de configuración.

Una vez levantamos el servicio si accedemos desde nuestro buscador mediante el puerto que hemos indicado ```localhost:5800``` podemos realizar la comprobación de que nuestro servidor de DNS resuelva correctamente.


# Añadir SSL

Primeramente comprobamos que en el fichero [```ports.conf```] que el  puerto **:443** (el propio de SSL) esté habilitado.

Posteriormente accedemos mediante "Atach shell" presionando click derecho en nuestro contenedor apache y escribimos en el terminal el siguiente comando para habilitar el SSL:

```s
$ a2enmod ssl

```

Nos indicará que debemos eliminar diferentes ficheros de nuestra configuración local (la que mapeamos) de [**```mods-enabled```**] que debemos eliminar ya que solo son necesarias las del propio volumen de apache (reiniciamos el servicio cada vez que eliminamos).

Creamos una carpeta dentro de [```confApache```] llamada [**```certs```**] donde generaremos el certificado con el comando:


```s
openssl req -new -newkey rsa:4096 -x509 -sha256 -days 365 -nodes -out apache-certificate.crt -keyout apache.key

```

En el especificaremos todos los parámetros que se nos indica, se generarán dos archivos, uno denominado ```apache-certificate.crt``` y ```apache.key```, por seguridad github no nos dejará subir dichos ficheros.


Posteriormente dentro de nuestra carpeta [**```html```**] creamos una nueva llamada [**```SSL```**] donde almacenaremos un [**```index.html```**] para comprobar finalmente el funcionamiento de nuestro SSL.

Para habilitar el SSL del virtualhost utilizaremos los siguientes comandos:

```s
$ a2ensite ssl
$ a2ensite default-ssl


```

 
Seguidamente vamos al fichero de configuración [**```default.ssl.conf```**] en [```sites-available```] y especificaremos en el DocumentRoot la ruta de nuestro directorio que tendrá el SSL (Línea 12) y las rutas de nuestros dos archivos del certificado que hemos generado (Linea 32 y Linea 33).

```xml
<IfModule mod_ssl.c>
	<VirtualHost _default_:443>
		ServerAdmin webmaster@localhost

		DocumentRoot /var/www/html/ssl

		# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
		# error, crit, alert, emerg.
		# It is also possible to configure the loglevel for particular
		# modules, e.g.
		#LogLevel info ssl:warn

		ErrorLog ${APACHE_LOG_DIR}/error.log
		CustomLog ${APACHE_LOG_DIR}/access.log combined

		# For most configuration files from conf-available/, which are
		# enabled or disabled at a global level, it is possible to
		# include a line for only one particular virtual host. For example the
		# following line enables the CGI configuration for this host only
		# after it has been globally disabled with "a2disconf".
		#Include conf-available/serve-cgi-bin.conf

		#   SSL Engine Switch:
		#   Enable/Disable SSL for this virtual host.
		SSLEngine on

		#   A self-signed (snakeoil) certificate can be created by installing
		#   the ssl-cert package. See
		#   /usr/share/doc/apache2/README.Debian.gz for more info.
		#   If both key and certificate are stored in the same file, only the
		#   SSLCertificateFile directive is needed.
		SSLCertificateFile	/etc/apache2/certs/apache-certificate.crt
		SSLCertificateKeyFile /etc/apache2/certs/apache.key

		#   Server Certificate Chain:
		#   Point SSLCertificateChainFile at a file containing the
		#   concatenation of PEM encoded CA certificates which form the
		#   certificate chain for the server certificate. Alternatively
		#   the referenced file can be the same as SSLCertificateFile
		#   when the CA certificates are directly appended to the server
		#   certificate for convinience.
		#SSLCertificateChainFile /etc/apache2/ssl.crt/server-ca.crt

		#   Certificate Authority (CA):
		#   Set the CA certificate verification path where to find CA
		#   certificates for client authentication or alternatively one
		#   huge file containing all of them (file must be PEM encoded)
		#   Note: Inside SSLCACertificatePath you need hash symlinks
		#		 to point to the certificate files. Use the provided
		#		 Makefile to update the hash symlinks after changes.
		#SSLCACertificatePath /etc/ssl/certs/
		#SSLCACertificateFile /etc/apache2/ssl.crt/ca-bundle.crt

		#   Certificate Revocation Lists (CRL):
		#   Set the CA revocation path where to find CA CRLs for client
		#   authentication or alternatively one huge file containing all
		#   of them (file must be PEM encoded)
		#   Note: Inside SSLCARevocationPath you need hash symlinks
		#		 to point to the certificate files. Use the provided
		#		 Makefile to update the hash symlinks after changes.
		#SSLCARevocationPath /etc/apache2/ssl.crl/
		#SSLCARevocationFile /etc/apache2/ssl.crl/ca-bundle.crl

		#   Client Authentication (Type):
		#   Client certificate verification type and depth.  Types are
		#   none, optional, require and optional_no_ca.  Depth is a
		#   number which specifies how deeply to verify the certificate
		#   issuer chain before deciding the certificate is not valid.
		#SSLVerifyClient require
		#SSLVerifyDepth  10

		#   SSL Engine Options:
		#   Set various options for the SSL engine.
		#   o FakeBasicAuth:
		#	 Translate the client X.509 into a Basic Authorisation.  This means that
		#	 the standard Auth/DBMAuth methods can be used for access control.  The
		#	 user name is the `one line' version of the client's X.509 certificate.
		#	 Note that no password is obtained from the user. Every entry in the user
		#	 file needs this password: `xxj31ZMTZzkVA'.
		#   o ExportCertData:
		#	 This exports two additional environment variables: SSL_CLIENT_CERT and
		#	 SSL_SERVER_CERT. These contain the PEM-encoded certificates of the
		#	 server (always existing) and the client (only existing when client
		#	 authentication is used). This can be used to import the certificates
		#	 into CGI scripts.
		#   o StdEnvVars:
		#	 This exports the standard SSL/TLS related `SSL_*' environment variables.
		#	 Per default this exportation is switched off for performance reasons,
		#	 because the extraction step is an expensive operation and is usually
		#	 useless for serving static content. So one usually enables the
		#	 exportation for CGI and SSI requests only.
		#   o OptRenegotiate:
		#	 This enables optimized SSL connection renegotiation handling when SSL
		#	 directives are used in per-directory context.
		#SSLOptions +FakeBasicAuth +ExportCertData +StrictRequire
		<FilesMatch "\.(cgi|shtml|phtml|php)$">
				SSLOptions +StdEnvVars
		</FilesMatch>
		<Directory /usr/lib/cgi-bin>
				SSLOptions +StdEnvVars
		</Directory>

		#   SSL Protocol Adjustments:
		#   The safe and default but still SSL/TLS standard compliant shutdown
		#   approach is that mod_ssl sends the close notify alert but doesn't wait for
		#   the close notify alert from client. When you need a different shutdown
		#   approach you can use one of the following variables:
		#   o ssl-unclean-shutdown:
		#	 This forces an unclean shutdown when the connection is closed, i.e. no
		#	 SSL close notify alert is send or allowed to received.  This violates
		#	 the SSL/TLS standard but is needed for some brain-dead browsers. Use
		#	 this when you receive I/O errors because of the standard approach where
		#	 mod_ssl sends the close notify alert.
		#   o ssl-accurate-shutdown:
		#	 This forces an accurate shutdown when the connection is closed, i.e. a
		#	 SSL close notify alert is send and mod_ssl waits for the close notify
		#	 alert of the client. This is 100% SSL/TLS standard compliant, but in
		#	 practice often causes hanging connections with brain-dead browsers. Use
		#	 this only for browsers where you know that their SSL implementation
		#	 works correctly.
		#   Notice: Most problems of broken clients are also related to the HTTP
		#   keep-alive facility, so you usually additionally want to disable
		#   keep-alive for those clients, too. Use variable "nokeepalive" for this.
		#   Similarly, one has to force some clients to use HTTP/1.0 to workaround
		#   their broken HTTP/1.1 implementation. Use variables "downgrade-1.0" and
		#   "force-response-1.0" for this.
		# BrowserMatch "MSIE [2-6]" \
		#		nokeepalive ssl-unclean-shutdown \
		#		downgrade-1.0 force-response-1.0

	</VirtualHost>
</IfModule>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet

```

Al modificar dicho archivo posteriormente reiniciamos el contenedor de apache. Por último para poder comprobar en nuestro localhost el SSL debemos habilitar en el contenedor el puerto **443** desde el [```docker-compose.yml```]

```yml
services:
  apache:
    container_name: apachePHP
    image: php:7.2-apache
    networks:
      bind9_subnet:
        ipv4_address: 10.1.0.253
    ports:
      - '80:80'
      - '8000:8000'
      - '443:443'
  
    volumes:
      - ./html:/var/www/html
      - ./confApache:/etc/apache2

```

Ejecutamos un:

```s
$ docker-compose down -v
$ docker-compose up

```
Y si entramos en nuestro buscador y realizamos la consulta ```https://localhost/``` ya nos saltará nuestro certificado, eso sin sin validar ya que lo hemos generado nosotros mismos.


# Añadir Wireshark

Para ello añadimos a nuestro [```docker-compose.yml```]

```yml
  wireshark:
    image: lscr.io/linuxserver/wireshark:latest
    container_name: wireshark
    cap_add:
      - NET_ADMIN
    security_opt:
      - seccomp:unconfined #optional
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
    volumes:
      - ./wireshark:/config
    ports:
      - 3000:3000 #optional
    restart: unless-stopped

```
Creamos una carpeta llamada ```wireshark``` que es en donde mapearemos el volumen como indicamos en el [```docker-compose.yml```] y posteriormente ejecutamos:

```s
$ docker-compose down -v
$ docker-compose up

```

Y si probamos a acceder desde ```https://localhost:3000``` podemos comprobar que ya tenemos nuestro **Wireshark** funcionando.

# Añadir autentificación

Para añadir la autentificación primero crearemos una carpeta denominada [**```pass```**] dentro de [**```confApache```**]

Una vez creada entraremos al contenedor Apache haciendo click derecho en ```Atach-Shell```, una vez dentro ejecutaremos el comando:

```
 htpasswd /etc/apache2/pass/password rolando

```

En el fichero que indicamos en dicha ruta se guardará una contraseña cifrada que nosotros previamente indiquemos por pantalla, vinculado al usuario que indicamos.

Posteriormente accedemos a [```default-ssl.conf```]( donde modificaremos el fichero añadiendiendo el siguiente trozo de código siguiendo el [```Tutorial```](https://httpd.apache.org/docs/2.4/mod/core.html#directory):

```xml

		<Directory /var/www/html/ssl>
		AuthType Basic
		AuthName "Restricted Files"
		AuthUserFile /etc/apache2/pass/password
		Require user rlv123
		</Directory> 

```

Donde indicamos en ```Directoy``` la ruta de nuestro sitio, posteriormente en ```AuthUserFile``` la ruta de nuestro fichero con la password cifrada y por último modificamos el ```Require user``` indicando nuestro usuario.

Realizamos un **restart** del contenedor de Apache y ya podemos comprobar su funcionamiento.

