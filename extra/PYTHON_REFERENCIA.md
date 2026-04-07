# 🐍 Python en GitHub Actions: Guía de Referencia Rápida

## 📋 Índice
1. [Python Básico](#python-básico)
2. [Variables y Output](#variables-y-output)
3. [Archivos y Directorios](#archivos-y-directorios)
4. [JSON y YAML](#json-y-yaml)
5. [APIs y HTTP](#apis-y-http)
6. [Procesamiento de Datos](#procesamiento-de-datos)
7. [Utilidades Comunes](#utilidades-comunes)

---

## 🐍 Python Básico

### Sintaxis Básica de Python en Workflows

```yaml
- name: Script Python inline
  shell: python
  run: |
    # Esto es un comentario
    print("¡Hola desde Python!")
    
    # Variables
    nombre = "GitHub Actions"
    version = 1.0
    
    # Imprimir con formato
    print(f"Usando {nombre} versión {version}")
```

### Variables en Python

```python
# Números
edad = 25
precio = 19.99
grande = 1000000

# Strings (texto)
nombre = "Juan"
apellido = 'Pérez'
completo = f"{nombre} {apellido}"  # f-string (formateo)

# Booleanos (True/False)
activo = True
completado = False

# Listas (arrays)
numeros = [1, 2, 3, 4, 5]
nombres = ["Ana", "Bob", "Carlos"]

# Diccionarios (objetos clave-valor)
persona = {
    "nombre": "Ana",
    "edad": 30,
    "ciudad": "Madrid"
}

# Acceder a valores
print(persona["nombre"])  # Ana
print(persona.get("edad"))  # 30
```

### Condicionales (if/else)

```python
edad = 18

if edad >= 18:
    print("Eres mayor de edad")
else:
    print("Eres menor de edad")

# Con elif (else if)
nota = 85

if nota >= 90:
    print("Excelente")
elif nota >= 70:
    print("Bien")
else:
    print("Necesitas mejorar")
```

### Bucles (loops)

```python
# For loop - iterar sobre lista
frutas = ["manzana", "banana", "naranja"]
for fruta in frutas:
    print(f"Me gusta la {fruta}")

# For loop - rango de números
for i in range(5):  # 0, 1, 2, 3, 4
    print(f"Número: {i}")

# While loop
contador = 0
while contador < 5:
    print(contador)
    contador += 1  # contador = contador + 1
```

### Funciones

```python
def saludar(nombre):
    return f"¡Hola {nombre}!"

resultado = saludar("María")
print(resultado)  # ¡Hola María!

# Función con múltiples parámetros
def sumar(a, b):
    return a + b

total = sumar(5, 3)
print(total)  # 8
```

---

## 📤 Variables y Output en GitHub Actions

### Leer Variables de Entorno de GitHub

```yaml
- name: Leer variables de GitHub
  shell: python
  run: |
    import os
    
    # Leer variables de GitHub
    repo = os.getenv('GITHUB_REPOSITORY')
    actor = os.getenv('GITHUB_ACTOR')
    ref = os.getenv('GITHUB_REF')
    sha = os.getenv('GITHUB_SHA')
    
    print(f"Repositorio: {repo}")
    print(f"Usuario: {actor}")
    print(f"Rama: {ref}")
    print(f"Commit: {sha}")
```

### Crear Output para Otros Steps

```yaml
- name: Generar outputs
  id: calcular
  shell: python
  run: |
    import os
    
    # Calcular algo
    resultado = 2 + 2
    fecha = "2026-01-30"
    
    # Escribir al archivo de output de GitHub
    github_output = os.getenv('GITHUB_OUTPUT')
    with open(github_output, 'a') as f:
        f.write(f'resultado={resultado}\n')
        f.write(f'fecha={fecha}\n')

- name: Usar outputs
  run: |
    echo "Resultado: ${{ steps.calcular.outputs.resultado }}"
    echo "Fecha: ${{ steps.calcular.outputs.fecha }}"
```

### Crear Variables de Entorno

```yaml
- name: Definir variables de entorno
  shell: python
  run: |
    import os
    
    # Valores calculados
    version = "1.2.3"
    ambiente = "production"
    
    # Escribir al archivo de entorno
    github_env = os.getenv('GITHUB_ENV')
    with open(github_env, 'a') as f:
        f.write(f'APP_VERSION={version}\n')
        f.write(f'ENVIRONMENT={ambiente}\n')

- name: Usar variables
  run: |
    echo "Versión: $APP_VERSION"
    echo "Ambiente: $ENVIRONMENT"
```

### Leer Secretos

```yaml
- name: Usar secretos
  shell: python
  env:
    API_KEY: ${{ secrets.API_KEY }}
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  run: |
    import os
    
    api_key = os.getenv('API_KEY')
    db_pass = os.getenv('DB_PASSWORD')
    
    # Verificar si existen (NO imprimirlos)
    if api_key:
        print("✅ API Key configurada")
    else:
        print("❌ API Key no encontrada")
        exit(1)
```

---

## 📁 Archivos y Directorios

### Leer Archivos

```yaml
- name: Leer archivo de texto
  shell: python
  run: |
    # Leer archivo completo
    with open('README.md', 'r') as f:
        contenido = f.read()
        print(contenido)
    
    # Leer línea por línea
    with open('requirements.txt', 'r') as f:
        for linea in f:
            print(linea.strip())
```

### Escribir Archivos

```yaml
- name: Crear archivo
  shell: python
  run: |
    # Escribir archivo
    with open('output.txt', 'w') as f:
        f.write("Primera línea\n")
        f.write("Segunda línea\n")
    
    # Añadir a archivo existente
    with open('output.txt', 'a') as f:
        f.write("Tercera línea\n")
    
    print("✅ Archivo creado")
```

### Verificar si Existe

```yaml
- name: Verificar archivos
  shell: python
  run: |
    import os
    
    # Verificar archivo
    if os.path.exists('README.md'):
        print("✅ README.md existe")
    else:
        print("❌ README.md no existe")
    
    # Verificar directorio
    if os.path.isdir('src/'):
        print("✅ Directorio src/ existe")
    
    # Verificar si es archivo
    if os.path.isfile('package.json'):
        print("✅ package.json es un archivo")
```

### Listar Archivos

```yaml
- name: Listar archivos
  shell: python
  run: |
    import os
    
    # Listar directorio actual
    archivos = os.listdir('.')
    print("Archivos en directorio actual:")
    for archivo in archivos:
        print(f"  - {archivo}")
    
    # Listar con más detalles
    for item in os.listdir('.'):
        if os.path.isfile(item):
            tamaño = os.path.getsize(item)
            print(f"📄 {item} ({tamaño} bytes)")
        elif os.path.isdir(item):
            print(f"📁 {item}/")
```

### Buscar Archivos Recursivamente

```yaml
- name: Buscar archivos Python
  shell: python
  run: |
    import pathlib
    
    # Buscar todos los archivos .py
    archivos_python = list(pathlib.Path('.').rglob('*.py'))
    
    print(f"Encontrados {len(archivos_python)} archivos Python:")
    for archivo in archivos_python:
        print(f"  - {archivo}")
```

### Crear Directorios

```yaml
- name: Crear directorios
  shell: python
  run: |
    import os
    
    # Crear un directorio
    os.mkdir('nuevo_dir')
    
    # Crear directorios anidados (incluye padres)
    os.makedirs('output/reports/2026', exist_ok=True)
    
    print("✅ Directorios creados")
```

---

## 📊 JSON y YAML

### Trabajar con JSON

```yaml
- name: Procesar JSON
  shell: python
  run: |
    import json
    
    # Crear objeto Python
    datos = {
        "nombre": "Mi Proyecto",
        "version": "1.0.0",
        "autor": "GitHub Actions",
        "dependencias": ["pytest", "requests"]
    }
    
    # Convertir a JSON y guardar en archivo
    with open('config.json', 'w') as f:
        json.dump(datos, f, indent=2)
    
    # Leer JSON desde archivo
    with open('config.json', 'r') as f:
        config = json.load(f)
        print(f"Proyecto: {config['nombre']}")
        print(f"Versión: {config['version']}")
    
    # Parsear JSON desde string
    json_string = '{"status": "success", "code": 200}'
    resultado = json.loads(json_string)
    print(f"Status: {resultado['status']}")
```

### Trabajar con YAML

```yaml
- name: Procesar YAML
  shell: python
  run: |
    import yaml
    
    # Leer YAML
    with open('application.yml', 'r') as f:
        config = yaml.safe_load(f)
    
    # Acceder a valores anidados
    project_name = config['metadata']['project_name']
    version = config['metadata']['version']
    
    print(f"Proyecto: {project_name}")
    print(f"Versión: {version}")
    
    # Crear y guardar YAML
    datos = {
        'database': {
            'host': 'localhost',
            'port': 5432,
            'name': 'mydb'
        },
        'features': ['auth', 'api', 'dashboard']
    }
    
    with open('config.yml', 'w') as f:
        yaml.dump(datos, f, default_flow_style=False)
```

### Procesar Evento de GitHub (JSON)

```yaml
- name: Procesar evento de GitHub
  shell: python
  run: |
    import json
    import os
    
    # GitHub pasa el evento como JSON en variable de entorno
    event_json = os.getenv('GITHUB_EVENT_PATH')
    
    with open(event_json, 'r') as f:
        event = json.load(f)
    
    # Extraer información según el tipo de evento
    if 'pull_request' in event:
        pr_number = event['pull_request']['number']
        pr_title = event['pull_request']['title']
        pr_author = event['pull_request']['user']['login']
        
        print(f"PR #{pr_number}: {pr_title}")
        print(f"Autor: {pr_author}")
    elif 'push' in event:
        commits = len(event['commits'])
        print(f"Push con {commits} commits")
```

---

## 🌐 APIs y HTTP

### Hacer Peticiones HTTP

```yaml
- name: Llamar API
  shell: python
  run: |
    import requests
    
    # GET request
    response = requests.get('https://api.github.com/repos/octocat/Hello-World')
    
    if response.status_code == 200:
        data = response.json()
        print(f"Repositorio: {data['full_name']}")
        print(f"Estrellas: {data['stargazers_count']}")
        print(f"Lenguaje: {data['language']}")
    else:
        print(f"Error: {response.status_code}")
```

### POST Request

```yaml
- name: Enviar datos
  shell: python
  run: |
    import requests
    
    url = 'https://httpbin.org/post'
    datos = {
        'usuario': 'test',
        'email': 'test@example.com'
    }
    
    response = requests.post(url, json=datos)
    print(f"Status: {response.status_code}")
    print(f"Respuesta: {response.json()}")
```

### GitHub API con Autenticación

```yaml
- name: GitHub API
  shell: python
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    import requests
    import os
    
    token = os.getenv('GITHUB_TOKEN')
    repo = os.getenv('GITHUB_REPOSITORY')
    
    headers = {
        'Authorization': f'Bearer {token}',
        'Accept': 'application/vnd.github.v3+json'
    }
    
    # Obtener issues
    url = f'https://api.github.com/repos/{repo}/issues'
    response = requests.get(url, headers=headers)
    
    issues = response.json()
    print(f"Issues abiertas: {len(issues)}")
    
    for issue in issues[:5]:  # Primeras 5
        print(f"  #{issue['number']}: {issue['title']}")
```

### Crear Comentario en PR

```yaml
- name: Comentar en PR
  if: github.event_name == 'pull_request'
  shell: python
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    import requests
    import os
    
    token = os.getenv('GITHUB_TOKEN')
    repo = os.getenv('GITHUB_REPOSITORY')
    pr_number = os.getenv('GITHUB_EVENT_NUMBER')
    
    url = f'https://api.github.com/repos/{repo}/issues/{pr_number}/comments'
    
    headers = {
        'Authorization': f'Bearer {token}',
        'Accept': 'application/vnd.github.v3+json'
    }
    
    comment = {
        'body': '✅ Build completado con éxito!'
    }
    
    response = requests.post(url, headers=headers, json=comment)
    
    if response.status_code == 201:
        print("✅ Comentario publicado")
    else:
        print(f"❌ Error: {response.status_code}")
```

---

## 🔧 Procesamiento de Datos

### Contar Líneas de Código

```yaml
- name: Contar líneas de código
  shell: python
  run: |
    import pathlib
    
    total_lines = 0
    total_files = 0
    
    # Extensiones a contar
    extensions = ['.py', '.js', '.java', '.ts']
    
    for ext in extensions:
        for file_path in pathlib.Path('.').rglob(f'*{ext}'):
            total_files += 1
            with open(file_path, 'r', errors='ignore') as f:
                lines = len(f.readlines())
                total_lines += lines
                print(f"{file_path}: {lines} líneas")
    
    print(f"\n📊 Total: {total_files} archivos, {total_lines} líneas")
```

### Análisis de Dependencias

```yaml
- name: Analizar requirements.txt
  shell: python
  run: |
    import re
    
    if not os.path.exists('requirements.txt'):
        print("❌ requirements.txt no encontrado")
        exit(1)
    
    with open('requirements.txt', 'r') as f:
        lineas = f.readlines()
    
    print("📦 Dependencias encontradas:\n")
    
    for linea in lineas:
        linea = linea.strip()
        if linea and not linea.startswith('#'):
            # Extraer nombre y versión
            match = re.match(r'([a-zA-Z0-9-_]+)(==|>=|<=)?(.+)?', linea)
            if match:
                nombre = match.group(1)
                operador = match.group(2) or ''
                version = match.group(3) or 'cualquiera'
                print(f"  - {nombre} {operador}{version}")
```

### Generar Reporte Markdown

```yaml
- name: Generar reporte
  shell: python
  run: |
    from datetime import datetime
    
    # Datos del reporte
    fecha = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    tests_passed = 42
    tests_failed = 0
    coverage = 85.5
    
    # Generar Markdown
    markdown = f"""
    # 📊 Reporte de Tests
    
    **Fecha:** {fecha}
    
    ## Resultados
    
    | Métrica | Valor |
    |---------|-------|
    | ✅ Tests Pasados | {tests_passed} |
    | ❌ Tests Fallados | {tests_failed} |
    | 📈 Cobertura | {coverage}% |
    
    ## Estado
    
    {'🎉 **Todos los tests pasaron!**' if tests_failed == 0 else '⚠️ **Hay tests fallando**'}
    """
    
    # Guardar reporte
    with open('report.md', 'w') as f:
        f.write(markdown)
    
    print("✅ Reporte generado: report.md")
```

### Procesar Logs

```yaml
- name: Analizar logs
  shell: python
  run: |
    import re
    from collections import Counter
    
    # Leer archivo de log
    with open('app.log', 'r') as f:
        logs = f.readlines()
    
    # Contar niveles de log
    levels = []
    errors = []
    
    for linea in logs:
        # Buscar nivel: [ERROR], [WARN], [INFO]
        match = re.search(r'\[(ERROR|WARN|INFO|DEBUG)\]', linea)
        if match:
            nivel = match.group(1)
            levels.append(nivel)
            
            if nivel == 'ERROR':
                errors.append(linea.strip())
    
    # Estadísticas
    counter = Counter(levels)
    
    print("📊 Estadísticas de Logs:\n")
    for nivel, cantidad in counter.most_common():
        print(f"  {nivel}: {cantidad}")
    
    if errors:
        print(f"\n❌ Errores encontrados ({len(errors)}):")
        for error in errors[:5]:  # Primeros 5
            print(f"  {error}")
```

---

## 🛠️ Utilidades Comunes

### Validar Versión Semántica

```yaml
- name: Validar versión
  shell: python
  run: |
    import re
    import sys
    
    version = "${{ github.ref_name }}"
    
    # Patrón SemVer: MAJOR.MINOR.PATCH
    pattern = r'^v?(\d+)\.(\d+)\.(\d+)(-[a-zA-Z0-9.]+)?(\+[a-zA-Z0-9.]+)?$'
    
    match = re.match(pattern, version)
    
    if match:
        major = match.group(1)
        minor = match.group(2)
        patch = match.group(3)
        prerelease = match.group(4) or ''
        build = match.group(5) or ''
        
        print(f"✅ Versión válida: {version}")
        print(f"  Major: {major}")
        print(f"  Minor: {minor}")
        print(f"  Patch: {patch}")
        if prerelease:
            print(f"  Pre-release: {prerelease}")
    else:
        print(f"❌ Versión inválida: {version}")
        print("Formato esperado: v1.2.3 o 1.2.3-beta+build")
        sys.exit(1)
```

### Buscar TODOs en Código

```yaml
- name: Buscar TODOs
  shell: python
  run: |
    import pathlib
    import re
    
    todos = []
    fixmes = []
    
    for py_file in pathlib.Path('.').rglob('*.py'):
        with open(py_file, 'r', errors='ignore') as f:
            for num_linea, linea in enumerate(f, 1):
                if 'TODO' in linea:
                    todos.append(f"{py_file}:{num_linea} - {linea.strip()}")
                if 'FIXME' in linea:
                    fixmes.append(f"{py_file}:{num_linea} - {linea.strip()}")
    
    if todos:
        print(f"📝 TODOs encontrados ({len(todos)}):\n")
        for todo in todos:
            print(f"  {todo}")
    
    if fixmes:
        print(f"\n🔧 FIXMEs encontrados ({len(fixmes)}):\n")
        for fixme in fixmes:
            print(f"  {fixme}")
    
    if not todos and not fixmes:
        print("✅ No hay TODOs ni FIXMEs pendientes")
```

### Calcular Hash de Archivos

```yaml
- name: Calcular checksums
  shell: python
  run: |
    import hashlib
    import pathlib
    
    def calcular_sha256(archivo):
        sha256 = hashlib.sha256()
        with open(archivo, 'rb') as f:
            while chunk := f.read(8192):
                sha256.update(chunk)
        return sha256.hexdigest()
    
    print("🔐 Checksums SHA-256:\n")
    
    # Calcular para archivos dist/
    for archivo in pathlib.Path('dist').glob('*'):
        if archivo.is_file():
            checksum = calcular_sha256(archivo)
            print(f"{archivo.name}")
            print(f"  {checksum}\n")
```

### Comparar Versiones

```yaml
- name: Comparar versiones
  shell: python
  run: |
    from packaging import version
    
    actual = "1.2.3"
    requerida = "1.0.0"
    
    if version.parse(actual) >= version.parse(requerida):
        print(f"✅ Versión {actual} cumple requisito >= {requerida}")
    else:
        print(f"❌ Versión {actual} no cumple requisito >= {requerida}")
        exit(1)
```

### Generar ID Único

```yaml
- name: Generar ID
  id: gen-id
  shell: python
  run: |
    import uuid
    import os
    from datetime import datetime
    
    # UUID aleatorio
    unique_id = str(uuid.uuid4())
    
    # Timestamp
    timestamp = datetime.now().strftime('%Y%m%d-%H%M%S')
    
    # ID combinado
    build_id = f"build-{timestamp}-{unique_id[:8]}"
    
    print(f"Build ID: {build_id}")
    
    # Guardar como output
    with open(os.getenv('GITHUB_OUTPUT'), 'a') as f:
        f.write(f'build-id={build_id}\n')
```

### Medir Tiempo de Ejecución

```yaml
- name: Medir tiempo
  shell: python
  run: |
    import time
    from datetime import datetime
    
    inicio = time.time()
    print(f"Inicio: {datetime.now()}")
    
    # Simular trabajo
    time.sleep(2)
    
    fin = time.time()
    duracion = fin - inicio
    
    print(f"Fin: {datetime.now()}")
    print(f"⏱️ Duración: {duracion:.2f} segundos")
```

---

## 🎓 Conceptos de Python que Necesitas Saber

### Módulos e Imports

```python
# Importar módulo completo
import os
import json

# Usar funciones del módulo
ruta = os.getcwd()
datos = json.dumps({'key': 'value'})

# Importar función específica
from datetime import datetime
ahora = datetime.now()

# Importar con alias
import requests as req
response = req.get('https://api.example.com')
```

### Manejo de Errores

```python
try:
    # Código que puede fallar
    with open('archivo.txt', 'r') as f:
        contenido = f.read()
except FileNotFoundError:
    print("❌ Archivo no encontrado")
except Exception as e:
    print(f"❌ Error: {e}")
else:
    print("✅ Todo bien")
finally:
    print("Esto siempre se ejecuta")
```

### List Comprehensions

```python
# En vez de:
numeros_pares = []
for i in range(10):
    if i % 2 == 0:
        numeros_pares.append(i)

# Usar:
numeros_pares = [i for i in range(10) if i % 2 == 0]
# [0, 2, 4, 6, 8]
```

### Formateo de Strings

```python
nombre = "Ana"
edad = 25

# f-strings (recomendado)
mensaje = f"Hola {nombre}, tienes {edad} años"

# format()
mensaje = "Hola {}, tienes {} años".format(nombre, edad)

# % (antiguo)
mensaje = "Hola %s, tienes %d años" % (nombre, edad)
```

### Operadores Útiles

```python
# Aritméticos
suma = 5 + 3        # 8
resta = 5 - 3       # 2
multiplicacion = 5 * 3  # 15
division = 5 / 3    # 1.666...
division_entera = 5 // 3  # 1
modulo = 5 % 3      # 2 (resto)
potencia = 5 ** 3   # 125

# Comparación
5 == 5  # True (igual)
5 != 3  # True (diferente)
5 > 3   # True (mayor)
5 < 3   # False (menor)
5 >= 5  # True (mayor o igual)

# Lógicos
True and False  # False
True or False   # True
not True        # False

# Pertenencia
'a' in 'hola'   # True
5 in [1,2,3]    # False
```

---

## 🎯 Cheatsheet Rápido

### Variables de GitHub Más Usadas

```python
import os

# Información del repositorio
repo = os.getenv('GITHUB_REPOSITORY')          # usuario/repo
actor = os.getenv('GITHUB_ACTOR')              # usuario que ejecutó
sha = os.getenv('GITHUB_SHA')                  # hash del commit
ref = os.getenv('GITHUB_REF')                  # refs/heads/main
ref_name = os.getenv('GITHUB_REF_NAME')        # main

# Workspace
workspace = os.getenv('GITHUB_WORKSPACE')      # /home/runner/work/repo/repo
runner_temp = os.getenv('RUNNER_TEMP')         # directorio temporal

# Outputs y Env
github_output = os.getenv('GITHUB_OUTPUT')     # archivo de outputs
github_env = os.getenv('GITHUB_ENV')           # archivo de env vars
```

### Operaciones de Archivo Comunes

```python
import os
import pathlib

# Verificar existencia
os.path.exists('archivo.txt')
os.path.isfile('archivo.txt')
os.path.isdir('directorio/')

# Crear/eliminar
os.mkdir('nuevo_dir')
os.makedirs('path/to/dir', exist_ok=True)
os.remove('archivo.txt')
os.rmdir('directorio/')

# Información
os.path.getsize('archivo.txt')      # tamaño en bytes
os.path.basename('/path/to/file')   # file
os.path.dirname('/path/to/file')    # /path/to

# Buscar archivos
list(pathlib.Path('.').rglob('*.py'))
```

### JSON y YAML Rápido

```python
import json
import yaml

# JSON
with open('data.json', 'r') as f:
    data = json.load(f)

with open('output.json', 'w') as f:
    json.dump(data, f, indent=2)

# YAML
with open('config.yml', 'r') as f:
    config = yaml.safe_load(f)

with open('output.yml', 'w') as f:
    yaml.dump(data, f)
```

---

*Documento de referencia rápida para Python en GitHub Actions*
*Última actualización: Enero 2026*

