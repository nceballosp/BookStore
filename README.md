# ST0263 Tópicos Especiales en Telemática
## Estudiante(s): 
- Eduardo Piñeros Manjarres, eapinerosm@eafit.edu.co
- Natalia Ceballos Posada, nceballosp@eafit.edu.co
## Profesor:
- Edwin Montoya Munera, emontoya@eafit.edu.co

# Proyecto 2 – Aplicación Escalable
## 1. Descripción de la actividad
Se tiene ya una aplicación BookStore Monolítica, que simula un Sistema de Ecommerce de Venta de Libros.
BookStore actualmente corre en una sola máquina, con docker, un docker para la base de datos y otro docker para la aplicación. Se usa docker-compose para articular los 2 contenedores (mysql y python-flask).

Este proyecto tiene tres objetivos:
- **Objetivo 1:** Desplegar la aplicación BookStore Monolítica en dos (2) Máquinas Virtuales en AWS, con un dominio propio, certificado SSL y Proxy inverso en NGINX. (un servidor para la base de datos y otro servidor para la aplicación + nginx).
- **Objetivo 2:** Realizar el escalamiento en nube de la aplicación monolítica, siguiente algún patrón de arquitectura de escalamiento de apps monolíticas en AWS. La aplicación debe ser escalada utilizando Máquinas Virtuales (VM) con autoescalamiento, base de datos aparte Administrada o si es implementada con VM con Alta Disponibilidad, y Archivos compartidos vía NFS (como un servicio o una VM con NFS con Alta Disponibilidad), base de datos en RDS.
- **Objetivo 3:** Desplegar la aplicación BookStore monolítica en un clúster kubernetes (EKS o microk8s), evolucionándola desde un despliegue como aplicación monolítica
- **Objetivo 4:** Buscar y desplegar una aplicación libre y de código basada en microservicios, que sea desplegada en un clúster kubernetes, y que integre servicios como EFS, RDS, etc, y que tenga al menos un patrón de replicación de datos como CDRS o similar en al menos un microservicio.

## 2. Desarrollo de los objetivos

