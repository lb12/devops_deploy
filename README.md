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

### Instalando Node
* Crear usuario node y bloquear acceso vía SSH

  * Creamos el usuario con el comando: `sudo adduser nodeUser`

  * Bloqueamos el acceso desde el exterior con: `sudo passwd -l nodeUser` (-l proviene de 'lock')

  * Logueamos internamente como nodeUser con: `sudo -u node -i` (-u proviene de user y -i de interactive)

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
* *TODO*



### Instalando y configurando MongoDB
* *TODO*
  
## **Ejercicio 2**

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