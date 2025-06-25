## Ejecución en 3 pasos: 

1. Crear un entorno virtual
   ```
   python3 -m venv .venv
   source .venv/bin/activate
   ```
2. Instalar las dependencias
   ```
   pip install -r requirements.txt
   ```
3. Correr el servidor de desarrollo
   ```
   ./manage.py runserver
   ```

Nota: Si tiene problemas con las dependencias, actualice `pip` usando `pip install -U pip`.
