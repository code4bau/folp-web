Ejecución en 3 pasos:
Crear un entorno virtual
python3 -m venv .venv
source .venv/bin/activate
Instalar las dependencias
pip install -r requirements.txt
Correr el servidor de desarrollo
./manage.py runserver
Nota: Si tiene problemas con las dependencias, actualice pip usando pip install -U pip.

He realizado los siguientes cambios en tu configuración para preparar la aplicación para producción:

1. Cambios en app/settings.py

- Modo DEBUG: La configuración de DEBUG ahora se controla mediante una variable de entorno.
  - Para desarrollo (local), no necesitas hacer nada. DEBUG será True por defecto.
  - Para producción, debes establecer la variable de entorno DJANGO_DEBUG a False.
- ALLOWED_HOSTS: He añadido '\*' a la lista para permitir que el servidor responda a cualquier host. Esto simplifica la configuración para Nginx.
- CSRF_TRUSTED_ORIGINS: He añadido https://163.10.29.39 para permitir peticiones POST desde tu dominio de producción.

2. Gunicorn

gunicorn ya está en tu fichero requirements.txt, así que no es necesario añadirlo.

3. Ficheros Estáticos

Antes de lanzar la aplicación en producción, Django necesita recolectar todos los ficheros estáticos en un solo directorio.

Ejecuta este comando en tu servidor:

1 python manage.py collectstatic

Esto copiará todos los ficheros estáticos a la carpeta staticfiles/ que has definido en tu settings.py.

4. Ejecutar con Gunicorn

En tu servidor de producción, en lugar de runserver, usarás Gunicorn para ejecutar la aplicación. Gunicorn se comunicará con Nginx a través de un socket de Unix.

Usa el siguiente comando para iniciar Gunicorn:

1 gunicorn --workers 3 --bind unix:/run/gunicorn.sock app.wsgi:application

- --workers 3: Un buen punto de partida es usar 2-4 workers por núcleo de CPU. Ajústalo según sea necesario.
- --bind unix:/run/gunicorn.sock: Crea un socket de Unix en /run/gunicorn.sock para que Nginx se comunique con Gunicorn.

5. Configuración de Nginx

He creado un fichero de configuración de ejemplo para Nginx llamado nginx.conf.example en la raíz de tu proyecto. Deberás copiar o enlazar este fichero a la configuración de Nginx en tu
servidor (normalmente en /etc/nginx/sites-available/ y luego crear un enlace simbólico en /etc/nginx/sites-enabled/).

El contenido del fichero es el siguiente:

    1 server {
    2     listen 80;
    3     server_name 163.10.29.39;
    4
    5     location = /favicon.ico { access_log off; log_not_found off; }
    6     location /static/ {
    7         root /home/ctrbts/dev/github.com/code4bau/folp-web;
    8     }
    9

10 location /media/ {
11 root /home/ctrbts/dev/github.com/code4bau/folp-web;
12 }
13
14 location / {
15 proxy_pass http://unix:/run/gunicorn.sock;
16 proxy_set_header Host $host;
17 proxy_set_header X-Real-IP $remote_addr;
18 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
19 proxy_set_header X-Forwarded-Proto $scheme;
20 }
21 }

Resumen de los pasos para el despliegue:

1.  Sube tu código al servidor.
2.  Instala las dependencias con pip install -r requirements.txt.
3.  Establece la variable de entorno DJANGO_DEBUG=False.
4.  Ejecuta python manage.py collectstatic.
5.  Inicia Gunicorn con el comando proporcionado.
6.  Configura Nginx usando el fichero de ejemplo.
7.  Reinicia Nginx.

Con esto, tu aplicación debería estar lista para funcionar en el entorno de producción.
