La guía se divide en dos partes principales:

1.  Infraestructura (Proxmox y LXC): Creación y configuración del servidor.
2.  Aplicación (Django): Modificaciones en tu código y despliegue.

---

Parte 1: Creación y Configuración del Servidor LXC

Paso 1: Preparar Plantilla de LXC en Proxmox

Primero, asegúrate de tener la plantilla de Ubuntu 23.04.

1.  En la interfaz web de Proxmox, ve a Datacenter -> pve -> local (o tu almacenamiento de plantillas).
2.  Selecciona la pestaña CT Templates.
3.  Haz clic en Templates.
4.  Busca ubuntu-23.04-standard y haz clic en Download.

Paso 2: Crear el Contenedor LXC

1.  Haz clic en el botón Create CT en la esquina superior derecha de la interfaz de Proxmox.
2.  Pestaña `General`:
    - Hostname: folp-server (o el nombre que prefieras).
    - Password: Establece una contraseña segura para el usuario root.
3.  Pestaña `Template`:
    - Storage: local (o donde descargaste la plantilla).
    - Template: Selecciona la plantilla ubuntu-23.04-standard.
4.  Pestaña `Disks`:
    - Disk size (GiB): 20 GB es un buen punto de partida. Puedes ampliarlo más tarde si es necesario.
5.  Pestaña `CPU`:
    - Cores: 2 cores es suficiente para empezar.
6.  Pestaña `Memory`:
    - Memory (MiB): 2048 MB (2 GB).
7.  Pestaña `Network`:
    - Déjalo en DHCP por ahora para que obtenga una IP automáticamente. Más tarde puedes configurarla como estática si lo deseas.
8.  Pestaña `DNS`:
    - Puedes dejar los valores por defecto o usar los DNS de tu preferencia (ej. 8.8.8.8).
9.  Confirmar y Finalizar: Revisa el resumen y haz clic en Finish.

Una vez creado, selecciona el contenedor en el menú de la izquierda y haz clic en Start. Luego, abre la Console.

Paso 3: Configuración Inicial del Servidor (Dentro del LXC)

Ahora, todos los siguientes comandos se ejecutan dentro de la consola del LXC que acabas de crear.

1.  Actualizar el sistema:
    1 apt-get update && apt-get upgrade -y

2.  Instalar paquetes necesarios:
    - postgresql: La base de datos.
    - nginx: El servidor web / proxy inverso.
    - python3-pip, python3-venv: Para gestionar tu aplicación Python.
    - git: Para clonar tu repositorio.

1 apt-get install -y postgresql postgresql-contrib nginx python3-pip python3-venv git

Paso 4: Configurar la Base de Datos PostgreSQL

1.  Acceder a PostgreSQL:
    1 sudo -u postgres psql

2.  Crear la base de datos: (Reemplaza folp_db si quieres otro nombre)
    1 CREATE DATABASE folp_db;

3.  Crear un usuario y contraseña: (¡Usa una contraseña segura!)

1 CREATE USER folp_user WITH PASSWORD 'tu_contraseña_super_segura';

4.  Configurar los parámetros recomendados para Django:

1 ALTER ROLE folp_user SET client_encoding TO 'utf8';
2 ALTER ROLE folp_user SET default_transaction_isolation TO 'read committed';
3 ALTER ROLE folp_user SET timezone TO 'UTC';

5.  Darle permisos al usuario sobre la base de datos:
    1 GRANT ALL PRIVILEGES ON DATABASE folp_db TO folp_user;

6.  Salir de psql:
    1 \q

---

Parte 2: Modificaciones y Despliegue de la Aplicación Django

Paso 5: Preparar el Código de tu Aplicación

Voy a asumir que subirás tu código usando git.

1.  Clonar el repositorio en el servidor:
    Navega al directorio /opt (un buen lugar para aplicaciones) y clona tu proyecto.

1 cd /opt
2 git clone <URL_DE_TU_REPOSITORIO> folp-web
3 cd folp-web

2.  Crear y activar un entorno virtual:
    1 python3 -m venv venv
    2 source venv/bin/activate
    Verás (venv) al principio de tu línea de comandos.

3.  Instalar dependencias:
    Tu requirements.txt actual no incluye las librerías para PostgreSQL ni para el servidor de aplicaciones WSGI. Necesitamos añadirlas.

1 pip install -r requirements.txt
2 pip install gunicorn psycopg2-binary python-dotenv
_ gunicorn: Servidor de aplicaciones WSGI para correr Django en producción.
_ psycopg2-binary: Adaptador de Python para PostgreSQL. \* python-dotenv: Para gestionar variables de entorno (como la contraseña de la BD) de forma segura.

Paso 6: Modificaciones Críticas en `app/settings.py`

Estos cambios son esenciales para que tu aplicación funcione en un entorno de producción.

1.  Crear un archivo `.env` para las variables de entorno:
    En la raíz de tu proyecto (/opt/folp-web/), crea un archivo llamado .env:

