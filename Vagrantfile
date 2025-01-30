# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Define la caja base
  config.vm.box = "debian/bullseye64"

  # Configura el reenvío de puertos
  config.vm.network "forwarded_port", guest: 8080, host: 8080

  # Configura una IP fija en la red privada
  config.vm.network "private_network", ip: "192.168.56.10"

  # Configuración de la máquina virtual
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"  # Asigna memoria a la VM
  end

  # Provisión de la VM con un script de shell
  config.vm.provision "shell", privileged: false, inline: <<-SCRIPT
    
    # Instalamos el gestor de paquetes de Python pip y el paquete pipenv para gestionar los entornos virtuales:
    sudo apt-get update && sudo apt-get install -y python3-pip
    pip3 install pipenv


    # Después instalaremos el paquete python-dotenv para cargar las variables de entorno.
    pip3 install python-dotenv

    #Creamos el directorio en el que almacenaremos nuestro proyecto:
    sudo mkdir -p /var/www/app
    sudo chown -R $USER:www-data /var/www/app
    sudo chmod -R 775 /var/www/app
    sudo cp -v /var/www/app/.env /vagrant/

    #Iniciamos ahora nuestro entorno virtual. Pipenv cargará las variables de entorno desde el fichero .env de forma automática:
    cd /var/www/app
    pipenv install --deploy --system

    #Creamos Una apliacion Flask:
    touch application.py wsgi.py
    sudo cp -v /var/www/app/application.py /vagrant

    #Corremos la App con Flask en el puerto 8080 que hemos especificado al crear la máquina virtual en http://127.0.0.1:8080.
    flask run --host 0.0.0.0 --port 8080
    sudo cp -v /var/www/app/wsgi.py /vagrant

    #Creamos el archivo wsgi.py para que nos deje interactuar con él y corremos en flash y luego mediante https://localhost:8080 podemos ver la aplicación desplegada.
    sudo cp -v /var/www/app/wsgi.py
    gunicorn --workers 4 --bind 0.0.0.0:8080 wsgi:app

    #NGINX

    #Actualizamos y arreglamos los servicios ya que sino nos dará un error.
    sudo apt update && sudo apt upgrade -y
    sudo apt --fix-broken install -y

    #Ahora si instalamos NGINX
    sudo apt install nginx -y
    sudo systemctl enable nginx
    sudo systemctl start nginx
    sudo cp -v /etc/systemd/system/flask_app.service /vagrant


    # Recargamos systemd y habilitamos el servicio
    sudo systemctl daemon-reload
    sudo systemctl enable flask_app
    sudo systemctl start flask_app

    # Copiamos el archivo de configuración de NGINX
    sudo cp -v /etc/nginx/sites-available/app.conf /vagrant

    #Ahora debemos crear un link simbólico del archivo de sitios webs disponibles al de sitios web activos:
    sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/

    #Añadimos la dirección IP a /etc/hosts para desplegar la aplicación y comprobamos que va todo bien con el servidor NGINX y lo reiniciamos.
    echo "192.168.56.10 app.izv www.app.izv" | sudo tee -a /etc/hosts
    sudo nginx -t
    sudo systemctl restart nginx
    sudo systemctl status nginx
    sudo cp -v /etc/hosts



  SCRIPT
end