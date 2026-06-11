# Plan de trabajo: Plugin QGIS para descarga de datos Catastro INSPIRE

> **Perfil**: Usuario con conocimientos básicos de Python y experiencia en desarrollo web  
> **Objetivo**: Crear un complemento QGIS para descargar y cargar datos de Catastro INSPIRE (parcelas, direcciones, edificios) por municipio  
> **Herramientas**: VS Code + Gemini Code Assist + Claude.ai + GitHub

---

## Fase 0 — Configuración del entorno de trabajo

### 0.1 Instalaciones base

- [ ] Instalar **VS Code**: https://code.visualstudio.com
- [ ] Instalar **Python 3.x**: https://python.org
- [ ] Instalar **QGIS** (versión LTR recomendada): https://qgis.org
- [ ] Crear cuenta en **GitHub**: https://github.com
- [ ] Crear cuenta en **Claude.ai**: https://claude.ai (plan gratuito)

### 0.2 Extensiones VS Code

Instalar en este orden desde el panel de extensiones (Ctrl+Shift+X):

| Extensión | Publisher | Para qué |
|---|---|---|
| Python | Microsoft | Soporte base Python |
| Pylance | Microsoft | Autocompletado inteligente |
| autopep8 | Microsoft | Formato automático PEP8 |
| Gemini Code Assist | Google | IA para código (gratuito) |
| XML Tools | Josh Johnson | Explorar feeds ATOM |
| GitLens | GitKraken | Control de versiones |

### 0.3 Configuración VS Code

Abrir `settings.json` (Ctrl+Shift+P → "Open User Settings JSON") y añadir:

```json
{
  "editor.formatOnSave": true,
  "python.formatting.provider": "autopep8",
  "editor.rulers": [79],
  "files.encoding": "utf8"
}
```

### 0.4 Repositorio GitHub

- [ ] Crear nuevo repositorio en GitHub: `qgis-catastro-downloader`
- [ ] Marcar como público (necesario para publicar en repositorio QGIS)
- [ ] Añadir `.gitignore` para Python
- [ ] Clonar en local con VS Code (Ctrl+Shift+P → "Git: Clone")

### 0.5 Estructura inicial de carpetas

Crear manualmente esta estructura en la carpeta del repositorio:

```
qgis-catastro-downloader/
├── .github/
│   └── copilot-instructions.md   # (se crea en Fase 4)
├── core/
│   ├── atom_parser.py
│   ├── downloader.py
│   └── qgis_loader.py
├── ui/
│   └── main_dialog.py
├── tests/
│   └── test_atom_parser.py
├── DEVLOG.md
├── README.md
└── metadata.txt
```

---

## Fase 1 — Entender los feeds ATOM de Catastro

> **Objetivo**: Comprender la estructura XML antes de escribir código  
> **Tiempo estimado**: 1-2 horas  
> **Herramienta**: Navegador + VS Code con XML Tools

### 1.1 Exploración manual

Abrir en el navegador los tres feeds y estudiar su estructura:

- Parcelas: https://www.catastro.hacienda.gob.es/INSPIRE/CadastralParcels/ES.SDGC.CP.atom.xml
- Direcciones: https://www.catastro.hacienda.gob.es/INSPIRE/Addresses/ES.SDGC.AD.atom.xml
- Edificios: https://www.catastro.hacienda.gob.es/INSPIRE/buildings/ES.SDGC.BU.atom.xml

### 1.2 Puntos clave a identificar

- El feed raíz lista **provincias**, cada una con su propio sub-feed
- Cada sub-feed de provincia lista **municipios** con URL de descarga directa (ZIP)
- Anotar en `DEVLOG.md` la estructura encontrada

### 1.3 Prompt para Claude

> *"Adjunto el contenido XML de este feed ATOM de Catastro. Explícame su estructura jerárquica: qué contiene el feed raíz, qué contiene cada entrada, cómo llegar hasta la URL de descarga de un municipio concreto. Dame un ejemplo con datos reales del XML."*

---

## Fase 2 — Script Python: parsear el feed ATOM

> **Objetivo**: Script independiente que lea el feed y devuelva la lista de municipios  
> **Tiempo estimado**: 2-4 horas  
> **Archivo**: `core/atom_parser.py`  
> **Herramienta**: VS Code + Gemini Code Assist + Claude.ai para dudas

### 2.1 Prompt inicial para Gemini Code Assist

Abrir `core/atom_parser.py` en VS Code y lanzar en el chat del agente:

> *"Write a Python function called `get_municipalities(feed_url, layer_type)` that:*
> *1. Fetches the ATOM XML from the given URL using the `requests` library*
> *2. Parses the XML with `xml.etree.ElementTree`*
> *3. Navigates two levels: root feed → province entries → municipality entries*
> *4. Returns a list of dicts with keys: `name`, `code`, `province`, `download_url`*
> *Follow PEP8 strictly. Add a complete docstring. Use descriptive variable names in English. Handle network errors with try/except."*

### 2.2 Prompt para testear en QGIS Python Console