## **Objetivo 1**
## 1. Prerrequisitos
- Un dominio web (GoDaddy, Hostinger, Namecheap)
- 2 Maquinas EC2 en AWS (Ubuntu) [Tutorial](https://aws.amazon.com/es/getting-started/hands-on/deploy-wordpress-with-amazon-rds/3)
- 2 Direcciones IP elásticas AWS
## 2. Pasos
### Creación de los grupos de seguridad
Lo primero que haremos es crear los grupos de seguridad para nuestras 2 maquinas, uno para el servidor web y otro para la base de datos
<img width="1300" alt="image" src="https://github.com/user-attachments/assets/7360951c-438d-4a48-a051-fc30b12db207" />

-----------------

Se añaden las primeras reglas para permitir el trafico web y conexiones SSH desde cualquier ip, la regla para el trafico MySQL lo definiremos luego
<img width="1791" alt="image" src="https://github.com/user-attachments/assets/76308d76-cd43-4008-ab61-8f4634a92086" />

----------------

En este paso es importante que la regla de MYSQL/Aurora apunte al grupo de seguridad que creamos para el servidor web, asi podemos evitar usar ips quemadas lo que nos da flexibilidad al desplegar.

Luego editamos las reglas del primer grupo de seguridad para hacer lo mismo pero en la dirección contraria

<img width="1557" alt="image" src="https://github.com/user-attachments/assets/2d90fbc3-bc3e-4384-af79-1d0031b839ec" />

----------------------

Asi nuestras 2 máquinas van a poder comunicarse sin problema.

### Creación y configuración de las máquinas EC2
El próximo paso es instanciar 2 máquinas ec2 y asignarle a cada una su grupo de seguridad y su dirección elástica.
[Tutorial](https://aws.amazon.com/es/getting-started/hands-on/deploy-wordpress-with-amazon-rds/3)

El resultado se debería ver algo asi
<img width="1400" alt="image" src="https://github.com/user-attachments/assets/a638e9a1-76a8-482a-b065-2bc00afc224f" />

-------------------

Nos conectamos a la instancia de la Base de datos, y ejecutamos los siguientes comandos para crear la base de datos y el usuario que usaremos para conectarnos

*Instalamos mysql server*

```sudo apt update -y```

```sudo apt install mysql-server```

```sudo mysql_secure_installation```

Luego de configurar la base de datos ejecutamos las siguientes queries dentro de la base de datos

```sudo mysql```

```CREATE DATABASE bookstore;```

```CREATE USER 'bookstore_user'@'172.31.18.18' IDENTIFIED BY 'password';```

(Cambiar "bookstore" por el nombre de la base de datos, "172.31.18.18" por la ip privada de la instancia del servidor web, y "password" por la clave que se quiera poner)

<img width="772" alt="image" src="https://github.com/user-attachments/assets/2ac79f77-2a84-4535-ad98-41bb07ae8272" />

--------------------------------

esto es necesario para crear el schema que vamos a utilizar asi como el usuario y su contraseña, estos últimos 2 pueden ser cambiados a su gusto.

Y asignamos los permisos del usuario a la base de datos usando 

```
GRANT ALL PRIVILEGES ON bookstore.* TO 'bookstore_user'@'172.31.18.18';
FLUSH PRIVILEGES;
```
```exit```

<img width="718" alt="image" src="https://github.com/user-attachments/assets/4532e2a2-4238-4a98-82f8-345a13e86a03" />

-----------------------------

Y modificamos la configuracion de mysql para que permita las conexiones a traves de la ip privada de nuestra otra maquina.

```sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf```

<img width="920" alt="image" src="https://github.com/user-attachments/assets/27ccbb78-29e7-4084-a41a-4fd0af4501da" />

---------------------------------

```Ctrl + x```

```sudo systemctl restart mysql```

Luego nos conectamos a la instancia del servidor, en la cual tenemos que instalar docker y nginx, clonar el repositorio y configurar el archivo .env con las variables de la base de datos:

**Comandos**

```sudo apt update -y```

```sudo apt install docker.io -y```

```git clone https://github.com/nceballosp/BookStore```

```cd Bookstore```

```sudo nano .env```

<img width="267" alt="image" src="https://github.com/user-attachments/assets/7166fabb-0e0e-4d9c-812d-a3cf2bbfa2f9" />

-----------------------------------------

para DB_HOST es importante utilizar la ip privada asignada a la base de datos

Luego de esto ya podemos construir la imagen y correrla en un contenedor.

Ejecutamos:

```sudo docker image build . -t bookstore```

```sudo docker container run -p 5000:5000 -d --name bookstore_c bookstore```

<img width="1739" alt="image" src="https://github.com/user-attachments/assets/af34a943-9a65-4e27-b711-d3cea950a9d4" />

--------------------------------------

Ya con esto tenemos la aplicación corriendo en un contenedor local solo falta redirigir el trafico de nginx a nuestra aplicación


**Instalar y configurar nginx**

```sudo apt install nginx -y```

Creamos el archivo de configuracion para nuestro dominio (bookstoretopicos.shop)

```sudo nano /etc/nginx/sites-available/bookstore```

<img width="888" alt="image" src="https://github.com/user-attachments/assets/b060f9d6-c419-4c36-aa01-7eaee3e38ba8" />

-------------------------------

Guardamos el archivo y ejecutamos los siguientes comandos para aplicar los cambios correctamente

```
sudo ln -s /etc/nginx/sites-available/bookstore /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

Ahora si nos dirigimos a https://bookstoretopicos.shop deberia funcionar la tienda

<img width="1871" alt="image" src="https://github.com/user-attachments/assets/3e514237-14cc-4b0d-b783-cf9f4ac044b9" />

-------------------------------

**Certificado SSL**

para hacer esto usaremos certbot

```sudo apt install certbot python3-certbot-nginx -y```

contestamos las preguntas y luego ejecutamos el siguiente comando con nuestros dominios

```sudo certbot --nginx -d bookstoretopicos.shop -d www.bookstoretopicos.shop```

**Objetivo cumplido**

------------------------------------

## **Objetivo 2**
Para el objetivo 2 usaremos modificaremos una copia de Bookstore-1, configurada para conectarse con otra base de datos del servicio RDS, esta imagen servira como plantilla para nuestro grupo de auto escalado.

## Pasos



## **Objetivo 2**


## 2. información general de diseño de alto nivel, arquitectura, patrones, mejores prácticas utilizadas.

## 3. Descripción del ambiente de desarrollo y técnico: lenguaje de programación, librerías, paquetes, etc, con sus números de versiones.

<!-- como se compila y ejecuta.
detalles del desarrollo.
detalles técnicos
descripción y como se configura los parámetros del proyecto (ej: ip, puertos, conexión a bases de datos, variables de ambiente, parámetros, etc)
opcional - detalles de la organización del código por carpetas o descripción de algún archivo. (ESTRUCTURA DE DIRECTORIOS Y ARCHIVOS IMPORTANTE DEL PROYECTO, comando 'tree' de Linux)
opcionalmente - si quiere mostrar resultados o pantallazos  -->

<!-- 4. Descripción del ambiente de EJECUCIÓN (en producción) lenguaje de programación, librerías, paquetes, etc, con sus números de versiones.

 IP o nombres de dominio en nube o en la máquina servidor.

 descripción y como se configura los parámetros del proyecto (ej: ip, puertos, conexión a bases de datos, variables de ambiente, parámetros, etc)

como se lanza el servidor.

una mini guía de como un usuario utilizaría el software o la aplicación

opcionalmente - si quiere mostrar resultados o pantallazos  -->

# 5. otra información que considere relevante para esta actividad.

# Referencias:


