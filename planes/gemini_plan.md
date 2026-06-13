Aquí tienes el plan de trabajo completo presentado directamente en el chat en formato Markdown para que puedas copiarlo y pegarlo cómodamente en un archivo local (por ejemplo, 

# Plan de Trabajo: Desarrollo de Complemento QGIS para Descarga de Datos de Catastro

Este plan de trabajo está estructurado para guiarte paso a paso en el desarrollo de tu primer plugin de QGIS, aplicando buenas prácticas de desarrollo y utilizando la IA de forma agéntica y didáctica.


## 🚀 Fase 0: Configuración del Entorno de Trabajo

**Objetivo:** Preparar VS Code, las extensiones necesarias y crear la estructura base del plugin.

### Paso 1: Instalar herramientas en QGIS
1. Abre QGIS.
2. Ve a **Complementos > Administrar e instalar complementos**.
3. Busca e instala **Plugin Builder 3** y **Plugin Reloader** (este último sirve para recargar el código de tu plugin sin tener que reiniciar QGIS cada vez que hagas cambios).

### Paso 2: Generar el esqueleto del plugin
1. Abre **Plugin Builder 3** en QGIS (suele aparecer con el icono de una herramienta o en el menú *Complementos*).
2. Rellenar los datos básicos del formulario:
   - *Class name*: `CatastroDownloader`
   - *Plugin name*: `Catastro Downloader`
   - *Description*: Descarga de datos INSPIRE de Catastro por municipio.
   - *Module name*: `catastro_downloader`
3. Guarda el plugin en la carpeta de complementos de tu perfil activo de QGIS (habitualmente se encuentra en: `AppData\Roaming\QGIS\QGIS3\profiles\default\python\plugins\`).

### Paso 3: Configurar VS Code
1. Abre VS Code, ve a *Archivo > Abrir carpeta* y selecciona la carpeta del plugin recién generada.
2. Asegúrate de tener instaladas las siguientes extensiones en tu VS Code: *Python (Microsoft)*, *Claude Code*, *PyQGIS 3 Autocomplete* y *Ruff*.
3. Cambia el intérprete de Python de VS Code (`Ctrl+Shift+P` -> `Python: Select Interpreter`) y selecciona el binario de Python que viene integrado dentro de la instalación de tu QGIS.
4. Crea un archivo llamado `CLAUDE.md` en la raíz del proyecto para alinear las directrices de calidad. Pega dentro lo siguiente:

```markdown
   # Proyecto: QGIS Catastro Downloader Plugin
   ## Tecnologías
   - Python 3.x
   - PyQGIS (QGIS 3.x API)
   - PyQt5 (Para la interfaz gráfica)
   ## Buenas Prácticas de Código
   - Estilo estricto PEP 8.
   - Siempre usar Type Hinting en funciones.
   - Comentarios y Docstrings en formato Google.
   - Separar la lógica de interfaz (GUI) de los servicios de descarga (ATOM de Catastro).

```

---

## 📐 Fase 1: Arquitectura de Archivos y Reglas de Calidad

**Objetivo:** Establecer la separación de conceptos (Capa GUI, Capa de Servicio y Capa QGIS Core) antes de picar código.

### Interacción con la IA

* **Tipo:** Agéntica / Creación de archivos.
* **Prompt exacto a lanzar:**

> Actúa como un Desarrollador Senior de Python y QGIS. Analiza el esqueleto de archivos de este plugin. Necesito estructurar el proyecto siguiendo una arquitectura limpia y separando conceptos para que sea mantenible.
> Por favor, realiza las siguientes acciones de forma autónoma:
> 1. Lee nuestro archivo 'CLAUDE.md' para adoptar las reglas de estilo (PEP 8, Type Hints, Google Docstrings).
> 2. Crea una carpeta llamada 'services' dentro del directorio del plugin.
> 3. Dentro de 'services', crea un archivo vacío llamado 'catastro_client.py' que contendrá la lógica de conexión con el ATOM de Catastro (Python puro, sin dependencias de QGIS).
> 4. Crea una carpeta llamada 'core' dentro del directorio del plugin para futuras funciones geoespaciales.
> 5. Asegúrate de generar los archivos '**init**.py' necesarios en estas nuevas carpetas para que Python reconozca los módulos de forma correcta.
> 
> 
> Explícame brevemente la estructura final creada.

---

## 🎨 Fase 2: Diseño de la Interfaz Gráfica (UI) y Vinculación

**Objetivo:** Diseñar el formulario en Qt Designer y conectar los desplegables de forma estática con el código de Python.

### Paso 1: Diseño Visual

1. Localiza el archivo `catastro_downloader_dialog_base.ui` en tu carpeta del plugin y ábrelo con **Qt Designer** (instalado junto a QGIS).
2. Modifica o arrastra componentes para diseñar una ventana sencilla que tenga:
* Un `QComboBox` con el objectName: `combo_provincia` (para seleccionar provincia).
* Un `QComboBox` con el objectName: `combo_municipio` (para seleccionar municipio).
* Un `QComboBox` con el objectName: `combo_tipo_dato` (con opciones fijas: Parcelas, Edificios, Direcciones).
* Un `QPushButton` con el objectName: `btn_descargar` (para ejecutar la acción).
* Un `QProgressBar` con el objectName: `progress_bar` (para controlar visualmente la descarga).


3. Guarda los cambios del archivo `.ui`.

### Interacción con la IA

* **Tipo:** Lectura de UI y Modificación de código (`plugin.py` o archivo de diálogo principal).
* **Prompt exacto a lanzar:**

> Analiza el archivo UI de nuestra interfaz gráfica '@catastro_downloader_dialog_base.ui' para identificar los objectName de los componentes (combos, botón y barra de progreso).
> Quiero que modifiques el código del diálogo principal de nuestro plugin ('catastro_downloader_dialog.py' o el archivo de control correspondiente) para hacer lo siguiente:
> 1. En el método de inicialización, rellena de forma estática el combo de tipos de datos con las opciones: 'Parcelas', 'Edificios' y 'Direcciones'.
> 2. Deja preparados los métodos vacíos (handlers) 'on_provincia_changed', 'on_municipio_changed' y 'on_btn_descargar_clicked'.
> 3. Conecta las señales nativas de los combos y del botón a estos nuevos métodos usando buenas prácticas de PyQt5 (por ejemplo, el evento .currentIndexChanged o .clicked).
> 4. Recuerda incluir Type Hints y Docstrings explicativos.
> 
> 
> Haz los cambios directamente en el archivo correspondiente utilizando tus habilidades de parcheo.

---

## 🌐 Fase 3: El Servicio de Datos (Parseo de ATOM de Catastro)

**Objetivo:** Escribir la lógica en Python puro para leer los ficheros XML del ATOM de Catastro y obtener las URLs de descarga de los ZIPs de los municipios.

### Interacción con la IA

* **Tipo:** Modo Planificación + Implementación de código lógico.
* **Prompt exacto a lanzar:**

> Vamos a desarrollar la lógica de negocio en 'services/catastro_client.py' para interactuar con los feeds ATOM de Catastro de España. Las URLs raíz de los servicios ATOM son:
> * Parcelas: https://www.catastro.hacienda.gob.es/INSPIRE/CadastralParcels/ES.SDGC.CP.atom.xml
> * Direcciones: https://www.catastro.hacienda.gob.es/INSPIRE/Addresses/ES.SDGC.AD.atom.xml
> * Edificios: https://www.catastro.hacienda.gob.es/INSPIRE/buildings/ES.SDGC.BU.atom.xml
> 
> 
> El flujo de Catastro requiere:
> 1. Leer el XML raíz para extraer el listado de provincias y sus URLs de feeds ATOM secundarias.
> 2. Leer el XML de la provincia seleccionada para extraer el listado de municipios y la URL final de su archivo ZIP (GML).
> 
> 
> Por favor, sigue estos pasos de manera estricta:
> 1. Primero, entra en MODO PLANIFICACIÓN y explícame la estructura de clases y métodos que propones (ej. usar 'xml.etree.ElementTree' y 'requests'). Incluye control de excepciones (errores HTTP, XML malformados) y tratamiento de caracteres especiales/tildes en nombres de municipios. NO modifiques ningún archivo aún.
> 2. Una vez te dé el visto bueno en el chat tras revisar tu propuesta, implementa el código completo en 'services/catastro_client.py' usando Type Hints y Docstrings en formato Google.
> 
> 

---

## ⚙️ Fase 4: Segundo Plano (Evitar congelamiento de QGIS) y Descarga

**Objetivo:** Conectar el botón de descarga para bajar el archivo ZIP/GML sin bloquear la interfaz de usuario de QGIS utilizando las herramientas de tareas nativas.

### Interacción con la IA

* **Tipo:** Integración de PyQGIS e hilos de ejecución (`QgsTask`).
* **Prompt exacto a lanzar:**

> Necesito conectar la acción del botón de descarga en nuestro plugin principal. Como la descarga de los datos GML de Catastro por municipio puede ser pesada, NO podemos usar peticiones síncronas convencionales que congelen la interfaz de QGIS y muestren la aplicación en blanco.
> Por favor, modifica nuestro código para integrar la descarga utilizando la clase nativa 'QgsTask' de PyQGIS:
> 1. Crea una tarea que reciba la URL del ZIP del municipio obtenido desde nuestro 'catastro_client.py'.
> 2. Esta tarea debe descargar el archivo en una ubicación temporal del sistema de forma segura (controlando rutas en Windows y caracteres especiales).
> 3. Conecta el progreso de la descarga con nuestra 'progress_bar' de la UI de forma dinámica.
> 4. Si la descarga tiene éxito, debe llamar a una función de callback que prepare el archivo para su descompresión. Si falla, debe capturar la excepción y mostrar un 'QgsMessageLog' o un 'QMessageBox' crítico pero amigable para el usuario.
> 
> 
> Realiza la implementación de forma limpia y explícame línea por línea cómo funciona la comunicación entre el hilo secundario (QgsTask) y el hilo principal de la interfaz de QGIS para asegurar mi aprendizaje.

---

## 🗺️ Fase 5: Carga de la Capa en QGIS y Gestión de Coordenadas

**Objetivo:** Descomprimir el archivo descargado, leer el GML e incorporarlo al lienzo del mapa de QGIS controlando el Sistema de Referencia de Coordenadas (CRS).

### Interacción con la IA

* **Tipo:** Implementación nativa GIS / Depuración / Evaluación.
* **Prompt exacto a lanzar:**

> Estamos en la fase final de la carga de datos en el mapa de QGIS. Una vez descargado el ZIP temporal del municipio, necesitamos procesarlo.
> Por favor, implementa las siguientes funciones en la capa correspondiente del plugin ('core' o el módulo principal):
> 1. Una función que descomprima de forma segura el archivo ZIP en una carpeta temporal, lidiando correctamente con posibles problemas de codificación de caracteres en nombres de archivos.
> 2. Una función que localice el archivo GML resultante y lo cargue en el panel de capas de QGIS usando 'QgsVectorLayer' y 'QgsProject.instance().addMapLayer()'.
> 3. Los datos de Catastro vienen por defecto en EPSG:25830 (ETRS89 / UTM zone 30N) para la península. Asegúrate de configurar el CRS correcto en la capa al cargarla. Verifica también el CRS del proyecto activo en QGIS por si fuese necesario que el motor realice una reproyección al vuelo para evitar desplazamientos espaciales en el mapa del usuario.
> 
> 
> Una vez escrito el código, simula un proceso de 'Code Review' y hazme 2 o 3 preguntas conceptuales breves sobre este bloque de código para verificar que he entendido cómo gestiona QGIS las capas vectoriales y los CRS.

---

## 🔍 Diario de Aprendizaje Recomendado (Para el final de cada hito)

A medida que completes cada una de las fases con éxito, puedes lanzar este prompt genérico para documentar tus lecciones y tener tu propio manual de consulta técnico adaptado:

> Claude, acabamos de terminar con éxito la [Fase X]. Por favor, genera un resumen técnico para mi archivo 'DIARIO.md' que contenga: 1) Qué componentes de PyQGIS o librerías de Python hemos aprendido a usar hoy, 2) El error más común que evitamos en esta fase, y 3) Un pequeño fragmento de 5 líneas del código clave comentado de forma ultra-didáctica.
