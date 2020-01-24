# Despliegue de aplicaciones - Módulo DevOps

El objetivo de esta práctica es el despliegue de dos aplicaciones en una instancia de AWS. Cada uno de estos despliegues se realizará en dos ejercicios distintos.

Se adjunta el fichero .pdf con el enunciado de la praćtica para facilitar su lectura en caso de ser necesario.

* ***Nota importante:***

Aunque no se requiere ni se pide en la práctica, en esta memoria, a nivel de recordatorio, se irá apuntando qué pasos se han seguido durante la realización de cada ejercicio para poder recordarlo de cara a futuro y tenerlo siempre a mano.


## **Instalar nginx**
* Actualizar repositorios usando `sudo apt update` y `sudo apt upgrade`

* Instalar nginx mediante `sudo apt install nginx -y`
* Abrir el puerto 80 en el firewall de AWS o del proveedor de hosting.

	Con esto ya podríamos ver la página por defecto de nginx en la DNS o IP pública de nuestro servidor.

## **Ejercicio 1**
Como vamos a subir una aplicación que hace uso de Node y de MongoDB tenemos que gestionar los requisitos de estas dependencias. Además, usaremos PM2 para gestionar las distintas aplicaciones que tengamos desplegadas.

La dirección en la que podemos ver este ejercicio desplegado es: **`http://ec2-15-188-195-222.eu-west-3.compute.amazonaws.com/`**

