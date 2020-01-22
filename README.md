# Despliegue de aplicaciones - Módulo DevOps

El objetivo de esta práctica es el despliegue de dos aplicaciones en una instancia de AWS. Cada uno de estos despliegues se realizará en dos ejercicios distintos.

Se adjunta el fichero .pdf con el enunciado de la praćtica para facilitar su lectura en caso de ser necesario.

Aunque no se requiere ni se pide en la práctica, esta memoria a nivel de recordatorio, se irá apuntando qué pasos se han seguido durante la realización de cada ejercicio para poder recordarlo de cara a futuro y tenerlo siempre a mano.


## Instalar nginx
* Actualizar repositorios
  
`sudo apt update`

`sudo apt upgrade`

* Instalar nginx

`sudo apt install nginx -y`
* Abrir el puerto 80 en el firewall de AWS.

Con esto ya podríamos ver la página por defecto de nginx en la DNS o IP pública de nuestro servidor.

## Ejercicio 1
### # TODO


## Ejercicio 2

Dirección IP en la que tengo la práctica que hice en Fundamentos de HTML y CSS: `15.188.195.222`

Se detalla a continuación todo el proceso seguido para llevar a cabo la práctica:


* Subir al servidor la carpeta con toda nuestra aplicación

Para subir la aplicación a la carpeta home del usuario administrador utilizamos el siguiente comando: 
`scp -i clave.pem -r nombreCarpeta usuarioAdmin@direccionServidor:~/`

* Crear Server Block en nginx

Para que nginx sepa cómo servir la aplicación, necesitamos indicarle la información en un server block. Esto lo hacemos creando un nuevo archivo en la carpeta `sites-available` en la ruta `/etc/nginx/sites-available/`. Se crea el archivo `web-ej-2` con el siguiente contenido:

```
server {
	listen 80 default_server;
	root /home/david/web-ej-2;
	index index.html;
	location / {
		try_files $uri $uri/ =404;
	}
}
```

* Crear un enlace simbólico (ln) en sites-enabled

Para activar este server block, se tiene que crear un ln de la carpeta de la web en sites-enabled:
`sudo ln -s /etc/nginx/sites-available/web-ej-2 /etc/nginx/sites-enabled/web-ej-2`

* Eliminar la carpeta que hay por defecto en sites-available

Para que no haya conflicto de servidores principales y/o por defecto, hay que eliminar la carpeta `default` de sites-available (también incluye la sentencia `default_server`).

`sudo rm -rf default`

* La página ya se debería mostrar en el navegador

## Extras opcionales
Realizar los ejercicios 1 y 2 pero utilizando Docker.