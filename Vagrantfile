# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Define la caja base
  config.vm.box = "debian/bullseye64"

  # Configura el reenvío de puertos
  config.vm.network "forwarded_port", guest: 8080, host: 8080

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
    sudo chmod -R 775 /var/www/app
    cp /var/www/app/.env /vagrant/

    #Iniciamos ahora nuestro entorno virtual. Pipenv cargará las variables de entorno desde el fichero .env de forma automática:
    pipenv shell

    #Usamos pipenv install flask gunicorn para instalar las dependencias necesarias para nuestro proyecto:
    pipenv install flask gunicorn

    #Creamos Una apliacion Flask:
    touch application.py wsgi.py
    sudo cp /var/www/app/application.py /vagrant

    #Corremos la App con Flask
    flask run --host '0.0.0.0'




  SCRIPT
end