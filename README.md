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

La forma que vamos a utilizar para conectarnos al arhivo html y php va a ser mediante uso de _Localhost:80_    y   _Localhost:8000_

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
