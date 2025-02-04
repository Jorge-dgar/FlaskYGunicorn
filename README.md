---

# README

Este repositorio contiene la configuración completa para desplegar una aplicación web Flask utilizando **Vagrant**, **Gunicorn**, **NGINX** y **Flask**. El entorno está configurado para facilitar la creación de una máquina virtual que corra de forma eficiente y permita gestionar las dependencias, la aplicación y el servidor web.

## Requisitos

Para poder trabajar con este proyecto, necesitas tener instalados los siguientes programas en tu máquina local:

- [Vagrant](https://www.vagrantup.com/downloads)
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

## Estructura de Archivos

A continuación se detalla la estructura de los archivos en el proyecto:

```
/
├── Vagrantfile                # Configuración para la máquina virtual
├── .env                       # Variables de entorno para la aplicación Flask
├── app.conf                   # Configuración de NGINX
├── application.py             # Código fuente de la aplicación Flask
├── flask_app.service          # Servicio systemd para Flask con Gunicorn
├── hosts                      # Configuración de IPs para acceder a la aplicación
├── wsgi.py                    # Punto de entrada para Gunicorn
└── imágenes                   # Directorio para almacenar imágenes adicionales
```

## Descripción de los Archivos

### `Vagrantfile`

El archivo `Vagrantfile` contiene la configuración necesaria para crear y configurar la máquina virtual. Se utiliza **Vagrant** y **VirtualBox** para configurar el entorno de desarrollo y garantizar que la aplicación se ejecute de manera consistente en diferentes máquinas.

#### Explicación del `Vagrantfile`

```ruby
Vagrant.configure("2") do |config|
  # Define la caja base, en este caso, Debian
  config.vm.box = "debian/bullseye64"

  # Configura el reenvío de puertos (puerto 8080 de la VM a 8080 del host)
  config.vm.network "forwarded_port", guest: 8080, host: 8080

  # Configuración de red privada con IP fija
  config.vm.network "private_network", ip: "192.168.56.10"

  # Configuración de la máquina virtual (asignación de memoria)
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"  # Asignación de memoria RAM a la VM
  end

  # Script de provisión para instalar dependencias, configurar la aplicación y el servidor
  config.vm.provision "shell", privileged: false, inline: <<-SCRIPT
    # Actualizar e instalar dependencias
    sudo apt-get update && sudo apt-get install -y python3-pip
    pip3 install pipenv python-dotenv

    # Crear el directorio para la aplicación y configurar permisos
    sudo mkdir -p /var/www/app
    sudo chown -R $USER:www-data /var/www/app
    sudo chmod -R 775 /var/www/app

    # Instalar dependencias del entorno virtual
    cd /var/www/app
    pipenv install --deploy --system

    # Crear la aplicación Flask
    touch application.py wsgi.py

    # Ejecutar la aplicación Flask en el puerto 8080
    flask run --host 0.0.0.0 --port 8080

    # Iniciar Gunicorn para manejar las solicitudes HTTP
    gunicorn --workers 4 --bind 0.0.0.0:8080 wsgi:app

    # Configuración de NGINX
    sudo apt install nginx -y
    sudo systemctl enable nginx
    sudo systemctl start nginx

    # Copiar configuración de NGINX
    sudo cp -v /etc/nginx/sites-available/app.conf /vagrant

    # Crear enlace simbólico para habilitar el sitio
    sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/

    # Añadir dirección IP a /etc/hosts y reiniciar NGINX
    echo "192.168.56.10 app.izv www.app.izv" | sudo tee -a /etc/hosts
    sudo nginx -t
    sudo systemctl restart nginx
  SCRIPT
end
```

Este `Vagrantfile` realiza lo siguiente:

1. **Red de la VM**: Configura una red privada con la IP `192.168.56.10`.
2. **Instalación de dependencias**: Se asegura de que se instalen **pipenv**, **python-dotenv**, y otras dependencias necesarias para la aplicación.
3. **Creación de la aplicación Flask**: Crea los archivos necesarios para la aplicación Flask y la ejecuta con **Gunicorn**.
4. **Configuración de NGINX**: Instala y configura el servidor web **NGINX** como proxy inverso para la aplicación.

### `.env`

El archivo `.env` contiene las variables de entorno necesarias para que Flask funcione correctamente:

```bash
FLASK_APP=wsgi.py
FLASK_ENV=production
```

### `app.conf`

Este archivo contiene la configuración de **NGINX** para hacer de proxy inverso, redirigiendo las solicitudes al socket de **Gunicorn**.

```nginx
server {
  listen 80;
  server_name app.izv www.app.izv;

  access_log /var/log/nginx/app.access.log;
  error_log /var/log/nginx/app.error.log;

  location / {
    include proxy_params;
    proxy_pass http://unix:/var/www/app/app.sock;
  }
}
```

### `application.py`

Este archivo contiene el código fuente de la aplicación Flask, que maneja las rutas y la respuesta a las solicitudes.

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    '''Página principal de la aplicación'''
    return '<h1>App desplegada</h1>'
```

### `flask_app.service`

El archivo `flask_app.service` configura un servicio **systemd** para ejecutar la aplicación con **Gunicorn** en segundo plano.

```ini
[Unit]
Description=flask app service - App con Flask y Gunicorn
After=network.target

[Service]
User=vagrant
Group=www-data
Environment="PATH=/home/vagrant/.local/share/virtualenvs/app-1lvW3LzD/bin"
WorkingDirectory=/var/www/app
ExecStart=/home/vagrant/.local/share/virtualenvs/app-1lvW3LzD/bin/gunicorn --workers 3 \
  --bind unix:/var/www/app/app.sock wsgi:app

[Install]
WantedBy=multi-user.target
```

### `hosts`

Este archivo configura la dirección IP de la máquina virtual para poder acceder a la aplicación a través del nombre de dominio `app.izv`.

```bash
127.0.0.1       localhost
127.0.0.2       bullseye
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
192.168.56.10 app.izv www.app.izv
```

### `wsgi.py`

Este archivo es el punto de entrada para **Gunicorn**, que ejecutará la aplicación Flask.

```python
from application import app

if __name__ == "__main__":
    app.run()
```

## Instalación y Uso

### 1. Inicia la máquina virtual con Vagrant

Para crear y configurar la máquina virtual, corre el siguiente comando en tu terminal:

```bash
vagrant up
```

### 2. Accede a la máquina virtual

Cuando la máquina virtual esté en funcionamiento, accede a ella con:

```bash
vagrant ssh
```

### 3. Verifica el estado de la aplicación

Una vez que todo esté corriendo, puedes acceder a la aplicación a través del navegador en la dirección `http://app.izv`.

### Sección de Imágenes

1. Añadir la dirección IP para acceder a la APP en /etc/hosts
En este paso, añadimos la dirección IP 192.168.56.10 en el archivo /etc/hosts para que podamos acceder a la aplicación a través del nombre de dominio app.izv o www.app.izv.

![añadimos la dirección IP para acceder a la APP en etc hosts](https://github.com/user-attachments/assets/97a00134-1571-400d-9caa-927781c65e38)


3. Aplicación desplegada en Internet
Una vez configurado el servidor, la aplicación Flask estará disponible en el navegador a través del dominio configurado en el archivo /etc/hosts.

![aplicación desplegada](https://github.com/user-attachments/assets/75065287-d84b-4fc0-82a2-a1152a0ef08b)
![app desplegada](https://github.com/user-attachments/assets/0347cfe9-8d09-4287-abb4-53f3344525a2)


5. Comprobamos que el servidor de NGINX funciona
Comprobamos que el servidor NGINX esté funcionando correctamente. Esto se puede hacer revisando el estado de NGINX y verificando que no haya errores en su configuración.

![instalamos y activamos el servidor NGINX](https://github.com/user-attachments/assets/bd9d2043-8909-427e-a14c-ea23c1c967ea)


7. Despliegue en Flask
Aquí verificamos que la aplicación Flask esté corriendo y accesible. En esta imagen se muestra cómo se puede acceder a la página principal de la aplicación Flask, por ejemplo, en http://app.izv.

![despliegue en flask](https://github.com/user-attachments/assets/e99e3e2c-f234-4a19-a22c-f680994a0329)

8. Despliegue en Gunicorn
Gunicorn es el servidor que ejecuta la aplicación Flask. En este paso verificamos que Gunicorn esté manejando correctamente las solicitudes HTTP.

![despliegue gunicorn](https://github.com/user-attachments/assets/f8e581b9-ab2b-4cdd-9b08-d29e29566a9d)

9. Iniciamos entorno virtual con Flask
Aquí configuramos y activamos un entorno virtual para asegurarnos de que todas las dependencias de Python estén aisladas y gestionadas correctamente.
![Iniciamos entorno virtual con FLASK](https://github.com/user-attachments/assets/899f8513-2299-4b47-94da-4e6e2c3eae69)


10. Instalamos Flask y Gunicorn
En este paso, instalamos las dependencias necesarias para el proyecto, como Flask y Gunicorn, que se utilizan para ejecutar la aplicación y servirla a través de HTTP.

![Instalamos flask y Gunicorn](https://github.com/user-attachments/assets/e1e519b2-0652-46fa-bffc-2432f27c3962)

11. Instalamos y activamos el servidor NGINX
Se instala y activa el servidor NGINX. Este servidor web se configura para actuar como un proxy inverso y redirigir las solicitudes a la aplicación Flask ejecutada por Gunicorn.

![instalamos y activamos el servidor NGINX](https://github.com/user-attachments/assets/66c29c99-57db-4aef-ab62-44b0c00ac283)

12. Una vez añadimos los permisos, nos metemos en la aplicación
Finalmente, después de configurar correctamente los permisos y asegurarnos de que todos los servicios estén corriendo, podemos acceder a la aplicación Flask en el navegador.

![Una vez añadimos los permisos nos metemos en la aplicación](https://github.com/user-attachments/assets/5edf712c-95b0-409a-911c-1cdb58c30065)

---
