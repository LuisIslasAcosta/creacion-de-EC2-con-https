--
# Configuración de Aplicación Flask en Amazon Linux con Nginx y Gunicorn

Este documento describe los pasos necesarios para configurar una aplicación Flask en una instancia de Amazon Linux utilizando Nginx como servidor web y Gunicorn como servidor WSGI (Web Server Gateway Interface).

## Prerrequisitos
- descarga mobaxTerm
- Una instancia de Amazon EC2 corriendo Amazon Linux.
- Acceso SSH a la instancia.
- Permisos para modificar las configuraciones del sistema.

## Instalación y Configuración

### 1. Permitir tráfico SSH y HTTP

Asegúrate de que tu grupo de seguridad permite el tráfico SSH y HTTP desde Internet.

### 2. Instalar Nginx

Ejecuta los siguientes comandos para instalar Nginx:

bash
sudo dnf install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx


### 3. Crear el directorio de la aplicación Flask
Crea el directorio para tu aplicación y establece los permisos adecuados:
bash
sudo mkdir -p /opt/flask_app
sudo chown ec2-user:ec2-user /opt/flask_app
cd /opt/flask_app


### 4. Instalar Python y PIP
Instala Python 3 y pip:
bash
sudo dnf install python3-pip -y


### 5. Instalar Flask y dependencias
Instala Flask y otras dependencias necesarias:
bash
sudo pip3 install Flask gunicorn flask-restful flask-swagger-ui


### 6. Crear el archivo app.py
Crea el archivo app.py con el siguiente contenido:
bash
vi app.py


bash
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return "Hello, Flask on Amazon Linux with Nginx and Gunicorn!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)


### 7. Ejecutar la aplicación Flask

Para probar la aplicación, ejecuta:

bash
python3 app.py



### 8. Configurar Gunicorn como servicio
Determina la ruta de Gunicorn:
bash
which gunicorn

Luego, crea el archivo de servicio para Gunicorn:

bash
sudo vi /etc/systemd/system/flask_app.service


Agrega el siguiente contenido:
bash
[Unit]
Description=Gunicorn instance to serve Flask app
After=network.target

[Service]
User=ec2-user
Group=ec2-user
WorkingDirectory=/opt/flask_app
ExecStart=/usr/local/bin/gunicorn --workers 3 --bind 0.0.0.0:8000 app:app

[Install]
WantedBy=multi-user.target


### 9. Recargar el daemon de systemd y empezar el servicio
Ejecuta los siguientes comandos:
bash
sudo systemctl daemon-reload
sudo systemctl start flask_app
sudo systemctl status flask_app


### 10. Generación de Certificado SSL Autofirmado
Paso 1: Instalar OpenSSL (si no está instalado)
Si OpenSSL no está instalado en tu instancia, puedes instalarlo con el siguiente comando:

bash

sudo dnf install openssl -y


Paso 2: Crear el directorio para los certificados SSL
Crea un directorio donde almacenarás el certificado SSL y la clave privada:

bash
sudo mkdir -p /etc/nginx/ssl


Paso 3: Generar el certificado y la clave privada
Genera un certificado SSL autofirmado utilizando OpenSSL. Este comando te pedirá que ingreses algunos detalles como el nombre de la empresa, país, etc.

bash

sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt


Al ejecutar este comando, se te pedirá que completes los detalles del certificado, como la "Organización", "Nombre Común (Common Name)", etc. Puedes dejar algunos campos en blanco, pero asegúrate de completar el Nombre Común con la dirección IP de tu servidor o el nombre de dominio si tienes uno.

Paso 4: Verificar los archivos generados
Después de ejecutar el comando anterior, deberías tener los siguientes archivos:

nginx.crt: El certificado SSL autofirmado.

nginx.key: La clave privada asociada al certificado.

Puedes verificar que ambos archivos se hayan creado correctamente:

bash

ls -l /etc/nginx/ssl/




### 11. Configurar Nginx para usar SSL
Ahora que tienes el certificado SSL autofirmado, debes configurar Nginx para usar HTTPS.

Paso 1: Editar la configuración de Nginx
Abre el archivo de configuración de Nginx:

bash

sudo vi /etc/nginx/nginx.conf



Paso 2: Modificar la configuración de Nginx para HTTPS
En este caso como anteriormente se agrego el protocolo http solamente, se tendra que remplazar por el siguiente codigo asi que se tiene que eliminar el anterior codigo del puerto 80 para evitar errores
Agrega una nueva sección server para manejar las solicitudes HTTPS. Aquí tienes un ejemplo de cómo debería verse la configuración:

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

bash

        server {
    listen 443 ssl;
    server_name 3.148.205.107; # Reemplaza con tu dirección IPv4 public

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}


    server {
    listen 80;
    server_name 3.148.205.107;# Reemplaza con tu dirección IPv4 public

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}




------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Paso 3: Verificar la configuración de Nginx
Antes de reiniciar Nginx, verifica que la configuración sea válida:

bash

sudo nginx -t

Si no hay errores de sintaxis, deberías ver un mensaje como nginx: configuration file /etc/nginx/nginx.conf test is successful.

Paso 4: Reiniciar Nginx
Reinicia Nginx para aplicar los cambios:

bash
sudo systemctl restart nginx



### 12. Instalacon de la base de datos
sudo yum install -y mariadb105

### (Estos pasos pupueden ser omitidos en caso de que se haya creado la RDS y se configurara apuntando a esta instancia)

NOTA: Si no te permite el mysql debes dd edirigirte a security group y en reglas agregar una nueva regla con lo siguiente
Tipo: MySQL/Aurora
Protocolo: TCP
Puerto: 3306
ID: (Se agrega el Private IPv4 addresses por ejemplo) 172.31.6.227

### 13. Instalar telnet
sudo yum install -y telnet

### 14. Comprobar conexion  de base de datos a instancia
telnet database-1.cbs4y06s6sd3.us-east-2.rds.amazonaws.com 3306


### 15 Error en la conexion
En caso de aparexer un mensaje como el siguiente
bash
Trying 172.31.48.109...
telnet: connect to address 172.31.48.109: Connection timed out

### (Estos pasos pupueden ser omitidos en caso de que se haya creado la RDS y se configurara apuntando a esta instancia)

1.- Los pasos adicionales son dirigirte a RDS 
2.- Ir a Databases
3.- Seleccionas la base de datos (database-1)
4.- Dirigirse a la pestaña de Connectivity & security 
5.- En la parte de abajo hay una seccion llamada Connected compute resources 
6.- Verifica que en esa seccion se encuentre la instacia de EC2 correcta y no otra


### 15.1 Vuelve a intentar la conexion
Vuelve a ingresar el codigo 
bash
telnet database-1.cbs4y06s6sd3.us-east-2.rds.amazonaws.com 3306

Cuando te aparezca un mensaje como el siguiente

[ec2-user@ip-172-31-6-227 flask_app]$ telnet database-1.cbs4y06s6sd3.us-east-2.rds.amazonaws.com 3306
Trying 172.31.48.109...
Connected to database-1.cbs4y06s6sd3.us-east-2.rds.amazonaws.com.
Escape character is '^]'.
J
8.0.40?(vb|Ra

Es señal de que la conexion de tu instancia a la base de datos se ejecuto de manera correcta

### 16. Ejecuta el comando mysql
Una vez se ejecute de manera correcta todo lo anterior, se debe ingresar el comando mysql para comprobar que
se tenga conexion entre la EC2 y RDS on el siguiente codigo
bash
mysql -u admin -p -h database-1.cbs4y06s6sd3.us-east-2.rds.amazonaws.com

e ingresa la contraseña que agregaste a tu base de datos si la conexion fue exitosa deberia aparecerte algo como lo siguiente

bash
mysql -u admin -p -h database-1.cbs4y06s6sd3.us-east-2.rds.amazonaws.com
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 1349
Server version: 8.0.40 Source distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

### 17. implementacion de ina api de fllask desde git

Para esto es necesarip instalar git en tu EC2 por lo cual se agregara el siguiente codigo
bash
sudo dnf install git

Despues de agregar este codigo tienes que aceptar la   descarga con "yes"

Se terminara de instalar, posteriormente se debe eliminar todo lo que este dentro de la carpeta de /opt/flask_app que se localiza en tu EC2 debido a que todo lo que necesitas en tu API ya se debe de encontrar en el Api que se migrara
Para ubicar los archivos que se encuentren en tu EC2 se ejecuta el comando 
bash
ls

Esto enlistara todos los archivos ubicadosd en tu carpeta y esta carpeta debe estar completamente vacia

Terminando se debe ejecutar el comando para migrar de GitHub a la EC2 con el comando:

bash
git clone (Link de tu ssh de tu repositorio) . (Es necesario agregar el un espacio y el punto despues del link debido a que sino se pone se descargara toda la carpeta localizada en tu GitHub y al agregar el punto se da la orden de descargar solo el contenido del respositorio directamente)
