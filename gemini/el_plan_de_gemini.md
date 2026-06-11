# 🗺️ Plan de Trabajo: Plugin de QGIS para Descarga de Catastro INSPIRE

Este plan de trabajo está estructurado para desarrollar el complemento de forma modular, asegurando la calidad del código y utilizando la IA como mentora en cada fase.

---

## 🛠️ Fase 0: Configuración del Entorno de Trabajo

### Paso 0.1: Preparar las carpetas del proyecto
1. Localiza la carpeta de plugins de QGIS en tu sistema.
   * *Ruta típica en Windows:* `C:\Usuarios\TuUsuario\AppData\Roaming\QGIS\QGIS3\profiles\default\python\plugins\`
2. Crea una carpeta nueva llamada `descargador_catastro`.
3. Abre **Visual Studio Code**, ve a `Archivo > Abrir carpeta` y selecciona la carpeta que acabas de crear.

### Paso 0.2: Inicializar Git (Tu red de seguridad)
1. Abre la terminal integrada de VS Code (`Ctrl + Ñ` o `Ctrl + \``).
2. Ejecuta el comando: `git init`
3. Crea un archivo llamado `.gitignore` y añade la línea `__pycache__/` para evitar subir archivos temporales de Python.

### Paso 0.3: Instalar Extensiones en VS Code
Instala las siguientes herramientas gratuitas desde el mercado de extensiones (`Ctrl+Shift+X`):
* [ ] **Python** (Microsoft)
* [ ] **Black Formatter** (Microsoft) -> *Configúralo para que formatee al guardar (`Editor: Format On Save` en los ajustes).*
* [ ] **Codeium** (para autocompletado e IA ilimitada) o **Continue.dev** conectado a la API de Gemini.

### Paso 0.4: Instalar Plugins de Desarrollo en QGIS
1. Abre QGIS.
2. Ve a `Complementos > Administrar e instalar complementos`.
3. Busca e instala:
   * [ ] **Plugin Reloader** (para recargar tu plugin sin reiniciar QGIS).
   * [ ] **QGIS DevTools** o **Remote Debugger** (para conectar QGIS con VS Code).

---

## 📂 Fase 1: Esqueleto del Plugin y Metadatos (QGIS Core)

El objetivo de esta fase es que QGIS reconozca la existencia de tu plugin, aunque todavía no haga nada.

### Paso 1.1: Archivos mínimos requeridos
*   **Instrucción para la IA:**
    > "Quiero empezar el desarrollo de mi plugin de QGIS para descargar datos de Catastro. Actúa como mi tutor de PyQGIS. Necesito crear los archivos mínimos obligatorios para que QGIS reconozca el complemento en su listado. Genérame el código y la explicación para los archivos `metadata.txt` (asigna nombres descriptivos y autor Patricio) e `__init__.py`. Explícame en comentarios qué hace cada línea."

### Paso 1.2: Registro en el diario de aprendizaje
*   **Acción:** Guarda el código generado en sus respectivos archivos, reinicia QGIS y verifica que el plugin aparece en el administrador de complementos.
*   **Instrucción para la IA:**
    > "He conseguido que QGIS reconozca el plugin. Redacta una entrada breve para mi `DIARIO.md` en formato Markdown que explique qué son el archivo `metadata.txt` y el punto de entrada `__init__.py` en la arquitectura de QGIS."

---

## 💻 Fase 2: Diseño de la Interfaz Gráfica (UI)

Diseñaremos una interfaz sencilla y limpia separada de la lógica del programa.

### Paso 2.1: Crear el archivo `.ui` con Qt Designer
1. Abre **Qt Designer** (viene instalado junto con QGIS).
2. Crea un nuevo diálogo (`Dialog without Buttons`).
3. Diseña la interfaz con tres elementos clave:
   * Un `QComboBox` (desplegable) para seleccionar el municipio.
   * Un grupo de `QCheckBox` (casillas) para elegir el tipo de datos (Parcelas, Edificios, Direcciones).
   * Un `QPushButton` (botón) para iniciar la descarga.
4. Guarda el archivo como `interface.ui` dentro de tu carpeta en VS Code.