> *"Write a minimal test script I can run in QGIS Python Console to call `get_municipalities()` with the Catastro CP feed URL and print the first 5 results."*

### 2.3 Prompt para documentar en DEVLOG

> *"Acabo de implementar el parsing del feed ATOM de Catastro en Python. El código hace [describe brevemente]. Los problemas que encontré fueron [describe]. Los resolví [describe]. Redacta una entrada para mi DEVLOG.md con las secciones: Qué implementé, Problemas encontrados, Cómo los resolví, Lo que aprendí, Próximo paso."*

---

## Fase 3 — Script Python: descargar y descomprimir datos

> **Objetivo**: Script que descargue el ZIP de un municipio y lo descomprima  
> **Tiempo estimado**: 2-3 horas  
> **Archivo**: `core/downloader.py`  
> **Herramienta**: VS Code + Gemini Code Assist

### 3.1 Prompt para Gemini Code Assist

> *"Write a Python function called `download_municipality_data(download_url, municipality_code, output_dir)` that:*
> *1. Downloads a ZIP file from `download_url` using `requests` with streaming*
> *2. Shows download progress (percentage)*
> *3. Saves the ZIP to a temp file inside `output_dir`*
> *4. Extracts all files to a subfolder named `municipality_code` inside `output_dir`*
> *5. Returns a list of paths to the extracted files*
> *Follow PEP8. Add docstring. Handle errors: network failure, disk space, corrupted ZIP."*

### 3.2 Prompt para gestión de errores

> *"Review this downloader function and add specific error messages in Spanish that a non-technical user could understand. For example, instead of raising a generic exception, show 'No se pudo conectar con el servidor de Catastro. Comprueba tu conexión a internet.'"*

### 3.3 Actualizar DEVLOG

> *"Acabo de implementar la descarga y descompresión de datos Catastro. [Describe lo que hiciste y los problemas]. Redacta la entrada del DEVLOG."*

---

## Fase 4 — Script PyQGIS: cargar capas en QGIS

> **Objetivo**: Cargar los archivos GML/GeoPackage descargados como capas en QGIS  
> **Tiempo estimado**: 2-4 horas  
> **Archivo**: `core/qgis_loader.py`  
> **Herramienta**: QGIS Python Console + Claude.ai (mejor para PyQGIS)

> ⚠️ **Nota**: Para esta fase usa Claude.ai en lugar de Gemini. PyQGIS es muy específico y Claude razona mejor los errores de integración.

### 4.1 Prompt para Claude

> *"Write a PyQGIS function called `load_layers_in_qgis(file_paths, municipality_name)` that:*
> *1. Receives a list of file paths (GML or GeoPackage files extracted from Catastro ZIP)*
> *2. For each file, creates a QgsVectorLayer*
> *3. Adds each layer to the current QGIS project with `QgsProject.instance().addMapLayer()`*
> *4. Groups all loaded layers inside a QgsLayerTree group named `municipality_name`*
> *5. Returns the list of successfully loaded layers*
> *Add docstring. Handle the case where a file cannot be loaded as a valid vector layer."*

### 4.2 Prompt para testear

> *"Give me a minimal test script for QGIS Python Console that calls `load_layers_in_qgis()` with a hardcoded file path to a GML file I already have on disk, so I can verify it loads correctly."*

### 4.3 Crear archivo de instrucciones para el agente

Crear `.github/copilot-instructions.md` con el prompt:

> *"I'm building a QGIS plugin. Here is my project structure and conventions. Generate a copilot-instructions.md file that describes: folder structure, coding conventions (PEP8, English names, docstrings), the three core modules (atom_parser, downloader, qgis_loader) and their responsibilities, and the rule that UI files must contain no business logic."*

---

## Fase 5 — Estructura del plugin QGIS

> **Objetivo**: Crear el esqueleto mínimo de un plugin QGIS funcional  
> **Tiempo estimado**: 2-3 horas  
> **Herramienta**: Plugin Builder de QGIS + Claude.ai

### 5.1 Generar esqueleto con Plugin Builder

- En QGIS: Complementos → Plugin Builder → Plugin Builder
- Rellenar: nombre `CatastroDownloader`, clase `CatastroDownloader`, módulo `catastro_downloader`
- Marcar: *Dialog with buttons*
- Copiar los archivos generados a la carpeta del repositorio

### 5.2 Prompt para Claude

> *"Here is the folder structure of a QGIS plugin generated with Plugin Builder [paste structure]. I already have three working modules: atom_parser.py, downloader.py, and qgis_loader.py. Explain me: 1) which files I need to edit to wire my modules into the plugin, 2) what `__init__.py` must contain, 3) what `metadata.txt` fields are mandatory. Be concise."*

### 5.3 Instalar el plugin en QGIS para desarrollo

> *"Explain how to install a QGIS plugin from a local folder for development, so changes are reflected without reinstalling. Which folder should I copy my plugin to on Windows/Mac/Linux?"*

---

## Fase 6 — Interfaz de usuario (UI)

> **Objetivo**: Crear el diálogo con selector de provincia, municipio y tipo de datos  
> **Tiempo estimado**: 3-5 horas  
> **Archivo**: `ui/main_dialog.py`  
> **Herramienta**: VS Code + Gemini Code Assist