### Instalando Node
* Crear usuario node y bloquear acceso vía SSH

  * Creamos el usuario con el comando: `sudo adduser nodeUser`

  * Bloqueamos el acceso desde el exterior con: `sudo passwd -l nodeUser` (-l proviene de 'lock')

  * Logueamos internamente como nodeUser con: `sudo -u nodeUser -i` (-u proviene de user y -i de interactive)

  * Instalamos nvm para que podamos instalar node con su manager de versiones, desde [esta doc](https://github.com/nvm-sh/nvm#install--update-script).
  * Instalamos node con: `nvm install node`. Utilizar siempre versiones LTS en la medida de lo posible.

### Descargamos la aplicación o el proyecto a desplegar
* Descargamos del repositorio con: `git clone https://github.com/lb12/nodepop` en la carpeta del usuario 'nodeUser' (`~`).

### Instalando y configurando PM2
* Instalamos PM2 de forma global con: `npm i pm2 -g`
* En la carpeta del usuario 'nodeUser' (`~`) creamos la configuración de PM2 para los procesos node que va a gestionar con: `pm2 ecosystem`.
* Antes de editar el archivo debemos conocer cuál es la versión de node que utilizamos y la ruta del ejecutable, así como el script que arranca nuestra aplicación.
  * *Ruta de node*: `nvm which current` nos devuelve la ruta del ejecutable de la versión que usamos actualmente de node.
  * *Script que arranca*: mirar el apartado "scripts" del package.json que es donde se indica cómo arranca la aplicación.
* Cambiamos el contenido de `/home/nodeUser/ecosystem.config.js` para dejarlo:
	``` 
	module.exports = {
		apps: [
			{
				"name": "Nodepop",
				"cwd": "./nodepop",
				"script": "bin/www",
				"exec_interpreter": "/home/node/.nvm/versions/node/v13.5.0/bin/node"
			}
		]
	}
	```
* Arrancamos la aplicación con: `pm2 start ecosystem.config.js`
* Configurar para que PM2 arranque la aplicación cuando se inicie el sistema:
  * Guardar la configuración de PM2: `pm2 save`
  * Obtener el comando que nos permitirá arrancar PM2 al inicio con: `pm2 startup`. 
  * Una vez hemos copiado el comando devuelto, salimos de la sesión de nodeUser con `logout` y ejecutamos el comando desde un superusuario: `sudo $comandoAnterior`
  * Si quisieramos eliminar ese script de arranque de PM2 podríamos hacerlo usando: `pm2 unstartup systemd`

### Configurando nginx como proxy inverso
* Crear el server block de la aplicación node como un **superusuario**: `sudo nano sites-available/nodepop`. Dentro del fichero añadimos el siguiente código:
  
  ```
  server {
	  server_name DNS_Publica_o_Dominio; # indicamos que atienda peticiones bajo esa DNS o ese dominio y aplique las reglas de abajo
	  location / {
		  proxy_set_header Host $http_host;
		  proxy_pass http://127.0.0.1:3000; # le pasamos las peticiones a la app de Nodepop
		  proxy_redirect off;
	  }
  }
  ```
* Crear un symbolic-link a la carpeta de sites-enabled para habilitar este server block: `sudo ln -s /etc/nginx/sites-available/nodepop /etc/nginx/sites-enabled/nodepop`
* Comprobamos la configuración de nginx con `sudo nginx -t` para verificar que esté todo ok.
* Si ocurriese un error de que el hash de server_name es muy largo, lo podemos resolver cambiando en la configuración de nginx (`sudo nano /etc/nginx/nginx.conf`) descomentando `server_names_hash_bucket_size 64` y modificándolo por 128 para que no nos quedemos cortos.
* Recargamos nginx para que todas las configuraciones se apliquen: `sudo service nginx reload`
* Aunque nodepop no hace uso de los WebSockets, lo dejo aquí por si acaso en un futuro se consultase este manual. Para que los WebSockets funcionen correctamente hay que añadir en el `location /` de la app las siguientes 3 líneas de código y recargar nginx con `sudo service nginx reload`:
	```
	proxy_http_version 1.1;
	proxy_set_header Upgrade $http_upgrade;
	proxy_set_header Connection "Upgrade";  
	```

### Configurando nginx como servidor de archivos estáticos
* Abrir el server block de la aplicación node y añadir la siguiente regla **ANTES** de `location / { ... }` que habíamos escrito antes:
	```
	location ~ ^/(images|stylesheets)/ {
		root /home/nodeUser/nodepop/public;
		add_header X-Owner lb12; #esto es solo para demostrar que nginx sirve estos archivos estáticos
		access_log off; # no nos interesa que se muestren en los logs
		expires max;
	}
	```
	Con esa regla le estamos diciendo que todo path que empiece por `/images` ó `/stylesheets`, lo sirva nginx y va a encontrar el contenido en la ruta `/home/nodeUser/nodepop/public`
* Comprobar la configuración con: `sudo nginx -t`
* Recargar el servicio de nginx con: `sudo service nginx reload`

### Eliminar el número de versión de nginx
* Descomentar en `/etc/nginx/nginx.conf` la línea: `server_tokens off;`
* Recargamos nginx para que todas las configuraciones se apliquen: `sudo service nginx reload`

### Instalando y configurando MongoDB
* Instalar MongoDB como se indica [aquí](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/#install-mongodb-community-edition) desde un **superusuario**.
  * Importar la clave pública usada por el sistema gestor de paquetes: `wget -qO - https://www.mongodb.org/static/pgp/server-4.2.asc | sudo apt-key add -`
  * Crear una lista de archivos para MongoDB: `echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.2.list`
  * Recargar la BD de paquetes locales: `sudo apt update`
  * Instalar todos los paquetes de MongoDB: `sudo apt-get install -y mongodb-org`
  * Para que MongoDB arranque como servicio hay que hacer: `sudo systemctl enable mongod.service`
  
## **Ejercicio 2**

Dirección IP en la que tengo la práctica que hice en Fundamentos de HTML y CSS: **`15.188.195.222`**

Se detalla a continuación todo el proceso seguido para llevar a cabo la práctica:


* Subir al servidor la carpeta con toda nuestra aplicación

Para subir la aplicación a la carpeta home del usuario administrador utilizamos el siguiente comando: 
`scp -i clave.pem -r nombreCarpeta usuarioAdmin@direccionServidor:~/`

* Crear Server Block en nginx

Para que nginx sepa cómo servir la aplicación, necesitamos indicarle la información en un server block. Esto lo hacemos creando un nuevo archivo en la carpeta `sites-available` en la ruta `/etc/nginx/sites-available/`. Se crea el archivo `web-ej-2` con el siguiente contenido:

```
server {
	listen 80 default_server;
	# no incluimos el server_name para que responda directamente por la IP de la máquina
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