### Paso 2.2: Cargar la interfaz en Python
*   **Instrucción para la IA:**
    > "Tengo un archivo de interfaz visual llamado `interface.ui` en la raíz de mi plugin. Necesito escribir el archivo principal del plugin en Python (llámalo `main.py`). Genera el código necesario utilizando `uic.loadUiType` para cargar esta interfaz de forma dinámica y añade los métodos básicos de inicialización del plugin (`initGui` y `unloadUi`) para integrarlo en el menú de QGIS. Recuerda separar la interfaz de la lógica pesada."

---

## ⚙️ Fase 3: Lógica Pura en Python (Descarga y Parseo de Catastro)

Aquí programaremos las funciones que interactúan con internet y el XML de Catastro, completamente aisladas de QGIS.

### Paso 3.1: Mapeo de Municipios y lectura de ATOM
*   **Instrucción para la IA:**
    > "Vamos a programar la lógica del servicio ATOM de Catastro INSPIRE. Crea un archivo independiente llamado `downloader.py`. Necesito una función en Python que reciba el nombre de un municipio, busque su código cadastral correspondiente en un diccionario estático JSON (por ejemplo, 'Córdoba': '14021') y luego lea el archivo XML del feed ATOM (ej. `https://www.catastro.hacienda.gob.es/INSPIRE/CadastralParcels/ES.SDGC.CP.atom.xml`) para extraer la URL exacta de descarga del archivo ZIP de ese municipio. No uses librerías de QGIS aquí, solo Python puro (`requests` y `xml.etree.ElementTree`). Dame el pseudocódigo o los pasos en comentarios primero."

### Paso 3.2: Descarga y descompresión del ZIP
*   **Instrucción para la IA:**
    > "Añade una función al archivo `downloader.py` que reciba la URL del archivo ZIP del municipio obtenida en el paso anterior, lo descargue en una carpeta temporal del sistema y lo descomprima. La función debe devolver la ruta local del archivo `.gml` que se encuentra dentro del ZIP. Añade control de errores con `try/except` por si el servicio de Catastro está caído o no hay internet."

### Paso 3.3: Generación de Docstrings y documentación de lógica
*   **Instrucción para la IA:**
    > "Analiza las dos funciones de `downloader.py` que acabamos de escribir. Genera sus *docstrings* profesionales en formato Google que detallen los parámetros de entrada, salida y excepciones controladas para que mi código sea limpio y legible para desarrolladores avanzados."

---

## 🔌 Fase 4: Conexión de Componentes y Carga en QGIS (PyQGIS)

El paso final: conectar el botón de la pantalla con la lógica de descarga y pintar los datos en el mapa de QGIS.

### Paso 4.1: Conectar el botón con la función de descarga
*   **Instrucción para la IA:**
    > "Quiero conectar el botón de descarga de mi interfaz (`interface.ui`) con las funciones que programamos en `downloader.py`. Modifica mi archivo `main.py` para que, al pulsar el botón, capture el municipio seleccionado en el `QComboBox`, llame a la lógica de descarga y obtenga la ruta del archivo GML descargado."

### Paso 4.2: Cargar la capa vectorial en QGIS de forma nativa
*   **Instrucción para la IA:**
    > "Una vez que tengo la ruta local del archivo GML de Catastro, necesito que el plugin lo cargue automáticamente como una capa vectorial en QGIS. Escribe la función de PyQGIS necesaria para añadir esta capa al mapa de forma limpia. Ten en cuenta que los datos de Catastro INSPIRE vienen en el sistema de coordenadas europeo EPSG:4258, asegúrate de indicarlo en el código al cargar la capa."

### Paso 4.3: Registro en el diario de aprendizaje
*   **Acción:** Haz un `git commit` con el mensaje "Plugin funcional - Versión Alfa".

---

## 📝 Fase 5: Documentación Final y Cierre del Proyecto

### Paso 5.1: Redacción del README profesional
*   **Instrucción para la IA (Usa el comando @Folder o @Codebase en el chat de tu extensión):**
    > "Analiza toda la estructura de archivos y el código de nuestro plugin de descarga de Catastro. Genera un archivo `README.md` profesional. Debe incluir una descripción del plugin, la arquitectura del código explicando la separación de interfaz y lógica, las dependencias requeridas y una guía clara sobre cómo puede instalarlo otro usuario de QGIS copiando la carpeta en su perfil local."