### 6.1 Prompt para Gemini Code Assist

> *"Write a PyQGIS dialog class called `CatastroDownloaderDialog` (inheriting from QDialog) with:*
> *- A QComboBox `province_combo` populated with Spanish province names*
> *- A QComboBox `municipality_combo` that updates dynamically when province changes*
> *- A QGroupBox with three QCheckBoxes: `cb_parcels`, `cb_addresses`, `cb_buildings`*
> *- A QLineEdit `output_dir_input` with a QPushButton to open a folder picker dialog*
> *- A QPushButton `download_btn` labeled 'Descargar y cargar en QGIS'*
> *- A QProgressBar `progress_bar`*
> *- A QLabel `status_label` for status messages*
> *No business logic in this file. Only UI definition and signal connections to slots defined in the main plugin class. PEP8. Docstrings."*

### 6.2 Prompt para conectar UI con lógica

> *"Now write the slot methods for the main plugin class that connect the dialog UI to the three core modules (atom_parser, downloader, qgis_loader). The download button should: 1) get selected municipality and checked layer types, 2) call downloader for each checked type, 3) update the progress bar, 4) call qgis_loader when download finishes, 5) show success or error message in status_label."*

---

## Fase 7 — Pruebas y corrección de errores

> **Objetivo**: Probar el plugin completo con municipios reales  
> **Tiempo estimado**: 2-4 horas (variable según errores)  
> **Herramienta**: QGIS + Claude.ai para debuggear

### 7.1 Municipios de prueba recomendados

Empezar con municipios pequeños para descargas rápidas. Buscar en el feed ATOM municipios con pocos datos.

### 7.2 Prompt para debuggear errores

> *"I'm getting this error in my QGIS plugin: [pega el traceback completo]. Here is the relevant code: [pega el código]. Explain what is causing the error and give me the corrected code."*

### 7.3 Prompt para revisión de calidad

> *"Review this complete module [paste code]. Identify: 1) any PEP8 violations, 2) missing error handling, 3) functions that are too long and should be split, 4) any hardcoded values that should be constants. Give me the improved version."*

---

## Fase 8 — Documentación final

> **Objetivo**: Documentar el plugin para usuarios y para la comunidad  
> **Tiempo estimado**: 2-3 horas  
> **Herramienta**: Claude.ai

### 8.1 Prompt para README

> *"Based on this QGIS plugin code [paste main files], write a README.md in Spanish with these sections: Descripción, Requisitos, Instalación, Uso paso a paso (con capturas de pantalla pendientes), Fuente de datos (Catastro INSPIRE), Licencia, Autor. Use clear language for non-technical GIS users."*

### 8.2 Prompt para metadata.txt

> *"Complete this QGIS plugin metadata.txt file [paste current file] with all fields required to publish in the official QGIS plugin repository. Explain what each field means."*

### 8.3 Prompt para DEVLOG final

> *"Aquí está el historial completo de mi DEVLOG.md [pega el contenido]. Redacta un resumen final del proyecto: qué construí, qué aprendí, qué haría diferente, y qué mejoras futuras podría añadir al plugin."*

---

## Fase 9 — Publicación (opcional)

> **Objetivo**: Publicar el plugin en el repositorio oficial de QGIS  
> **Tiempo estimado**: 1-2 horas

- [ ] Revisar que `metadata.txt` está completo
- [ ] Crear un tag de versión en GitHub (`v1.0.0`)
- [ ] Generar el ZIP del plugin
- [ ] Registrarse en https://plugins.qgis.org
- [ ] Subir el ZIP

### Prompt para preparar la publicación

> *"My QGIS plugin is ready to publish. Here is my metadata.txt [paste]. Check if all mandatory fields for the QGIS plugin repository are present and correctly formatted. What ZIP structure does the repository expect?"*

---

## Referencia rápida de prompts

| Situación | Herramienta | Tipo de prompt |
|---|---|---|
| Generar código nuevo | Gemini Code Assist (VS Code) | Prompt detallado con requisitos técnicos |
| Debuggear error de PyQGIS | Claude.ai | Pega traceback + código relevante |
| Entender por qué funciona algo | Claude.ai | "Explícame línea a línea qué hace este código" |
| Revisar calidad del código | Gemini o Claude | "Review for PEP8, error handling, readability" |
| Redactar DEVLOG | Claude.ai | Describe en bruto lo que hiciste |
| Generar documentación | Claude.ai | Pega el código + pide el documento |

---

## Convenciones de commits Git

Usar siempre este formato:

```
feat: descripción de la nueva funcionalidad
fix: descripción del error corregido
docs: actualización de documentación
refactor: mejora de código sin cambiar funcionalidad
test: añadir o modificar tests
```

Ejemplos:
```
feat: parse ATOM feed and extract municipality list
fix: handle missing download URL in province sub-feed
docs: update README with installation instructions
```

---

*Documento generado como guía de aprendizaje y desarrollo. Actualizar el DEVLOG.md al finalizar cada fase.*