1 SECRET_KEY=tu_nueva_secret_key_muy_larga_y_aleatoria
2 DEBUG=False
3 DATABASE_URL=postgres://folp_user:tu_contraseña_super_segura@localhost:5432/folp_db
4 ALLOWED_HOSTS=tu_dominio.com,www.tu_dominio.com,IP_DEL_SERVIDOR
_ SECRET_KEY: Genera una nueva. No uses la de desarrollo.
_ DEBUG: Debe ser `False` en producción.
_ DATABASE_URL: Usa los datos que configuraste en PostgreSQL.
_ ALLOWED_HOSTS: Lista de dominios/IPs que pueden servir tu aplicación.

2.  Modificar `settings.py` para leer el archivo `.env`:
    Al principio de app/settings.py, añade:

1 import os
2 from dotenv import load_dotenv
3 import dj_database_url
4
5 # Build paths inside the project like this: BASE_DIR / 'subdir'.
6 BASE_DIR = Path(**file**).resolve().parent.parent
7
8 load_dotenv(os.path.join(BASE_DIR, '.env'))

3.  Ajustar las variables de configuración:
    Busca estas variables en settings.py y modifícalas para que lean desde las variables de entorno:

        * SECRET_KEY:

    1 SECRET_KEY = os.environ.get('SECRET_KEY')

        * DEBUG:

    1 DEBUG = os.environ.get('DEBUG', 'False').lower() == 'true'

        * ALLOWED_HOSTS:

1 ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

       * DATABASES: Reemplaza la configuración de sqlite3 por esta:

1 DATABASES = {
2 'default': dj_database_url.config(conn_max_age=600, ssl_require=False)
3 }
(Necesitarás añadir import dj_database_url al principio del fichero).

       * STATIC_ROOT: Asegúrate de que esta línea esté presente y descomentada. Es donde Django copiará todos los archivos estáticos.

1 STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

Paso 7: Configurar Gunicorn con `systemd`

Para que tu aplicación se ejecute de forma robusta, crearemos un servicio de systemd.

1.  Crear el archivo de servicio:
    1 nano /etc/systemd/system/gunicorn.service

2.  Pegar la siguiente configuración:
    (Asegúrate de que las rutas WorkingDirectory y ExecStart coincidan con tu configuración).


    1     [Unit]
    2     Description=gunicorn daemon for folp-web
    3     After=network.target
    4
    5     [Service]
    6     User=root  # O un usuario no privilegiado si creas uno
    7     Group=www-data
    8     WorkingDirectory=/opt/folp-web
    9     ExecStart=/opt/folp-web/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/opt/folp-web/app.sock app.wsgi:application

10
11 [Install]
12 WantedBy=multi-user.target
_ --workers 3: Un buen punto de partida. La recomendación es 2 _ num_cores + 1. \* --bind unix:/opt/folp-web/app.sock: Gunicorn se comunicará con Nginx a través de un socket Unix, que es más eficiente que un puerto de red.

Paso 8: Configurar Nginx como Proxy Inverso

Nginx recibirá las peticiones del exterior y las redirigirá a Gunicorn. También servirá los archivos estáticos directamente.

1.  Crear el archivo de configuración de Nginx:
    1 nano /etc/nginx/sites-available/folp-web

2.  Pegar la siguiente configuración:
    (Reemplaza tu_dominio.com y las rutas si son diferentes).


    1     server {
    2         listen 80;
    3         server_name tu_dominio.com www.tu_dominio.com IP_DEL_SERVIDOR;
    4
    5         location = /favicon.ico { access_log off; log_not_found off; }
    6
    7         location /static/ {
    8             root /opt/folp-web;
    9         }

10
11 location /media/ {
12 root /opt/folp-web;
13 }
14
15 location / {
16 include proxy_params;
17 proxy_pass http://unix:/opt/folp-web/app.sock;
18 }
19 }

3.  Activar la configuración:

1 ln -s /etc/nginx/sites-available/folp-web /etc/nginx/sites-enabled/
2 # Elimina la configuración por defecto para evitar conflictos
3 rm /etc/nginx/sites-enabled/default

4.  Probar la configuración de Nginx:
    1 nginx -t
    Si dice syntax is ok y test is successful, todo va bien.

Paso 9: Pasos Finales y Lanzamiento

1.  Activa el entorno virtual (source /opt/folp-web/venv/bin/activate).
2.  Recopilar archivos estáticos:
    1 python manage.py collectstatic
    Escribe yes cuando te lo pida. Esto copiará todos los archivos de app/static a staticfiles.
3.  Aplicar las migraciones de la base de datos:
    1 python manage.py migrate
4.  Crear un superusuario (opcional, para el admin de Django):
    1 python manage.py createsuperuser
5.  Iniciar y habilitar los servicios:
    1 systemctl start gunicorn
    2 systemctl enable gunicorn
    3 systemctl restart nginx

¡Y listo! Ahora deberías poder acceder a tu aplicación a través de la IP de tu contenedor LXC o del dominio que hayas configurado.

Si algo falla, los primeros lugares para buscar errores son:

- systemctl status gunicorn
- journalctl -u gunicorn
- /var/log/nginx/error.log
