# Plan de Trabajo: Plugin QGIS — Catastro INSPIRE Downloader

> **Propósito de este documento:** Guía de desarrollo agentico para VS Code + Claude Code.
> Claude debe leer este fichero al inicio de cada sesión y ejecutar las tareas de la fase activa de forma autónoma, creando y modificando ficheros reales sin pedir confirmación salvo en los puntos marcados con `⚠️ PAUSA`.
>
> **Objetivo del plugin:** Complemento QGIS que descarga datos INSPIRE de Catastro español
> (parcelas, direcciones, edificios) por municipio y los carga como capas vectoriales en QGIS.
>
> **Stack:** QGIS 3.28 LTR+ · Python 3.9+ · PyQt5 · xml.etree · urllib · zipfile
> (sin dependencias externas: ni `requests`, ni `lxml`, ni `numpy`)
>
> **Entorno de desarrollo:** Windows 11 · VS Code · Claude Code extension · Git

---

## Índice

- [CLAUDE.md — Reglas permanentes del agente](#claudemd--reglas-permanentes-del-agente)
- [Arquitectura objetivo](#arquitectura-objetivo)
- [Fase 0 — Configuración del entorno](#fase-0--configuración-del-entorno)
- [Fase 1 — Exploración de los feeds ATOM](#fase-1--exploración-de-los-feeds-atom)
- [Fase 2 — Esqueleto del plugin](#fase-2--esqueleto-del-plugin)
- [Fase 3 — Módulo de parseo ATOM](#fase-3--módulo-de-parseo-atom)
- [Fase 4 — Módulo de descarga y extracción](#fase-4--módulo-de-descarga-y-extracción)
- [Fase 5 — Módulo de carga de capas](#fase-5--módulo-de-carga-de-capas)
- [Fase 6 — Interfaz de usuario PyQt5](#fase-6--interfaz-de-usuario-pyqt5)
- [Fase 7 — Asincronía con QgsTask](#fase-7--asincronía-con-qgstask)
- [Fase 8 — Integración y pruebas](#fase-8--integración-y-pruebas)
- [Fase 9 — Empaquetado y publicación](#fase-9--empaquetado-y-publicación)
- [Fase 10 — Documentación final](#fase-10--documentación-final)
- [Referencia de recursos](#referencia-de-recursos)

---

## CLAUDE.md — Reglas permanentes del agente

> Este bloque define las reglas que Claude debe aplicar en **todas** las fases sin excepción.
> Al iniciar una sesión de trabajo, Claude debe confirmar que las ha leído.

```markdown
# CLAUDE.md — Reglas del proyecto catastro-inspire-downloader

## Identidad del proyecto
Plugin QGIS para descarga de datos INSPIRE de Catastro español.
Target: QGIS 3.28 LTR+ / Python 3.9+
Sin dependencias externas: solo librería estándar Python + PyQGIS/PyQt5.

## Estructura de módulos (inmutable)
catastro_inspire/
├── __init__.py          ← punto de entrada QGIS
├── metadata.txt         ← metadatos del plugin
├── plugin.py            ← clase principal, solo inicialización
├── dialog.py            ← UI PyQt5, sin lógica de negocio
├── atom_parser.py       ← parseo feeds ATOM (xml.etree puro, sin imports QGIS)
├── downloader.py        ← descarga HTTP (urllib) y extracción ZIP (zipfile)
├── layer_loader.py      ← carga capas con iface.addVectorLayer()
├── resources/
│   └── icon.png
└── tests/
    └── test_atom_parser.py

## Convenciones de código
- PEP 8 estricto, líneas ≤ 88 caracteres
- Type hints en todas las funciones públicas
- Docstrings en español, formato Google Style
- logging en lugar de print() en todos los módulos
- Nunca bare except: capturar excepciones específicas
- Mensajes al usuario via QgsMessageBar (no QMessageBox salvo errores críticos)
- Rutas temporales via tempfile.mkdtemp(), limpiar en finally

## Reglas de arquitectura
- atom_parser.py NO importa nada de qgis.* (testeable sin QGIS)
- downloader.py NO importa nada de qgis.* (testeable sin QGIS)
- dialog.py NO contiene lógica de descarga ni parseo
- La comunicación entre QgsTask y la UI usa únicamente señales Qt
- Nunca bloquear el hilo principal con operaciones de red

## API PyQGIS frecuente (verificar versión 3.28)
- Cargar capa:    iface.addVectorLayer(path, name, "ogr")
- Mensaje info:   iface.messageBar().pushMessage("Catastro", msg, level=Qgis.Info)
- Mensaje error:  iface.messageBar().pushMessage("Catastro", msg, level=Qgis.Critical)
- Task manager:   QgsApplication.taskManager().addTask(task)
- Ruta temporal:  tempfile.mkdtemp(prefix="catastro_")

## URLs de los feeds ATOM de Catastro
PARCELAS:    https://www.catastro.hacienda.gob.es/INSPIRE/CadastralParcels/ES.SDGC.CP.atom.xml
DIRECCIONES: https://www.catastro.hacienda.gob.es/INSPIRE/Addresses/ES.SDGC.AD.atom.xml
EDIFICIOS:   https://www.catastro.hacienda.gob.es/INSPIRE/buildings/ES.SDGC.BU.atom.xml

## Flujo de navegación ATOM (dos niveles)
1. Feed raíz → lista de gerencias provinciales (cada <entry> tiene link a sub-feed)
2. Sub-feed provincial → lista de municipios (cada <entry> tiene <link rel="enclosure"> con ZIP)
3. ZIP descargado → contiene uno o varios GML

## Lo que la IA NO debe asumir sin verificar
- Estructura exacta de los namespaces XML del feed real
- Comportamiento del servidor Catastro (puede devolver 503, timeouts)
- Nombre exacto de los ficheros GML dentro del ZIP
- Métodos deprecados en QGIS 3.34+ respecto a 3.28
```

---

## Arquitectura objetivo

```
catastro_inspire/
│
├── plugin.py            ← Registro en QGIS, menú, toolbar
│   └── CatastroInspirePlugin
│       ├── initGui()
│       └── unload()
│
├── dialog.py            ← Ventana principal (PyQt5)
│   └── CatastroDialog
│       ├── combo_tipo_dato      (Parcelas / Edificios / Direcciones)
│       ├── combo_provincia      (poblado desde atom_parser)
│       ├── combo_municipio      (poblado dinámicamente)
│       ├── btn_descargar
│       └── progress_bar
│
├── atom_parser.py       ← Lógica de navegación ATOM (Python puro)
│   ├── get_feed(url) → ET.Element
│   ├── get_province_entries(feed) → List[Dict]
│   └── get_municipality_entries(province_url) → List[Dict]
│       # Cada Dict: {"name": str, "download_url": str}
│
├── downloader.py        ← Descarga y extracción (Python puro)
│   ├── download_file(url, dest_dir) → Path
│   └── extract_gml(zip_path) → List[Path]
│
└── layer_loader.py      ← Integración con QGIS
    ├── load_gml_as_layer(gml_path, layer_name) → bool
    └── repair_geometries_if_needed(layer) → QgsVectorLayer
```

**Flujo de datos principal:**

```
Usuario selecciona tipo + municipio
        ↓
dialog.py → lanza QgsTask
        ↓
QgsTask → atom_parser.get_municipality_entries() → URL del ZIP
        ↓
QgsTask → downloader.download_file() → ZIP local
        ↓
QgsTask → downloader.extract_gml() → GML local
        ↓ (señal taskCompleted)
dialog.py → layer_loader.load_gml_as_layer() → capa en QGIS
```

---

## Fase 0 — Configuración del entorno

**Tiempo estimado:** 2–3 horas  
**Entregables:** Repositorio inicializado, VS Code configurado, `CLAUDE.md` creado, entorno Python de QGIS seleccionado.

### Checklist de instalaciones

- [ ] VS Code ≥ 1.85
- [ ] QGIS 3.28 LTR instalado
- [ ] Git instalado (`git config --global user.name` y `user.email`)
- [ ] Claude Code extension instalada en VS Code

### Extensiones VS Code recomendadas

| Extensión | ID | Para qué |
|---|---|---|
| Python | `ms-python.python` | Base imprescindible |
| Pylance | `ms-python.vscode-pylance` | IntelliSense PyQGIS |
| Python Debugger | `ms-python.debugpy` | Breakpoints en plugin |
| Ruff | `charliermarsh.ruff` | Linting y formato automático |
| autoDocstring | `njpwerner.autodocstring` | Esqueletos docstring |
| Error Lens | `usernamehw.errorlens` | Errores inline |
| XML | `redhat.vscode-xml` | Inspección feeds ATOM |
| Git Graph | `mhutchie.git-graph` | Historial visual |

### `settings.json` de VS Code

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
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true
}
```

### Intérprete Python de QGIS

Localizar el ejecutable de Python incluido con QGIS:
- **Windows OSGeo4W:** `C:\OSGeo4W\apps\Python39\python.exe`
- **Windows instalador estándar:** `C:\Program Files\QGIS 3.28\apps\Python39\python.exe`

En VS Code: `Ctrl+Shift+P` → *"Python: Select Interpreter"* → seleccionar el de QGIS.  
Verificar que Pylance resuelve `from qgis.core import QgsVectorLayer` sin errores.

### Plugins QGIS de desarrollo

Instalar desde el gestor de complementos de QGIS:
- **Plugin Builder 3** — genera el esqueleto inicial del plugin
- **Plugin Reloader** — recarga el plugin sin reiniciar QGIS

### Inicialización del repositorio

```bash
mkdir catastro-inspire-downloader
cd catastro-inspire-downloader
git init
```

`.gitignore` inicial:

```gitignore
__pycache__/
*.pyc
*.zip
*.gml
temp/
.vscode/settings.json
*.egg-info/
dist/
```

### `ruff.toml`

```toml
line-length = 88
target-version = "py39"

[lint]
select = ["E", "F", "W", "I"]
ignore = ["E501"]
```

### Configuración del debugging remoto (opcional pero recomendado)

Crear `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach to QGIS",
      "type": "python",
      "request": "attach",
      "connect": { "host": "localhost", "port": 5678 },
      "pathMappings": [
        {
          "localRoot": "${workspaceFolder}",
          "remoteRoot": "${workspaceFolder}"
        }
      ]
    }
  ]
}
```

Symlink para que QGIS vea el plugin durante desarrollo (PowerShell como administrador):

```powershell
$pluginsDir = "$env:APPDATA\QGIS\QGIS3\profiles\default\python\plugins"
$devDir = "$PWD\catastro_inspire"
New-Item -ItemType SymbolicLink -Path "$pluginsDir\catastro_inspire" -Target $devDir
```

### ✅ Criterio de completitud de Fase 0

- VS Code formatea automáticamente al guardar un `.py`
- Pylance no marca error en `from qgis.core import QgsVectorLayer`
- Plugin Reloader disponible en la barra de QGIS
- Repositorio con al menos un commit inicial (`git commit -m "chore: init repo"`)

---

## Fase 1 — Exploración de los feeds ATOM

**Tiempo estimado:** 1–2 horas  
**Entregables:** `DEVLOG.md` con la estructura real documentada. Esta fase produce conocimiento, no código.

### Tareas manuales (el desarrollador, no Claude)

1. Abrir en navegador el feed raíz de Parcelas:  
   `https://www.catastro.hacienda.gob.es/INSPIRE/CadastralParcels/ES.SDGC.CP.atom.xml`

2. Anotar en `DEVLOG.md`:
   - Namespaces declarados en `<feed>`
   - Elementos de cada `<entry>` (title, link, id, updated…)
   - Cómo se distingue un link a sub-feed del link a descarga directa

3. Abrir el sub-feed de una provincia y anotar:
   - Cómo se identifica el municipio (campo `<title>` o `<id>`)
   - Dónde está la URL del ZIP (`<link rel="enclosure">` o similar)

4. Descargar manualmente el ZIP de un municipio pequeño, extraerlo y anotar:
   - Nombres de los ficheros GML encontrados
   - Si hay más de un GML, qué contiene cada uno (abrir en QGIS manualmente)

### Plantilla `DEVLOG.md`

```markdown
# DEVLOG — catastro-inspire-downloader

## [YYYY-MM-DD] — Fase 1: Exploración ATOM

### Feed raíz (Parcelas)
- Namespaces: [anotar]
- Estructura de <entry>: [anotar]
- URL del sub-feed provincial: campo [anotar]

### Sub-feed provincial
- Estructura de <entry>: [anotar]
- URL del ZIP municipal: campo [anotar]
- Tildes y caracteres especiales: [¿se codifican o aparecen literalmente?]

### ZIP descargado
- Ficheros GML encontrados: [listar]
- CRS detectado en QGIS: [anotar]
- Tipos de geometría: [anotar]

### Preguntas abiertas
- [Lo que no quedó claro]
```

### 🤖 Prompts para esta fase

**Prompt 1.A — Interpretar el XML observado**

```
Estoy construyendo un plugin QGIS para descargar datos INSPIRE de Catastro español.
He inspeccionado el feed ATOM raíz de Parcelas y este es el fragmento XML relevante:

[PEGAR 20-30 LÍNEAS DEL XML REAL]

Explícame:
1. Qué significa cada elemento relevante para mi caso de uso
2. Cómo navego desde este feed hasta la URL de descarga de un municipio
3. Qué namespaces necesito declarar en Python para parsear esto con xml.etree.ElementTree
4. Qué casos de inconsistencia debo anticipar (títulos con tildes, entradas vacías, etc.)
```

**Prompt 1.B — Aclarar contenido del ZIP**

```
He descargado manualmente el ZIP de datos INSPIRE de Catastro para un municipio pequeño.
Dentro encontré estos ficheros: [LISTAR FICHEROS].

Para cada tipo de fichero (especialmente los GML), explícame:
1. Qué contiene geométricamente
2. Qué atributos son relevantes para un usuario de QGIS
3. Si hay múltiples GML, ¿debo cargar todos o dejar que el usuario elija?
4. Cómo identificar qué GML corresponde a qué tipo de dato sin abrirlo
```

---

## Fase 2 — Esqueleto del plugin

**Tiempo estimado:** 1–2 horas  
**Entregables:** Estructura de carpetas completa, `metadata.txt` válido, módulos vacíos con docstrings.

### 🤖 Prompts para esta fase

**Prompt 2.A — Generar esqueleto completo**

```
Actúa como desarrollador senior de plugins QGIS. Lee el CLAUDE.md de este proyecto
para adoptar todas las convenciones antes de generar código.

Genera la estructura completa de archivos para el plugin 'catastro_inspire':

1. metadata.txt con:
   - name = Catastro INSPIRE Downloader
   - qgisMinimumVersion = 3.28
   - description (en español)
   - version = 0.1.0
   - author, email, homepage, tracker, repository (puedes poner placeholders)
   - tags, category = Web

2. __init__.py con la función classFactory() mínima correcta

3. plugin.py con la clase CatastroInspirePlugin:
   - __init__(self, iface)
   - initGui() que añade acción al menú Web y toolbar
   - unload() que elimina la acción correctamente
   - run() que abre el diálogo

4. Módulos vacíos (solo imports y docstring de módulo):
   - atom_parser.py
   - downloader.py
   - layer_loader.py
   - dialog.py

5. tests/__init__.py vacío

Crea todos los ficheros directamente. Después explícame el ciclo de vida
initGui/unload y por qué es importante limpiar en unload().
```

**Prompt 2.B — Verificar que el plugin carga**

```
He generado el esqueleto del plugin. Este es el contenido de mis ficheros clave:

__init__.py: [PEGAR]
plugin.py: [PEGAR]
metadata.txt: [PEGAR]

Dime si hay algún error que impediría que QGIS cargue el plugin.
Explícame qué comprueba QGIS al cargar un plugin (metadata, classFactory, initGui).
```

---

## Fase 3 — Módulo de parseo ATOM

**Tiempo estimado:** 2–3 horas  
**Entregables:** `atom_parser.py` funcional y testeado sin QGIS.

### Contrato del módulo

```python
# atom_parser.py — sin imports de qgis.*

FEED_URLS: dict[str, str] = {
    "parcelas": "https://www.catastro.hacienda.gob.es/INSPIRE/CadastralParcels/ES.SDGC.CP.atom.xml",
    "direcciones": "https://www.catastro.hacienda.gob.es/INSPIRE/Addresses/ES.SDGC.AD.atom.xml",
    "edificios": "https://www.catastro.hacienda.gob.es/INSPIRE/buildings/ES.SDGC.BU.atom.xml",
}

def get_feed(url: str, timeout: int = 30) -> ET.Element:
    """Descarga y parsea un feed ATOM. Lanza HTTPError, URLError o ParseError."""

def get_province_entries(feed: ET.Element) -> list[dict]:
    """Extrae lista de provincias del feed raíz.
    Retorna: [{"name": str, "feed_url": str}, ...]
    """

def get_municipality_entries(province_feed_url: str) -> list[dict]:
    """Extrae lista de municipios de un sub-feed provincial.
    Retorna: [{"name": str, "download_url": str}, ...]
    """

def find_municipality(
    province_feed_url: str,
    municipality_name: str,
    fuzzy: bool = True
) -> dict | None:
    """Busca un municipio por nombre. Con fuzzy=True tolera tildes y mayúsculas.
    Retorna el dict del municipio o None si no se encuentra.
    """
```

### 🤖 Prompts para esta fase

**Prompt 3.A — Implementar atom_parser.py**

```
Lee CLAUDE.md para adoptar las convenciones del proyecto antes de escribir código.

Implementa atom_parser.py según el contrato definido en el plan de trabajo.
Los namespaces reales que observé en el feed son: [PEGAR LOS NAMESPACES DEL DEVLOG]
La estructura real de las entradas es: [PEGAR FRAGMENTO XML DEL DEVLOG]

Requisitos adicionales:
- get_feed(): timeout configurable, maneja ConnectionError, urllib.error.HTTPError,
  urllib.error.URLError y xml.etree.ElementTree.ParseError con mensajes descriptivos
- find_municipality(): normaliza tildes con unicodedata.normalize() antes de comparar,
  usa difflib.get_close_matches() para búsqueda aproximada
- Incluir un bloque if __name__ == "__main__" que ejecute una prueba básica
  contra el feed real (útil para probar sin QGIS)

Después del código, señala qué partes deberían verificarse contra el XML real
porque la IA podría estar asumiendo la estructura incorrecta.
```

**Prompt 3.B — Revisar el código generado**

```
Este es el código de atom_parser.py que generamos:

[PEGAR CÓDIGO]

Hazme una revisión crítica:
1. ¿Hay algún caso edge donde el parseo fallaría silenciosamente?
2. ¿El manejo de namespaces es robusto si Catastro cambia el prefijo?
3. ¿find_municipality() funciona correctamente con "MÁLAGA" vs "Málaga" vs "malaga"?
4. ¿Qué debería añadir a tests/test_atom_parser.py para cubrir los casos más importantes?
```

**Prompt 3.C — Generar tests unitarios**

```
Crea tests/test_atom_parser.py con pruebas unitarias para atom_parser.py usando pytest.

Tests necesarios:
1. test_get_province_entries_from_fixture(): usa un XML de ejemplo pegado como string
2. test_get_municipality_entries_from_fixture(): ídem para sub-feed provincial
3. test_find_municipality_exact(): búsqueda exacta que sí encuentra
4. test_find_municipality_fuzzy(): búsqueda con tilde omitida que sí encuentra
5. test_find_municipality_not_found(): busca algo que no existe, retorna None
6. test_get_feed_timeout(): mockea urllib para simular timeout

Para los fixtures XML, usa fragmentos mínimos que contengan la estructura real
que observamos en el DEVLOG. Explícame cómo ejecutar pytest sin tener QGIS instalado.
```

---

## Fase 4 — Módulo de descarga y extracción

**Tiempo estimado:** 1–2 horas  
**Entregables:** `downloader.py` funcional.

### Contrato del módulo

```python
# downloader.py — sin imports de qgis.*

def download_file(
    url: str,
    dest_dir: str,
    timeout: int = 60,
    max_retries: int = 2,
    on_progress: Callable[[int], None] | None = None
) -> Path:
    """Descarga un fichero ZIP a dest_dir.
    on_progress recibe porcentaje 0-100.
    Reintenta max_retries veces ante HTTPError 503 o timeout.
    Lanza DownloadError con mensaje descriptivo en caso de fallo definitivo.
    """

def extract_gml(zip_path: Path, dest_dir: Path | None = None) -> list[Path]:
    """Extrae todos los ficheros .gml del ZIP.
    Retorna lista de rutas a los GML extraídos.
    Lanza ExtractionError si el ZIP está corrupto o no contiene GML.
    """

class DownloadError(Exception): ...
class ExtractionError(Exception): ...
```

### 🤖 Prompts para esta fase

**Prompt 4.A — Implementar downloader.py**

```
Lee CLAUDE.md. Implementa downloader.py según el contrato del plan de trabajo.

El servidor de Catastro es lento (puede tardar 2-5 minutos para municipios grandes)
y a veces devuelve HTTP 503. Requisitos específicos:

- Usa urllib.request.urlopen (no requests)
- Descarga en chunks de 8192 bytes para poder reportar progreso
- on_progress(porcentaje): llamado cada vez que se descarga un chunk
- Backoff exponencial entre reintentos (esperar 2^intento segundos)
- Limpia el fichero parcial si la descarga falla
- El nombre del fichero en dest_dir debe preservar el nombre original de la URL
- extract_gml(): maneja encoding de nombres de fichero dentro del ZIP
  (algunos ZIPs de Catastro usan cp437 en lugar de utf-8)

Incluye bloque __main__ con ejemplo de uso.
```

**Prompt 4.B — Manejo de errores del servidor Catastro**

```
El servidor de Catastro tiene estos comportamientos conocidos:

- HTTP 503 en horario pico (9:00-14:00 laborables)
- Timeouts frecuentes en ZIPs grandes (municipios con >10.000 parcelas)
- A veces el ZIP se descarga completo pero está corrupto

Revisa el código de downloader.py y mejora el manejo de estos tres casos
con mensajes de error que sean útiles para el usuario final (no para el desarrollador).
Los mensajes deben aparecer en la barra de mensajes de QGIS, no en la consola.
```

---

## Fase 5 — Módulo de carga de capas

**Tiempo estimado:** 1–2 horas  
**Entregables:** `layer_loader.py` funcional.

### Contrato del módulo

```python
# layer_loader.py — requiere qgis.core

def load_gml_as_layer(
    gml_path: Path,
    layer_name: str,
    force_crs: str = "EPSG:25830"
) -> bool:
    """Carga un GML como capa vectorial en el proyecto QGIS activo.
    force_crs: CRS a asignar si el GML no declara uno válido.
    Retorna True si la capa se cargó correctamente.
    """

def load_all_gml_layers(
    gml_paths: list[Path],
    municipality_name: str,
    data_type: str
) -> dict[str, bool]:
    """Carga múltiples GML (caso parcelas: CP_CadastralParcel + CP_CadastralZoning).
    Nombra cada capa como "{data_type} — {municipality_name} — {tipo_geometría}".
    Retorna dict {nombre_capa: éxito}.
    """

def repair_geometries_if_needed(layer: QgsVectorLayer) -> int:
    """Repara geometrías inválidas con makeValid().
    Retorna el número de geometrías reparadas (0 si ninguna era inválida).
    """
```

### 🤖 Prompts para esta fase

**Prompt 5.A — Implementar layer_loader.py**

```
Lee CLAUDE.md. Implementa layer_loader.py según el contrato del plan de trabajo.

Contexto sobre los datos de Catastro:
- El CRS oficial es EPSG:25830 (ETRS89 UTM 30N) para la península
- Algunos GML de Parcelas contienen dos tipos de features:
  CP_CadastralParcel (parcelas individuales) y CP_CadastralZoning (manzanas)
  QGIS los carga como sub-capas separadas — debes cargar ambas
- Los GML de Edificios y Direcciones suelen tener una sola capa

Para load_gml_as_layer():
- Usa iface.addVectorLayer()
- Si la capa no es válida tras cargarla, intenta con driver="GML" explícito
- Asigna el CRS con layer.setCrs() si no se detecta automáticamente
- Añade mensaje informativo en QgsMessageBar al terminar

Para repair_geometries_if_needed():
- Solo reparar si QgsGeometry.isGeosValid() falla
- Usar feature.geometry().makeValid() (disponible desde QGIS 3.20)
- Loguear cuántas geometrías se repararon con logging.info
- No perder atributos durante la reparación
```

**Prompt 5.B — Verificar compatibilidad de CRS**

```
En mi plugin de Catastro, cargo capas GML con CRS EPSG:25830 en proyectos QGIS
que pueden tener cualquier CRS activo (EPSG:4326, EPSG:3857, etc.).

Explícame:
1. ¿QGIS reproyecta automáticamente "al vuelo" si el proyecto tiene un CRS diferente?
2. ¿Debo hacer algo explícito en el código para que funcione?
3. ¿Hay diferencia de comportamiento entre QGIS 3.28 y 3.34?
4. ¿Debo añadir alguna comprobación o aviso al usuario?
```

---

## Fase 6 — Interfaz de usuario PyQt5

**Tiempo estimado:** 2–3 horas  
**Entregables:** `dialog.py` funcional con todos los controles conectados.

### Especificación de la UI

```
Ventana: CatastroDialog (QDialog)
Título: "Catastro INSPIRE Downloader"
Tamaño mínimo: 420 × 300 px

Controles (de arriba a abajo):
┌─────────────────────────────────────────┐
│  Tipo de dato:   [QComboBox: combo_tipo] │
│  Provincia:      [QComboBox: combo_prov] │
│  Municipio:      [QComboBox: combo_mun ] │
│                                          │
│  [QPushButton: btn_descargar]            │
│  [QProgressBar: progress_bar]  (oculta)  │
│  [QLabel: lbl_status]                    │
└─────────────────────────────────────────┘

Comportamiento:
- combo_tipo: opciones fijas ["Parcelas", "Edificios", "Direcciones"]
- combo_prov: se puebla al abrir el diálogo (async si es posible)
- combo_mun: se puebla cuando cambia combo_prov
- btn_descargar: deshabilitado hasta que haya municipio seleccionado
- progress_bar: visible solo durante descarga
- lbl_status: muestra mensajes de estado al usuario
```

### 🤖 Prompts para esta fase

**Prompt 6.A — Implementar dialog.py**

```
Lee CLAUDE.md. Implementa dialog.py en Python puro (sin archivo .ui de Qt Designer)
según la especificación de UI del plan de trabajo.

La clase CatastroDialog debe:
1. Construir todos los widgets en __init__() con setupUi() propio
2. Conectar señales en _connect_signals() separado
3. Usar atom_parser para poblar combo_prov en showEvent() (no en __init__)
4. on_province_changed(): poblar combo_mun con get_municipality_entries()
5. on_download_clicked(): delegar al slot de Fase 7
6. _update_download_button(): habilitar btn_descargar solo si hay municipio válido

Patrones obligatorios:
- Separación estricta UI / lógica (dialog.py no importa downloader ni layer_loader)
- Llamar disconnect() en closeEvent() para evitar memory leaks
- Type hints y docstrings Google style en todos los métodos públicos
```

**Prompt 6.B — Manejar estados de la UI durante descarga**

```
Mi diálogo de Catastro necesita manejar tres estados visuales:

Estado IDLE: controles habilitados, progress_bar oculta, lbl_status vacío
Estado LOADING: btn_descargar deshabilitado, combo_* deshabilitados,
                progress_bar visible e indeterminada, lbl_status = "Descargando..."
Estado DONE: igual que IDLE, lbl_status = "Capa cargada: {nombre}"
Estado ERROR: igual que IDLE, lbl_status en rojo = "Error: {mensaje}"

Implementa un método set_ui_state(state: UIState) donde UIState es un Enum
con los cuatro estados. Esto evita que tengamos que habilitar/deshabilitar
widgets uno a uno en varios lugares del código.
```

---

## Fase 7 — Asincronía con QgsTask

**Tiempo estimado:** 2–3 horas  
**Entregables:** Flujo de descarga asíncrono usando `QgsTask`, sin bloqueos en QGIS.

### Diseño de la tarea

```python
class DownloadCatastroTask(QgsTask):
    """QgsTask que ejecuta la descarga completa en segundo plano.

    Señales:
        progress_updated(int): emite porcentaje 0-100
        download_finished(list[Path]): emite lista de GML descargados al terminar
        download_failed(str): emite mensaje de error si falla
    """
```

### 🤖 Prompts para esta fase

**Prompt 7.A — Implementar DownloadCatastroTask**

```
Lee CLAUDE.md. Implementa la clase DownloadCatastroTask que extiende QgsTask.

La tarea recibe en __init__:
- data_type: str (parcelas/direcciones/edificios)
- province_feed_url: str
- municipality_name: str
- temp_dir: str

El método run() debe:
1. Llamar a atom_parser.find_municipality() para obtener la URL del ZIP
2. Llamar a downloader.download_file() con on_progress que emite setProgress()
3. Llamar a downloader.extract_gml() para obtener la lista de GML
4. Retornar True si todo fue bien, False si falló

Regla crítica: run() se ejecuta en un hilo secundario.
NUNCA llamar a iface.* ni a métodos de PyQt que modifiquen la UI desde run().
Toda comunicación con la UI debe ser via señales.

Explícame por qué QgsTask es mejor que threading.Thread para plugins QGIS.
```

**Prompt 7.B — Conectar la tarea con el diálogo**

```
Tengo implementados:
- DownloadCatastroTask (Fase 7.A)
- dialog.py con on_download_clicked() vacío (Fase 6.A)
- layer_loader.load_all_gml_layers() (Fase 5.A)

Implementa on_download_clicked() en dialog.py que:
1. Crea un DownloadCatastroTask con los valores actuales del diálogo
2. Conecta download_finished a un slot que llama a layer_loader
3. Conecta download_failed a un slot que muestra el error y restaura la UI
4. Conecta progressChanged a progress_bar.setValue()
5. Lanza la tarea con QgsApplication.taskManager().addTask()
6. Cambia el estado de la UI a LOADING

⚠️ PAUSA: Antes de implementar, explícame qué pasa si el usuario cierra
el diálogo mientras la tarea está corriendo. ¿Cómo gestionamos eso?
```

---

## Fase 8 — Integración y pruebas

**Tiempo estimado:** 2–4 horas  
**Entregables:** Plugin funcionando end-to-end, casos de error documentados.

### Checklist de pruebas manuales

- [ ] Parcelas + municipio pequeño → capa se carga correctamente
- [ ] Edificios + municipio → capa se carga
- [ ] Direcciones + municipio → capa se carga
- [ ] Municipio con tilde en el nombre (Cádiz, Málaga, Castellón...) → funciona
- [ ] Cargar tres tipos del mismo municipio → tres grupos de capas en QGIS
- [ ] Sin conexión a internet → mensaje de error amigable, no crash
- [ ] Cerrar diálogo durante descarga → tarea se cancela, no quedan temporales
- [ ] Recargar plugin con Plugin Reloader → sin errores ni duplicados en menú
- [ ] Proyecto QGIS con CRS distinto a EPSG:25830 → capas se reproyectan al vuelo

### Tabla de casos de error esperados

| Situación | Mensaje esperado para el usuario |
|---|---|
| HTTP 503 del servidor | "El servidor de Catastro no está disponible. Inténtalo fuera del horario pico (9-14h)." |
| Timeout de descarga | "Tiempo de espera agotado. El municipio puede ser muy grande. Inténtalo de nuevo." |
| Municipio sin datos | "No se encontraron datos para este municipio en el servicio INSPIRE." |
| ZIP corrupto | "El fichero descargado está corrupto. Inténtalo de nuevo." |
| GML no cargable | "El fichero GML no pudo cargarse en QGIS. Comprueba el log de mensajes." |

### 🤖 Prompts para esta fase

**Prompt 8.A — Auditoría de manejo de errores**

```
Este es el flujo completo de mi plugin desde el clic en "Descargar" hasta la capa en QGIS:

dialog.py — on_download_clicked():
[PEGAR CÓDIGO]

DownloadCatastroTask.run():
[PEGAR CÓDIGO]

downloader.download_file():
[PEGAR CÓDIGO]

layer_loader.load_all_gml_layers():
[PEGAR CÓDIGO]

Identifica todos los puntos donde puede fallar que actualmente NO están cubiertos.
Para cada uno: excepción posible, qué vería el usuario, cómo capturarlo.
Prioriza los más probables con el servidor real de Catastro.
```

**Prompt 8.B — Prueba de humo desde consola Python de QGIS**

```
Genera un script que pueda ejecutar en la consola Python de QGIS (sin UI)
para verificar que los módulos principales funcionan:

1. Importa atom_parser y descarga la lista de provincias de Parcelas
2. Busca "MADRID" en la primera provincia que corresponda
3. Descarga el ZIP a una carpeta temporal
4. Extrae el GML
5. Carga la primera capa en el proyecto activo
6. Imprime en consola si cada paso tuvo éxito o falló

Este script permite detectar errores de importación y lógica antes de
probar el plugin completo con la UI.
```

---

## Fase 9 — Empaquetado y publicación

**Tiempo estimado:** 1–2 horas  
**Entregables:** `catastro_inspire.zip` válido para instalar en QGIS.

### 🤖 Prompts para esta fase

**Prompt 9.A — Script de empaquetado**

```
Crea un script Python en la raíz del proyecto llamado build_plugin.py que:

1. Verifica que metadata.txt tiene version, name y qgisMinimumVersion
2. Copia los ficheros del plugin (excluye: __pycache__, .venv, tests,
   .vscode, *.pyc, *.zip, DEVLOG.md, build_plugin.py)
3. Genera catastro_inspire.zip con la estructura correcta:
   catastro_inspire/
   ├── __init__.py
   ├── metadata.txt
   └── [resto de ficheros]
4. Valida que el ZIP se puede descomprimir y contiene metadata.txt
5. Imprime la ruta del ZIP generado y cómo instalarlo en QGIS

Instrucciones para instalar el ZIP: Complementos → Administrar e instalar
complementos → Instalar desde ZIP.
```

**Prompt 9.B — Revisar metadata.txt para publicación**

```
Este es mi metadata.txt actual:
[PEGAR CONTENIDO]

Revísalo para publicación en el repositorio oficial de plugins de QGIS:
1. ¿Faltan campos obligatorios?
2. ¿Los valores son apropiados para un repositorio público?
3. ¿Hay algo que podría provocar el rechazo del plugin en la revisión?

Referencia: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/plugins/plugins.html#plugin-metadata-table
```

---

## Fase 10 — Documentación final

**Tiempo estimado:** 2–3 horas  
**Entregables:** `README.md` completo, docstrings auditados.

### 🤖 Prompts para esta fase

**Prompt 10.A — Generar README.md**

```
Mi plugin QGIS de Catastro está terminado. Aquí está el código de los módulos principales:

atom_parser.py: [PEGAR]
downloader.py: [PEGAR]
layer_loader.py: [PEGAR]
dialog.py: [PEGAR]

Y mi DEVLOG.md: [PEGAR]

Genera un README.md en español que incluya:
1. Descripción del plugin (para usuarios no técnicos)
2. Capturas de pantalla (describirlas en texto con [SCREENSHOT: descripción])
3. Requisitos: QGIS 3.28+, sin dependencias adicionales
4. Instalación paso a paso desde ZIP
5. Guía de uso con pasos numerados
6. Sección técnica para desarrolladores: arquitectura en 2 párrafos
7. Limitaciones conocidas
8. Contribuciones y licencia

Tono: claro, directo, accesible para técnicos GIS sin experiencia en Python.
```

**Prompt 10.B — Auditoría de docstrings**

```
Estos son los docstrings de todas las funciones públicas del plugin:

[PEGAR FIRMAS + DOCSTRINGS DE atom_parser.py, downloader.py, layer_loader.py, dialog.py]

Para cada función, evalúa:
1. ¿El docstring describe el comportamiento real actual?
2. ¿Los tipos en Args/Returns coinciden con los type hints del código?
3. ¿Los Raises documentan todas las excepciones que el código puede lanzar?
4. ¿Hay algo que un desarrollador nuevo malinterpretaría?

Devuelve una tabla: función | estado (OK/MEJORAR/INCORRECTO) | cambio sugerido.
```

**Prompt 10.C — Reflexión de aprendizaje**

```
He terminado mi primer plugin QGIS usando desarrollo asistido por IA.
Módulos desarrollados: atom_parser, downloader, layer_loader, dialog, plugin.
DEVLOG completo: [PEGAR]

Ayúdame a escribir una reflexión de 3-4 párrafos para el README titulada
"Notas técnicas del desarrollo" que mencione:
1. Las decisiones de diseño más importantes y por qué
2. Dónde fue más útil la IA y dónde tuve que verificar manualmente
3. Qué haría diferente en una segunda versión

Tono: técnico pero accesible, honesto sobre las limitaciones del proceso.
```

---

## Referencia de recursos

| Recurso | URL |
|---|---|
| Plugin de referencia | https://github.com/sigdeletras/Spanish-Inspire-Catastral-Downloader |
| PyQGIS API 3.28 | https://qgis.org/pyqgis/3.28/ |
| PyQGIS Cookbook | https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/ |
| QgsTask documentación | https://qgis.org/pyqgis/master/core/QgsTask.html |
| INSPIRE Catastro | https://www.catastro.hacienda.gob.es/INSPIRE/ |
| Publicar en repositorio | https://plugins.qgis.org/publish/ |

### Índice de todos los prompts

| ID | Fase | Qué genera |
|---|---|---|
| 1.A | Exploración | Interpretación del XML observado |
| 1.B | Exploración | Interpretación del contenido del ZIP |
| 2.A | Esqueleto | Estructura completa de ficheros |
| 2.B | Esqueleto | Verificación de carga en QGIS |
| 3.A | ATOM parser | Implementación de atom_parser.py |
| 3.B | ATOM parser | Code review del parseo |
| 3.C | ATOM parser | Tests unitarios |
| 4.A | Downloader | Implementación de downloader.py |
| 4.B | Downloader | Manejo de errores del servidor |
| 5.A | Layer loader | Implementación de layer_loader.py |
| 5.B | Layer loader | Verificación de compatibilidad CRS |
| 6.A | UI | Implementación de dialog.py |
| 6.B | UI | Gestión de estados visuales |
| 7.A | Async | Implementación de DownloadCatastroTask |
| 7.B | Async | Conexión tarea-diálogo |
| 8.A | Integración | Auditoría de errores end-to-end |
| 8.B | Integración | Script de prueba de humo |
| 9.A | Empaquetado | Script build_plugin.py |
| 9.B | Empaquetado | Revisión de metadata.txt |
| 10.A | Documentación | README.md completo |
| 10.B | Documentación | Auditoría de docstrings |
| 10.C | Documentación | Reflexión de aprendizaje |

---

## Reglas de uso con Claude Code (VS Code)

### Cómo iniciar cada sesión de trabajo

Al abrir VS Code con este proyecto, lanza este prompt inicial:

```
Lee el fichero CLAUDE.md y el fichero PLAN_PLUGIN_CATASTRO_INSPIRE.md.
Confirma que has leído ambos resumiendo en 3 puntos las convenciones principales.
Después dime en qué fase estamos según el estado actual de los ficheros del proyecto.
```

### Regla de oro

> **No aceptes código que no puedas explicar en voz alta.**
> Si no puedes decir "esta función hace X porque Y", pide que te lo expliquen antes de continuar.

### Patrón de prompt más efectivo para aprender

```
Antes de escribir el código, explícame [concepto o decisión técnica].
Luego genera el código con docstrings en español y type hints.
Finalmente, dime qué partes deberías verificar contra la documentación
oficial de PyQGIS porque la IA podría estar asumiendo algo incorrecto.
```

### Lo que Claude NO sabe con certeza

- La estructura XML real del feed de Catastro (puede haber cambiado)
- Comportamiento del servidor Catastro en producción (timeouts, campos ausentes)
- Métodos deprecados en versiones de QGIS más nuevas que su conocimiento
- Errores específicos de tu entorno (versión de QGIS, SO, configuración Windows)

**Siempre contrasta con inspección manual del XML real antes de asumir la estructura.**

---

*Plan unificado a partir de propuestas de ChatGPT, DeepSeek, Gemini y Claude.*  
*Actualiza DEVLOG.md al terminar cada fase.*
