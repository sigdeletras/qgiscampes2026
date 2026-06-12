# Plan de trabajo: Plugin QGIS — Catastro INSPIRE Downloader

> **Objetivo:** Desarrollar un complemento QGIS que permita descargar datos INSPIRE de Catastro
> (parcelas, direcciones, edificios) por municipio y cargarlos como capas en QGIS.
>
> **Perfil:** Usuario QGIS con conocimientos básicos de Python y experiencia en desarrollo web.
> **Entorno:** VS Code + Claude (Claude.ai o Claude Code extension).
> **Versión objetivo:** QGIS 3.28 LTR o superior / Python 3.9+

---

## Índice

- [Fase 0 — Configuración del entorno](#fase-0--configuración-del-entorno)
- [Fase 1 — Exploración del territorio](#fase-1--exploración-del-territorio)
- [Fase 2 — Esqueleto del plugin](#fase-2--esqueleto-del-plugin)
- [Fase 3 — Módulo de parseo ATOM](#fase-3--módulo-de-parseo-atom)
- [Fase 4 — Módulo de descarga y extracción](#fase-4--módulo-de-descarga-y-extracción)
- [Fase 5 — Módulo de carga de capas](#fase-5--módulo-de-carga-de-capas)
- [Fase 6 — Interfaz de usuario](#fase-6--interfaz-de-usuario)
- [Fase 7 — Integración y pruebas](#fase-7--integración-y-pruebas)
- [Fase 8 — Documentación final](#fase-8--documentación-final)
- [Referencia de prompts](#referencia-de-prompts)

---

## Fase 0 — Configuración del entorno

**Objetivo:** Tener VS Code listo para desarrollar y depurar el plugin antes de escribir
una sola línea de código del proyecto.

**Tiempo estimado:** 2–3 horas

### 0.1 Instalaciones base

- [ ] VS Code actualizado (≥ 1.85)
- [ ] Python 3.9+ disponible en el sistema
- [ ] QGIS 3.28 LTR instalado
- [ ] Git instalado y configurado (`git config --global user.name` y `user.email`)
- [ ] Node.js 18+ (necesario para Claude Code CLI si se usa)

### 0.2 Extensiones VS Code

Instalar en este orden (`Ctrl+Shift+X`):

| Extensión | ID | Para qué |
|---|---|---|
| Python | `ms-python.python` | Base imprescindible |
| Pylance | `ms-python.vscode-pylance` | Autocompletado e IntelliSense |
| Python Debugger | `ms-python.debugpy` | Breakpoints y depuración |
| Ruff | `charliermarsh.ruff` | Linting y formateo automático |
| autoDocstring | `njpwerner.autodocstring` | Esqueletos de docstring automáticos |
| Error Lens | `usernamehw.errorlens` | Errores inline en el código |
| Git Graph | `mhutchie.git-graph` | Historial visual de Git |
| XML | `redhat.vscode-xml` | Inspección de feeds ATOM |
| indent-rainbow | `oderwat.indent-rainbow` | Visualizar indentación Python |
| Claude Code | `anthropic.claude-code` | IA integrada (requiere plan Pro) |

### 0.3 Configuración `settings.json`

Abrir con `Ctrl+Shift+P` → *"Open User Settings (JSON)"* y añadir:

```json
{
  "python.analysis.typeCheckingMode": "basic",
  "python.analysis.autoImportCompletions": true,
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit"
    }
  },
  "editor.rulers": [88],
  "editor.bracketPairColorization.enabled": true,
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true
}
```

### 0.4 Intérprete Python de QGIS

- [ ] Localizar el ejecutable Python de QGIS:
  - Windows: `C:\Program Files\QGIS 3.28\apps\Python39\python.exe`
  - Linux/Mac: `$(which python3)` en el entorno de QGIS
- [ ] En VS Code: `Ctrl+Shift+P` → *"Python: Select Interpreter"* → seleccionar el de QGIS
- [ ] Verificar que Pylance resuelve `import qgis.core` sin errores

### 0.5 Plugins QGIS de desarrollo

Instalar desde el gestor de complementos de QGIS:

- [ ] **Plugin Builder 3** — genera el esqueleto inicial del plugin
- [ ] **Plugin Reloader** — recarga el plugin sin reiniciar QGIS (atajo: `Ctrl+F5`)

### 0.6 Repositorio Git

```bash
mkdir catastro-inspire-downloader
cd catastro-inspire-downloader
git init
```

Crear `.gitignore` inicial:

```
__pycache__/
*.pyc
*.zip
*.gml
temp/
.vscode/settings.json
```

### 0.7 Fichero `CLAUDE.md` (contexto para el agente)

Crear en la raíz del proyecto. Este fichero lo leerá Claude Code automáticamente
al abrir el proyecto:

```markdown
# Contexto del proyecto

Plugin QGIS para descarga de datos INSPIRE de Catastro español.
Target: QGIS 3.28 LTR+ / Python 3.9+
Dependencias: solo librería estándar Python (sin requests, sin lxml)

## Estructura de módulos
- plugin.py        — punto de entrada QGIS, solo inicialización
- dialog.py        — UI con PyQt5, sin lógica de negocio
- atom_parser.py   — parseo de feeds ATOM (xml.etree), sin dependencias QGIS
- downloader.py    — descarga HTTP (urllib) y extracción ZIP (zipfile)
- layer_loader.py  — carga de capas con iface.addVectorLayer()

## Convenciones
- Docstrings en español, formato Google style
- Type hints en todas las funciones públicas
- Manejo explícito de excepciones, nunca bare except
- Mensajes al usuario via QgsMessageBar, no print()
- Rutas temporales via tempfile.mkdtemp()

## API PyQGIS frecuente
- Cargar capa: iface.addVectorLayer(path, name, "ogr")
- Mensaje: iface.messageBar().pushMessage("Catastro", msg, level=Qgis.Info)
- Ruta temp: tempfile.mkdtemp()
```

### 0.8 Fichero `ruff.toml`

```toml
line-length = 88
target-version = "py39"

[lint]
select = ["E", "F", "W", "I"]
ignore = ["E501"]
```

### ✅ Criterio de completitud de Fase 0

- VS Code formatea automáticamente al guardar un fichero `.py`
- Pylance no marca error en `from qgis.core import QgsVectorLayer`
- Plugin Reloader está disponible en la barra de QGIS
- El repositorio tiene al menos un commit inicial

---

## Fase 1 — Exploración del territorio

**Objetivo:** Entender la estructura real de los feeds ATOM y los ficheros descargables
*antes* de escribir código. Esta fase no produce código, produce conocimiento.

**Tiempo estimado:** 1–2 horas

### 1.1 Inspección manual de los feeds

Abrir en el navegador (o con la extensión XML de VS Code):

| Feed | URL |
|---|---|
| Parcelas | `https://www.catastro.hacienda.gob.es/INSPIRE/CadastralParcels/ES.SDGC.CP.atom.xml` |
| Direcciones | `https://www.catastro.hacienda.gob.es/INSPIRE/Addresses/ES.SDGC.AD.atom.xml` |
| Edificios | `https://www.catastro.hacienda.gob.es/INSPIRE/buildings/ES.SDGC.BU.atom.xml` |

Anotar en `DEVLOG.md`:
- ¿Qué elementos XML contiene una entrada (`<entry>`)?
- ¿Dónde está la URL del sub-feed provincial?
- ¿Cómo se llaman los namespaces declarados?

### 1.2 Inspección del sub-feed provincial

- [ ] Tomar la URL de una provincia del feed raíz y abrirla
- [ ] Localizar las entradas de municipio: ¿qué campo contiene el nombre? ¿y la URL de descarga?

### 1.3 Descarga manual de un municipio

- [ ] Descargar manualmente el ZIP de un municipio pequeño
- [ ] Extraerlo y anotar: ¿cuántos ficheros GML contiene? ¿cómo se llaman?
- [ ] Cargar el GML manualmente en QGIS (`Capa → Añadir capa vectorial`)
- [ ] Verificar que se carga correctamente y observar la estructura de atributos

### 1.4 Registro en DEVLOG

Crear `DEVLOG.md` y añadir la primera entrada:

```markdown
## [Fecha] — Exploración de feeds ATOM

**Estructura del feed raíz:**
[Anotar aquí lo observado]

**Estructura del sub-feed provincial:**
[Anotar aquí lo observado]

**Contenido del ZIP descargado:**
[Lista de ficheros encontrados]

**Preguntas abiertas:**
[Lo que no quedó claro]
```

### 🤖 Prompts para esta fase

**Prompt 1.A — Entender el XML observado**

```
Estoy construyendo un plugin QGIS para descargar datos INSPIRE de Catastro español.
He inspeccionado el feed ATOM raíz y este es un fragmento del XML que he encontrado:

[PEGAR AQUÍ 20-30 LÍNEAS DEL XML]

Explícame:
1. Qué significa cada elemento relevante
2. Cómo llego desde este feed hasta la URL de descarga de un municipio concreto
3. Qué namespaces necesito declarar en Python para parsear esto con xml.etree.ElementTree
```

**Prompt 1.B — Aclarar estructura de ficheros descargados**

```
He descargado manualmente el ZIP de datos INSPIRE de Catastro para un municipio.
Dentro encontré estos ficheros: [LISTAR FICHEROS].

Explícame:
1. Qué contiene cada fichero GML
2. Cuál debo cargar en QGIS para obtener las parcelas catastrales
3. Si hay algún fichero auxiliar necesario o puedo ignorar el resto
```

---

## Fase 2 — Esqueleto del plugin

**Objetivo:** Crear la estructura mínima de ficheros que QGIS necesita para reconocer
y cargar el complemento, sin lógica funcional aún.

**Tiempo estimado:** 2–3 horas

### 2.1 Generar con Plugin Builder

- [ ] En QGIS: `Complementos → Plugin Builder → Plugin Builder 3`
- [ ] Rellenar:
  - Class name: `CatastroInspireDownloader`
  - Plugin name: `Catastro INSPIRE Downloader`
  - Module name: `catastro_inspire`
  - Minimum QGIS version: `3.28`
  - Description: `Descarga datos INSPIRE de Catastro por municipio`
- [ ] Generar en la carpeta del repositorio Git

### 2.2 Revisar y limpiar el esqueleto

Plugin Builder genera más de lo necesario. Reducir a:

```
catastro_inspire/
├── __init__.py
├── metadata.txt
├── plugin.py
├── dialog.py
├── dialog_base.ui        ← diseñado en Qt Designer
├── atom_parser.py        ← crear vacío
├── downloader.py         ← crear vacío
├── layer_loader.py       ← crear vacío
├── resources.py          ← generado por Plugin Builder
├── CLAUDE.md
├── DEVLOG.md
├── README.md
└── ruff.toml
```

### 2.3 Verificar que QGIS carga el plugin

- [ ] Copiar/enlazar la carpeta al directorio de plugins de QGIS:
  - Windows: `%APPDATA%\QGIS\QGIS3\profiles\default\python\plugins\`
  - Linux: `~/.local/share/QGIS/QGIS3/profiles/default/python/plugins/`
- [ ] En QGIS: `Complementos → Administrar e instalar → Instalados`
- [ ] Verificar que aparece sin errores y se puede activar
- [ ] Hacer commit: `"feat: esqueleto inicial generado con Plugin Builder"`

### 🤖 Prompts para esta fase

**Prompt 2.A — Revisar metadata.txt**

```
Soy principiante en desarrollo de plugins QGIS. Plugin Builder me ha generado
este fichero metadata.txt:

[PEGAR CONTENIDO DEL metadata.txt]

Revísalo y dime:
1. Qué campos son obligatorios y cuáles opcionales
2. Qué valores debería cambiar para un plugin de uso personal/educativo
3. Si hay algo incorrecto o que pueda causar problemas al cargar el plugin
```

**Prompt 2.B — Entender el esqueleto generado**

```
Plugin Builder me ha generado un plugin QGIS con esta estructura:

[PEGAR LISTADO DE FICHEROS Y CONTENIDO DE plugin.py]

Soy desarrollador web con Python básico. Explícame:
1. Qué hace el método initGui() y por qué es necesario
2. Qué hace unload() y cuándo se llama
3. Cuál es el flujo de ejecución desde que el usuario hace clic en el botón del plugin
   hasta que se abre el diálogo
No generes código todavía, solo explícame la arquitectura.
```

**Prompt 2.C — Crear módulos vacíos con estructura correcta**

```
Voy a crear tres módulos Python vacíos para mi plugin QGIS:
- atom_parser.py: parseará feeds ATOM de Catastro con xml.etree.ElementTree
- downloader.py: descargará ZIPs con urllib y los extraerá con zipfile
- layer_loader.py: cargará ficheros GML en QGIS con iface.addVectorLayer()

Para cada módulo, genera:
1. El docstring de módulo explicando su responsabilidad
2. Los imports que necesitará (solo librería estándar + qgis donde sea necesario)
3. Un comentario indicando las funciones públicas que implementará
No implementes las funciones todavía.

Contexto del proyecto: QGIS 3.28+, Python 3.9, sin dependencias externas (solo stdlib).
```

---

## Fase 3 — Módulo de parseo ATOM

**Objetivo:** Implementar `atom_parser.py` — la lógica de parseo de los feeds ATOM para
obtener la lista de provincias y municipios disponibles.

**Tiempo estimado:** 3–5 horas

### 3.1 Funciones a implementar

```python
# atom_parser.py

def get_province_entries(feed_url: str) -> list[dict]:
    """Parsea el feed raíz y devuelve lista de provincias con su URL de sub-feed."""

def get_municipality_entries(province_feed_url: str) -> list[dict]:
    """Parsea el sub-feed provincial y devuelve lista de municipios con URL de descarga."""
```

### 3.2 Validar en consola Python de QGIS

Antes de integrar en el plugin, probar el módulo directamente
en `Complementos → Consola Python de QGIS`:

```python
import importlib.util, sys
spec = importlib.util.spec_from_file_location("atom_parser", "/ruta/al/atom_parser.py")
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)
provincias = mod.get_province_entries("https://...")
print(provincias[:3])
```

### 🤖 Prompts para esta fase

**Prompt 3.A — Implementar get_province_entries**

```
Voy a implementar la función get_province_entries en atom_parser.py de mi plugin QGIS.

El feed ATOM raíz de Catastro tiene esta estructura (fragmento real que he inspeccionado):
[PEGAR 30-40 LÍNEAS DEL XML DEL FEED RAÍZ]

Requisitos:
- Solo usar xml.etree.ElementTree (sin lxml, sin requests)
- La función recibe la URL del feed y devuelve una lista de dicts con 'name' y 'feed_url'
- Manejar explícitamente: timeout de 30s, error HTTP, feed vacío
- Docstring en español formato Google con Args, Returns y Raises
- Type hints en la firma

Antes de escribir el código, explícame qué namespace necesito declarar
y cómo navegaré el árbol XML para llegar a los datos que necesito.
```

**Prompt 3.B — Implementar get_municipality_entries**

```
Ahora necesito implementar get_municipality_entries en atom_parser.py.

El sub-feed provincial tiene esta estructura (fragmento real):
[PEGAR FRAGMENTO DEL SUB-FEED PROVINCIAL]

Mismos requisitos que la función anterior. Además:
- El dict de cada municipio debe incluir 'name' y 'download_url'
- Hay municipios sin datos disponibles (entradas sin link de descarga): omitirlos silenciosamente

Explícame primero cómo distingo una entrada con datos de una sin datos en el XML,
luego escribe el código.
```

**Prompt 3.C — Code review del módulo**

```
Este es el código completo de atom_parser.py que hemos desarrollado:

[PEGAR CÓDIGO COMPLETO]

Revísalo como si fuera un code review para un desarrollador junior. Señala:
1. Funciones demasiado largas o con demasiada responsabilidad
2. Casos de error no manejados que podrían ocurrir con los feeds reales de Catastro
3. Cualquier cosa que un desarrollador nuevo no entendería sin contexto adicional
4. Si los docstrings describen con precisión el comportamiento real

No reescribas el código completo: dame una lista priorizada de mejoras.
```

---

## Fase 4 — Módulo de descarga y extracción

**Objetivo:** Implementar `downloader.py` — descarga del ZIP desde la URL del municipio
y extracción del GML.

**Tiempo estimado:** 2–4 horas

### 4.1 Funciones a implementar

```python
# downloader.py

def download_zip(url: str, dest_dir: str) -> str:
    """Descarga el ZIP en dest_dir y devuelve la ruta local del fichero."""

def extract_gml(zip_path: str, dest_dir: str) -> str:
    """Extrae el ZIP y devuelve la ruta del fichero GML principal."""

def download_and_extract(url: str) -> str:
    """Función de conveniencia: descarga y extrae, devuelve ruta del GML."""
```

### 4.2 Consideraciones importantes

- Usar `tempfile.mkdtemp()` para el directorio temporal
- El ZIP puede contener múltiples GML: identificar el principal
- Algunos feeds requieren `User-Agent` en la cabecera HTTP para no recibir 403
- Limpiar ficheros temporales si la operación falla

### 🤖 Prompts para esta fase

**Prompt 4.A — Implementar descarga con urllib**

```
Necesito implementar download_zip en downloader.py para mi plugin QGIS.

La función debe:
- Descargar un ZIP desde una URL de Catastro usando solo urllib (sin requests)
- Guardar el ZIP en un directorio temporal (tempfile.mkdtemp)
- Incluir cabecera User-Agent para evitar errores 403
- Manejar: timeout 60s, errores HTTP, disco lleno
- Mostrar el progreso es opcional en esta versión

Contexto: este código correrá dentro de QGIS 3.28, Python 3.9.
Escribe primero los casos de error que hay que cubrir, luego el código.
```

**Prompt 4.B — Implementar extracción del GML**

```
Tengo esta función download_zip funcionando:
[PEGAR CÓDIGO]

Ahora necesito extract_gml que:
- Recibe la ruta del ZIP descargado
- Lo extrae con zipfile en el mismo directorio temporal
- Identifica y devuelve la ruta del GML principal (puede haber varios ficheros)
- El GML principal de Catastro suele tener un patrón de nombre concreto: [INDICAR EL QUE ENCONTRASTE EN FASE 1]

¿Cómo identifico el GML correcto si hay varios? Dame tu razonamiento antes del código.
```

**Prompt 4.C — Aprender del código generado**

```
Has generado este código para downloader.py:
[PEGAR CÓDIGO COMPLETO]

Explícamelo línea a línea como si fuera un tutorial, especialmente:
1. Por qué urllib.request.urlopen en lugar de otras alternativas
2. Cómo funciona el context manager 'with' en este contexto
3. Por qué comprobamos el Content-Type antes de guardar
4. Qué hace exactamente zipfile.ZipFile en modo lectura

Quiero entenderlo completamente antes de integrarlo en el plugin.
```

---

## Fase 5 — Módulo de carga de capas

**Objetivo:** Implementar `layer_loader.py` — cargar el fichero GML como capa vectorial
en el proyecto QGIS activo.

**Tiempo estimado:** 1–2 horas

### 5.1 Funciones a implementar

```python
# layer_loader.py

def load_gml_layer(gml_path: str, layer_name: str) -> QgsVectorLayer:
    """Carga un fichero GML como capa vectorial en el proyecto QGIS actual."""
```

### 5.2 Puntos críticos de la API PyQGIS

```python
# Patrón correcto para cargar y verificar una capa
layer = QgsVectorLayer(gml_path, layer_name, "ogr")

# CRÍTICO: isValid() no lanza excepción, hay que comprobarlo explícitamente
if not layer.isValid():
    raise ValueError(f"QGIS no pudo cargar la capa: {gml_path}")

# addMapLayer registra la capa en el panel de capas
QgsProject.instance().addMapLayer(layer)
```

### 🤖 Prompts para esta fase

**Prompt 5.A — Implementar layer_loader**

```
Necesito implementar load_gml_layer en layer_loader.py para QGIS 3.28.

La función debe:
- Recibir la ruta al GML y un nombre para la capa
- Cargar la capa con QgsVectorLayer usando el proveedor "ogr"
- Verificar que la capa es válida (isValid()) antes de añadirla
- Añadirla al proyecto con QgsProject.instance().addMapLayer()
- Si la capa no es válida, lanzar una excepción descriptiva
- Devolver la capa cargada

Antes de escribir el código, explícame por qué QgsVectorLayer puede devolver
un objeto "inválido" sin lanzar excepción, y qué situaciones provocan eso.
```

**Prompt 5.B — Entender la API de QGIS**

```
En el código de layer_loader.py usamos QgsProject.instance().addMapLayer(layer).

Explícame:
1. Qué es QgsProject.instance() y por qué se usa el patrón singleton
2. Qué diferencia hay entre addMapLayer() y addVectorLayer() de iface
3. Si el usuario cierra el proyecto QGIS, ¿qué pasa con los ficheros temporales?
4. ¿Hay alguna forma de agrupar las capas cargadas en un grupo con nombre?

No necesito código ahora, solo la explicación para entender mejor la arquitectura de QGIS.
```

---

## Fase 6 — Interfaz de usuario

**Objetivo:** Implementar `dialog.py` — el diálogo con los controles para que el usuario
elija tipo de dato, provincia, municipio y lance la descarga.

**Tiempo estimado:** 4–6 horas

### 6.1 Controles necesarios

```
┌─────────────────────────────────────────┐
│  Catastro INSPIRE Downloader            │
├─────────────────────────────────────────┤
│  Tipo de datos:  [ComboBox ▼]           │
│    ○ Parcelas catastrales               │
│    ○ Direcciones                        │
│    ○ Edificios                          │
│                                         │
│  Provincia:      [ComboBox ▼]           │
│  Municipio:      [ComboBox ▼]           │
│                                         │
│  [Barra de progreso / estado]           │
│                                         │
│           [Cancelar]  [Descargar]       │
└─────────────────────────────────────────┘
```

### 6.2 Lógica del diálogo

- Al cambiar el tipo de dato: recargar provincias (poblar combo)
- Al cambiar provincia: cargar municipios en segundo plano (poblar combo)
- Al pulsar Descargar: llamar a `downloader` + `layer_loader`
- Deshabilitar el botón durante la descarga para evitar doble clic

### 🤖 Prompts para esta fase

**Prompt 6.A — Diseñar el diálogo**

```
Necesito implementar el diálogo principal de mi plugin QGIS en dialog.py.

El diálogo debe tener:
- QComboBox para tipo de dato (Parcelas / Direcciones / Edificios) — valores fijos
- QComboBox para provincia — se carga al arrancar el diálogo
- QComboBox para municipio — se carga al seleccionar provincia
- QLabel para mostrar estado ("Cargando...", "Descargando...", "Listo")
- QPushButton "Descargar" — deshabilitado hasta que haya municipio seleccionado
- QPushButton "Cerrar"

Tengo estos módulos ya implementados:
- atom_parser.get_province_entries(feed_url) → list[dict] con 'name' y 'feed_url'
- atom_parser.get_municipality_entries(feed_url) → list[dict] con 'name' y 'download_url'
- downloader.download_and_extract(url) → str (ruta al GML)
- layer_loader.load_gml_layer(path, name) → QgsVectorLayer

Genera dialog.py con:
1. La clase CatastroDialog que hereda de QDialog
2. Los métodos de inicialización de la UI (sin fichero .ui, solo código Python)
3. Los slots conectados a cada señal
4. Manejo de errores visible al usuario via QMessageBox

Importante: las llamadas a atom_parser y downloader bloquearán la UI en esta versión.
Acepto ese compromiso para el MVP. Añade un comentario TODO donde habría que
implementar QgsTask en el futuro.
```

**Prompt 6.B — Entender señales y slots de Qt**

```
En dialog.py conectamos señales y slots de Qt. Por ejemplo:
self.comboProvince.currentIndexChanged.connect(self._on_province_changed)

Vengo de desarrollo web. Explícame la analogía exacta con eventos JavaScript:
1. ¿Qué es una señal en Qt y qué equivale en el DOM?
2. ¿Qué es un slot y cómo se diferencia de un addEventListener callback?
3. ¿Por qué se llama disconnect() en el destructor y qué pasa si no se hace?
4. ¿currentIndexChanged pasa el índice o el valor al slot?
```

**Prompt 6.C — Depurar la UI**

```
Mi diálogo de QGIS muestra este comportamiento inesperado:
[DESCRIBIR EL PROBLEMA CON DETALLE]

Este es el código relevante de dialog.py:
[PEGAR EL CÓDIGO]

Y este es el error/log que veo en la Consola Python de QGIS:
[PEGAR EL TRACEBACK O MENSAJE]

Analiza el problema y propón una solución. Explícame por qué ocurre,
no solo cómo arreglarlo.
```

---

## Fase 7 — Integración y pruebas

**Objetivo:** Conectar todos los módulos, probar el flujo completo y manejar los casos
de error más comunes.

**Tiempo estimado:** 3–5 horas

### 7.1 Checklist de pruebas manuales

- [ ] Municipio con datos disponibles → capa se carga correctamente
- [ ] Municipio sin datos → mensaje de error amigable (no crash)
- [ ] Sin conexión a internet → mensaje de error amigable
- [ ] Nombre de municipio con tildes (Málaga, Cádiz, etc.) → funciona
- [ ] Cargar tres tipos de dato del mismo municipio → tres capas separadas en QGIS
- [ ] Cerrar el diálogo durante la descarga → no deja ficheros temporales huérfanos
- [ ] Recargar el plugin con Plugin Reloader → no error

### 7.2 Casos de error conocidos de los feeds de Catastro

| Situación | Respuesta esperada |
|---|---|
| Feed devuelve 403 | Mensaje: "Error de acceso al servidor. Inténtalo de nuevo." |
| Municipio sin datos INSPIRE | Mensaje: "No hay datos disponibles para este municipio." |
| GML no cargable en QGIS | Mensaje: "El fichero descargado no es válido." |
| Timeout de red | Mensaje: "Tiempo de espera agotado. Comprueba tu conexión." |

### 🤖 Prompts para esta fase

**Prompt 7.A — Revisar manejo de errores end-to-end**

```
Este es el flujo completo de mi plugin de Catastro, desde el clic en "Descargar"
hasta la capa cargada en QGIS:

[PEGAR CÓDIGO RELEVANTE DE dialog.py, downloader.py Y layer_loader.py]

Identifica todos los puntos donde puede fallar y que actualmente no están cubiertos.
Para cada uno:
1. ¿Qué excepción o comportamiento inesperado puede ocurrir?
2. ¿Qué debería ver el usuario?
3. ¿Cómo deberíamos capturarlo?

Prioriza los errores más probables con los feeds reales de Catastro.
```

**Prompt 7.B — Añadir entrada al DEVLOG tras integración**

```
Acabo de completar la integración del plugin de Catastro. Los módulos que conecté son:
atom_parser → downloader → layer_loader → dialog

Los problemas que encontré durante la integración fueron:
[DESCRIBIR PROBLEMAS REALES ENCONTRADOS]

Ayúdame a redactar una entrada de DEVLOG.md de 10-15 líneas que documente:
1. Qué decisiones de integración tomé y por qué
2. Los problemas encontrados y cómo los resolví
3. Qué aprendí que no sabía al empezar esta fase
```

**Prompt 7.C — Generar casos de prueba manual**

```
Mi plugin QGIS de Catastro ya funciona en el caso feliz. Quiero probarlo
de forma más exhaustiva antes de darlo por terminado.

Genera una lista de 10 casos de prueba manual (sin código de testing automático)
que cubra los escenarios más importantes, incluyendo casos de error.
Para cada caso, indica: precondición, acción del usuario y resultado esperado.
```

---

## Fase 8 — Documentación final

**Objetivo:** Producir la documentación mínima para que el plugin sea comprensible y
mantenible: README completo, docstrings revisados y guía de usuario básica.

**Tiempo estimado:** 2–3 horas

### 8.1 README.md final

Estructura sugerida:

```markdown
# Catastro INSPIRE Downloader — Plugin QGIS

## Descripción
## Requisitos
## Instalación
## Uso paso a paso
## Arquitectura técnica
## Decisiones técnicas y por qué
## Problemas conocidos y limitaciones
## Registro de aprendizaje (resumen del DEVLOG)
## Licencia
```

### 8.2 Revisión final de docstrings

Recorrer todos los módulos y verificar que cada función pública tiene docstring
con Args, Returns y Raises correctos y actualizados.

### 🤖 Prompts para esta fase

**Prompt 8.A — Generar README completo**

```
Mi plugin QGIS de Catastro está terminado. Aquí está el código de los módulos principales:

atom_parser.py: [PEGAR]
downloader.py: [PEGAR]
layer_loader.py: [PEGAR]
dialog.py: [PEGAR]

Y este es mi DEVLOG.md con el historial de desarrollo: [PEGAR]

Genera un README.md completo en español que incluya:
1. Descripción clara del plugin para alguien que no sabe programar
2. Requisitos (versión QGIS, SO compatibles)
3. Instrucciones de instalación paso a paso
4. Guía de uso con pasos numerados
5. Sección de arquitectura técnica (para desarrolladores) en 1-2 párrafos
6. Limitaciones conocidas (menciona que la descarga bloquea la UI, que no hay caché, etc.)

Tono: claro, directo, sin jerga innecesaria.
```

**Prompt 8.B — Auditoría final de docstrings**

```
Estos son todos los docstrings actuales de mi plugin:

[PEGAR TODAS LAS FIRMAS DE FUNCIÓN CON SUS DOCSTRINGS]

Para cada docstring, dime si:
1. Describe con precisión el comportamiento real de la función
2. Los tipos en Args y Returns coinciden con los type hints
3. Los Raises mencionan todas las excepciones que el código puede lanzar
4. Hay algo ambiguo que un desarrollador nuevo malinterpretaría

Dame el resultado como tabla: función | estado | mejora sugerida.
```

**Prompt 8.C — Reflexión final de aprendizaje**

```
He terminado mi primer plugin QGIS. A lo largo del proceso desarrollé estos módulos:
[LISTAR MÓDULOS Y SU FUNCIÓN]

Y documenté estos aprendizajes en el DEVLOG: [PEGAR DEVLOG COMPLETO]

Ayúdame a escribir una reflexión de 3-4 párrafos para añadir al README bajo el título
"Qué aprendí desarrollando este plugin". Que sea honesta, técnica pero accesible,
y que mencione específicamente qué aportó la IA y dónde tuve que verificar yo mismo.
```

---

## Referencia de prompts

Índice rápido de todos los prompts del plan:

| ID | Fase | Objetivo |
|---|---|---|
| 1.A | Exploración | Entender estructura XML del feed ATOM |
| 1.B | Exploración | Entender ficheros descargados (ZIP/GML) |
| 2.A | Esqueleto | Revisar metadata.txt generado |
| 2.B | Esqueleto | Entender arquitectura del esqueleto |
| 2.C | Esqueleto | Crear módulos vacíos con estructura correcta |
| 3.A | ATOM | Implementar get_province_entries |
| 3.B | ATOM | Implementar get_municipality_entries |
| 3.C | ATOM | Code review del módulo completo |
| 4.A | Descarga | Implementar descarga con urllib |
| 4.B | Descarga | Implementar extracción del GML |
| 4.C | Descarga | Aprender del código generado |
| 5.A | Capas | Implementar load_gml_layer |
| 5.B | Capas | Entender la API de QGIS |
| 6.A | UI | Diseñar e implementar el diálogo |
| 6.B | UI | Entender señales y slots de Qt |
| 6.C | UI | Depurar comportamiento inesperado |
| 7.A | Integración | Revisar manejo de errores end-to-end |
| 7.B | Integración | Documentar integración en DEVLOG |
| 7.C | Integración | Generar casos de prueba manual |
| 8.A | Documentación | Generar README completo |
| 8.B | Documentación | Auditoría final de docstrings |
| 8.C | Documentación | Reflexión final de aprendizaje |

---

## Notas generales sobre el uso de IA

### Regla de oro

> **No aceptes código que no puedas explicar en voz alta.**
> Si no puedes decir "esta función hace X porque Y", pide que te lo expliquen antes de continuar.

### Patrón de prompt más útil para aprender

```
Antes de escribir el código, explícame [concepto/decisión técnica].
Luego genera el código con docstrings en español y type hints.
Finalmente, señala qué partes debería verificar contra la documentación oficial de PyQGIS.
```

### Lo que la IA NO sabe bien

- Métodos deprecados o nuevos de la API QGIS 3.34+
- Comportamiento real de los feeds ATOM de Catastro (timeouts, campos ausentes)
- Errores específicos de tu entorno (versión de QGIS, SO, configuración)

**Siempre contrasta con:**
- [PyQGIS API Reference](https://qgis.org/pyqgis/master/)
- [PyQGIS Developer Cookbook](https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/)
- Inspección manual del XML real antes de asumir la estructura

---

*Documento generado como guía de desarrollo. Actualiza las secciones de DEVLOG
y README conforme avances en el proyecto.*
