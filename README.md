# MPG Link Shortener (Acortador de Enlaces para Discord)

Un sistema híbrido de acortado de URLs diseñado específicamente para **Bot Designer For Discord (BDFD)**. Este repositorio contiene el comando nativo listo para BDFD utilizando funciones `$httpGet` y la infraestructura del backend en Python (Flask + SQLite) lista para ser auto-alojada.

---

## 🚀 Comando BDFD (Código Principal)

Crea un comando en tu panel de BDFD con el disparador (trigger) que desees (por ejemplo: `!aurl` o `!acortar`) y pega el siguiente código:

```rb
$nomention
$var[url;$message]
$httpGet[https://mpgacortador.pythonanywhere.com/acortar?url=$var[url]]
$if[$message==]
❌ ¡Por favor, introduce una URL para acortar!
$deletecommand
$deleteIn[5s]
$elseif[$httpStatus==200]
$title[🔗 Enlace Acortado con Éxito]
$description[Aquí tienes tu enlace personalizado generado por el sistema MPG:

**🔗 URL Funcional:** $httpResult[url_corta]
**🆔 ID Único:** `$httpResult[id]`]
$color[#43b581]
$footer[Sistema de acortado By MPG]
$deletecommand
$else 
❌️ Proporciona una URL válida $httpStatus 
$deletecommand
$deleteIn[5s]
$endif
```
<div align="center">
  <img src="https://i.ibb.co/7ddqz47m/Screenshot-20260624-232950-Discord.jpg" alt="Ejemplo del comando de BDFD acortador de URL's " width="500">
</div>

## ⚙️ Modificación para VPS o Dominio Propio
Si dispones de un servidor VPS, un hosting dedicado o un dominio propio, puedes clonar este proyecto y adaptarlo a tu marca en segundos. Solo debes modificar las siguientes líneas del entorno:
 1. **En el archivo de Python (flask_app.py):** Cambia el valor de la variable BASE_DOMINIO introduciendo tu dominio personalizado o la IP de tu VPS:
(El flask_app.py lo digo por si el dominio propio es de PythonAnywhere)

   ```python
   BASE_DOMINIO = "https://tu-dominio-o-vps.com/"
   
   ```

 2. **En el comando de BDFD:** Modifica la URL que se encuentra dentro de la función $httpGet para que apunte a tu nueva dirección:
   ```rb
   $httpGet[https://tu-dominio-o-vps.com/acortar?url=$var[url]]
   
   ```
## 🐍 Servidor Backend (Python + Flask)
Este es el código del servidor que procesa las peticiones de Discord, genera identificadores alfanuméricos únicos que no se repiten jamás y maneja la base de datos local mediante SQLite de forma automática.

```python
import string
import secrets
import sqlite3
import os
from flask import Flask, request, redirect, jsonify

app = Flask(__name__)

BASE_DOMINIO = "https://mpgacortador.pythonanywhere.com/"
DB_PATH = os.path.join(os.path.dirname(__file__), 'urls.db')

def init_db():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS enlaces (
            id TEXT PRIMARY KEY,
            url_larga TEXT NOT NULL
        )
    ''')
    conn.commit()
    conn.close()

def generar_id_unico(longitud=6):
    caracteres = string.ascii_letters + string.digits
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    while True:
        nuevo_id = ''.join(secrets.choice(caracteres) for _ in range(longitud))
        cursor.execute('SELECT 1 FROM enlaces WHERE id = ?', (nuevo_id,))
        if not cursor.fetchone():
            conn.close()
            return nuevo_id

@app.route('/acortar', methods=['GET'])
def acortar():
    try:
        url_larga = request.args.get('url')
        if not url_larga:
            return jsonify({"error": "Falta la URL"}), 400
            
        id_unico = generar_id_unico()
        
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute('INSERT INTO enlaces (id, url_larga) VALUES (?, ?)', (id_unico, url_larga))
        conn.commit()
        conn.close()
        
        return jsonify({
            "status": "success",
            "url_corta": f"{BASE_DOMINIO}{id_unico}",
            "id": id_unico
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route('/<id_unico>', methods=['GET'])
def redireccionar(id_unico):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute('SELECT url_larga FROM enlaces WHERE id = ?', (id_unico,))
        resultado = cursor.fetchone()
        conn.close()
        
        if resultado:
            return redirect(resultado[0])
        return "<h1>Error: Enlace no encontrado en el sistema MPG.</h1>", 404
    except Exception as e:
        return f"Error interno: {str(e)}", 500

try:
    init_db()
except Exception:
    pass

```

Darle las gracias a [TwisSpark](https://discord.gg/NKDTAnQKXC) inspirarme a usar Python en algun proyecto 
