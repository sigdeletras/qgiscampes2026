Aquí tienes el **plan de trabajo completo** para que lo copies directamente:

---

# Plan de Trabajo: Plugin Catastro INSPIRE para QGIS

**Autor:** [Tu nombre]  
**Fecha inicio:** [Fecha actual]  
**Herramientas:** VSCode + Claude + QGIS 3.x  
**Objetivo:** Plugin que descarga datos INSPIRE de Catastro (parcelas/direcciones/edificios) por municipio y los carga en QGIS

---

## Fase 0: Configuración del entorno (1 día)

### 0.1 Instalación de herramientas base

**Prompt para Claude:**
```
Estoy en Windows 11. Necesito instalar las herramientas para desarrollar un plugin QGIS. Dame los comandos exactos paso a paso para instalar:

1. Python 3.9+ (el que usa QGIS)
2. OSGeo4W con QGIS 3.x
3. Visual Studio Code
4. Extensión Python de Microsoft en VSCode
5. qgis-venv-creator (para el entorno virtual)

Quiero que los comandos sean verificables y me expliques qué hace cada uno.
```

### 0.2 Configuración del proyecto y entorno virtual

**Prompt para Claude:**
```
He instalado QGIS en C:\OSGeo4W. Ahora quiero:

1. Crear la carpeta de mi plugin en: C:\Users\mi_usuario\Documents\QGIS_plugins\catastro_inspire
2. Usando qgis-venv-creator, dime los comandos exactos para crear un entorno virtual .venv que referencie las librerías de QGIS
3. ¿Cómo abro esta carpeta en VSCode y selecciono el entorno virtual?

Nota: Soy principiante en entornos virtuales, explícame qué está pasando.
```

### 0.3 Configuración del debugging remoto

**Prompt para Claude:**
```
Quiero poder depurar mi plugin en VSCode mientras se ejecuta en QGIS. Tengo instalado QGIS DevTools en QGIS. Dime:

1. Cómo activar el modo depuración en QGIS DevTools (qué puerto usa)
2. El contenido exacto del archivo .vscode/launch.json para conectarme
3. Cómo crear un symlink entre mi carpeta de desarrollo y la carpeta de plugins de QGIS (ruta: C:\Users\mi_usuario\AppData\Roaming\QGIS\QGIS3\profiles\default\python\plugins)

Dame comandos para Windows PowerShell.
```

### 0.4 Instalación de skills de Claude para QGIS

**Prompt para Claude:**
```
Necesito que actúes como experto en desarrollo de plugins QGIS. Quiero que memorices estas reglas para todo el proyecto:

1. Usa QgsTask o QgsNetworkAccessManager para descargas largas (no requests síncrono)
2. Separa UI de lógica (patrón MVC)
3. Usa logging en lugar de print
4. Sigue PEP 8, type hints, docstrings
5. Para XML con namespaces, usa nsmap con prefijos

Confírmame que has incorporado estas reglas y pregunta si necesitas alguna aclaración antes de empezar a codificar.
```

### 0.5 Estructura inicial de carpetas

**Prompt para Claude:**
```
Genera la estructura de carpetas y archivos iniciales para mi plugin 'catastro_inspire', incluyendo:

- __init__.py (versión mínima)
- catastro_inspire.py (clase principal del plugin)
- metadata.txt (con las claves básicas: name, description, version, qgisMinimumVersion)
- resources.py (vacío por ahora)

Dame el contenido de cada archivo. Usa nombres en inglés para el código pero comentarios en español.
```

---

## Fase 1: Componente de descarga ATOM (2-3 días)

### 1.1 Cliente básico para feeds ATOM

**Prompt para Claude:**
```
En el archivo descarga_atom.py, crea una clase `ClienteAtom` con:

- __init__(self): configura logging y timeout = 30 segundos
- obtener_feed(self, url: str) -> ET.Element: descarga y parsea XML con namespaces
- extraer_entradas_gerencia(self, feed_root: ET.Element) -> List[Dict]: del feed principal, extrae cada <entry> con título y link

Requisitos:
- Maneja errores: ConnectionError, Timeout, HTTPError, ParseError
- Usa logging.info/warning/error según el caso
- Incluye type hints y docstring Google Style

Después del código, explícame qué hace el manejo de namespaces y por qué es necesario.
```

### 1.2 Navegación al segundo nivel (búsqueda por municipio)

**Prompt para Claude:**
```
Amplía `ClienteAtom` con el método:

`buscar_url_por_municipio(self, feed_gerencia_url: str, nombre_municipio: str) -> Optional[str]`

Este método debe:
1. Descargar el feed de segundo nivel (el de una gerencia concreta)
2. Buscar en los <entry> cuyo <title> contenga el nombre_municipio
3. Extraer la URL de descarga del <link rel="enclosure">
4. Devolver None si no encuentra

Bonus: Usa difflib.get_close_matches para tolerar pequeñas diferencias en el nombre.

Dame el código completo y un ejemplo de cómo lo llamaría desde el plugin principal.
```

### 1.3 Integración de los tres tipos de datos

**Prompt para Claude:**
```
Crea una clase `ApiCatastro` que unifique la descarga de los tres tipos:

- parcelas: https://www.catastro.hacienda.gob.es/INSPIRE/CadastralParcels/ES.SDGC.CP.atom.xml
- direcciones: https://www.catastro.hacienda.gob.es/INSPIRE/Addresses/ES.SDGC.AD.atom.xml
- edificios: https://www.catastro.hacienda.gob.es/INSPIRE/buildings/ES.SDGC.BU.atom.xml

Métodos:
- listar_gerencias(tipo: str) -> List[Dict]
- descargar_municipio(tipo: str, municipio: str) -> Optional[str] (ruta del GML descargado)

Implementa la lógica de dos niveles (feed principal → feed gerencia → búsqueda municipio). Usa la clase ClienteAtom existente.
```

### 1.4 Barra de progreso y manejo de hilos

**Prompt para Claude:**
```
Quiero que la descarga larga no congele QGIS. Crea una función que use QgsTask:

`descargar_con_progreso(tipo: str, municipio: str, feedback: QgsFeedback) -> str`

Dentro:
- Emite señales de progreso (0-100%)
- Permite cancelación con feedback.isCanceled()
- Retorna la ruta del archivo descargado o lanza excepción

Necesito el código completo y una explicación de por qué QgsTask es mejor que threading.Thread para plugins QGIS.
```

---

## Fase 2: Carga de datos en QGIS (2 días)

### 2.1 Cargar GML como capa vectorial

**Prompt para Claude:**
```
En `cargador_gml.py`, crea la función:

`cargar_gml_en_qgis(ruta_gml: str, nombre_capa: str) -> bool`

Usa iface.addVectorLayer() con driver 'ogr'. Maneja:
- CRS: forzar a EPSG:25830 (ETRS89 UTM 30N) si no se detecta automáticamente
- Si el GML tiene múltiples geometrías, carga todas
- Retorna True si éxito, False si error con logging

Añade un ejemplo de uso dentro de QGIS. Explícame cómo probar esta función de forma aislada antes de integrarla al plugin.
```

### 2.2 Manejo de parcelas (dos capas: parcelas + zonas)

**Prompt para Claude:**
```
El GML de parcelas contiene dos tipos de objetos: CP_CadastralParcel (parcelas) y CP_CadastralZoning (manzanas/polígonos).

Crea `cargar_parcelas_completas(ruta_gml: str) -> Dict[str, bool]` que:
1. Cargue ambas capas en QGIS
2. Renombre a "Parcelas - {municipio}" y "Zonas - {municipio}"
3. Devuelva diccionario con éxito/fracaso de cada una

Si el usuario elige "Edificios" o "Direcciones" (que tienen solo un tipo), usa la función genérica anterior.
```

### 2.3 Limpieza de geometrías inválidas

**Prompt para Claude:**
```
A veces los GML del Catastro tienen geometrías inválidas (auto-intersecciones, anillos mal orientados). Añade a `cargador_gml.py`:

`reparar_geometrias_si_necesario(capa: QgsVectorLayer) -> QgsVectorLayer`

Debe:
- Iterar sobre las features
- Si QgsGeometry().isGeosValid() es False, aplicar .makeValid()
- Usar QgsProcessingAlg solo si hay muchas geometrías inválidas
- Logging de cuántas geometrías se repararon

¿Qué precauciones debo tomar para no perder atributos durante la reparación?
```

---

## Fase 3: Interfaz de usuario (2-3 días)

### 3.1 Diálogo básico con Qt Designer

**Prompt para Claude:**
```
Necesito un diálogo para mi plugin con:

- QComboBox para seleccionar tipo de datos: "Parcelas", "Direcciones", "Edificios"
- QLineEdit con autocompletado para municipio (cargar lista desde CSV o API)
- QProgressBar (inicialmente oculta)
- QPushButton "Descargar y cargar"
- QTextEdit para logs (opcional)

Dame el código Python para este diálogo (usando PyQt5 sin archivo .ui) y explícame cómo conectarlo a la clase principal del plugin.
```

### 3.2 Autocompletado de municipios

**Prompt para Claude:**
```
Quiero que el campo municipio tenga autocompletado con todos los municipios de España. Crea:

`cargar_municipios_desde_ine() -> List[str]`

Que descargue el código INE de municipios desde:
https://www.ine.es/daco/daco42/codmun/codmunmapa.htm

O alternativamente, usa un CSV local si la descarga falla. Dame el código y dime qué archivos debo incluir en mi plugin para tener una lista offline de respaldo.
```

### 3.3 Conectar UI con la lógica

**Prompt para Claude:**
```
En la clase principal `CatastroInspirePlugin`, implementa:

`on_descargar_clicked(self)`

Que:
1. Obtiene tipo y municipio del diálogo
2. Muestra la progress bar
3. Crea un QgsTask con la función `descargar_con_progreso` (Fase 1.4)
4. Al terminar, llama a cargar_gml_en_qgis (Fase 2)
5. Oculta progress bar y muestra mensaje de éxito/error

Usa QgsApplication.taskManager().addTask() para ejecutar el task. Dame el código completo y un diagrama de flujo en texto.
```

---

## Fase 4: Pruebas y depuración (2 días)

### 4.1 Pruebas unitarias con datos de ejemplo

**Prompt para Claude:**
```
Crea pruebas unitarias para `descarga_atom.py` usando pytest:

1. test_obtener_feed_valido(): mockea una respuesta HTTP exitosa
2. test_extraer_entradas_gerencia(): con un XML de ejemplo pequeño
3. test_buscar_municipio_existente()
4. test_buscar_municipio_no_existente()

Dame los mocks y los fixtures necesarios. Explícame cómo ejecutar pytest desde VSCode.
```

### 4.2 Prueba de integración con QGIS

**Prompt para Claude:**
```
Necesito una "prueba de humo" que ejecute mi plugin dentro de QGIS y verifique:

1. El diálogo se abre sin errores
2. Al seleccionar "Direcciones" y municipio "MADRID", descarga un archivo
3. La capa se carga en el proyecto

Dame un script de prueba que pueda ejecutar desde la consola Python de QGIS, sin usar el UI del plugin.
```

### 4.3 Manejo de errores específicos del servidor del Catastro

**Prompt para Claude:**
```
El servidor del Catastro es conocido por ser lento. Quiero manejar específicamente:

- timeout después de 60 segundos: reintentar 1 vez
- HTTP 503 (servicio no disponible): mensaje amigable y sugerir horarios de baja carga
- Área de extensión fuera de límites (solo si implemento WFS): no aplica a ATOM

Implementa una función `descarga_con_reintentos(url, max_intentos=2)` que use la lógica anterior. Dame el código y ejemplos de los mensajes de error que debo mostrar al usuario.
```

---

## Fase 5: Documentación y empaquetado (1 día)

### 5.1 Documentación con IA para usuarios

**Prompt para Claude:**
```
Genera un README.md para mi plugin con:

- Instalación desde archivo ZIP
- Uso paso a paso (con capturas de pantalla descritas en texto)
- Requisitos (QGIS 3.16+)
- Enlace a datos INSPIRE del Catastro
- Posibles errores y soluciones

Usa formato Markdown y emojis para hacerlo más amigable.
```

### 5.2 Generar el archivo ZIP para distribución

**Prompt para Claude:**
```
Dame un script Python (empacador.py) que:

1. Copie todos los archivos necesarios (excepto .venv, tests, .vscode)
2. Genere un archivo catastro_inspire.zip listo para instalar en QGIS
3. Verifique que metadata.txt tenga la versión correcta

Explícame cómo ejecutarlo y cómo instalar el ZIP resultante en QGIS (Plugins -> Instalar desde ZIP).
```

### 5.3 Lecciones aprendidas (para tu diario)

**Prompt para Claude:**
```
Revisa todo el código que hemos generado. Dime:

1. Las 3 decisiones de diseño más importantes que tomamos y por qué
2. Los 2 errores más comunes que podría encontrar un principiante al modificar este código
3. Qué mejoraría si tuviera 1 semana más (rendimiento, UX, etc.)

Esto irá a mi MEMORIAS_APRENDIZAJE.md. Quiero respuestas honestas y detalladas.
```

---

## Anexos

### A. Sistema de tracking de aprendizaje

**Diario diario (actualiza cada día):**

```markdown
# [YYYY-MM-DD] - [Tema del día]

## Qué intenté hacer
[Descripción de la tarea]

## Código que generó Claude
[Pega el código principal]

## Lo que NO entendía al principio
[Concepto que te resultó confuso]

## Cómo lo terminé entendiendo
[Explicación con tus palabras]

## Error que cometí (si aplica)
[Error específico y cómo lo arreglaste]

## Próximo paso
[Mañana voy a...]
```

**Checklist diario de calidad:**
- [ ] ¿Cada función tiene type hints y docstring?
- [ ] ¿Hay logging en lugar de print?
- [ ] ¿Los nombres de variables son descriptivos?
- [ ] ¿El código sigue PEP 8 (líneas <80 caracteres)?
- [ ] ¿He añadido una entrada a mi diario de aprendizaje?

### B. Recursos de respaldo

| Recurso | URL | Uso |
|---------|-----|-----|
| Plugin existente | https://github.com/sigdeletras/Spanish-Inspire-Catastral-Downloader | Referencia de arquitectura |
| Documentación PyQGIS | https://qgis.org/pyqgis/master/ | Consulta de clases específicas |
| INSPIRE Catastro | https://www.catastro.hacienda.gob.es/INSPIRE/ | Validar cambios en el servicio |

### C. Solución de problemas comunes

**Problema 1:** El entorno virtual no encuentra las librerías de QGIS
- **Prompt para Claude:** *"Mi VSCode no detecta qgis.core. ¿Cómo verifico si mi .venv está bien configurado? Dame comandos para inspeccionar el PYTHONPATH."*

**Problema 2:** El servidor del Catastro devuelve HTTP 504 Gateway Timeout
- **Prompt para Claude:** *"El servidor del Catastro es muy lento. ¿Cómo puedo implementar una cola de reintentos con backoff exponencial? Dame el código."*

**Problema 3:** QGIS se congela durante la descarga
- **Prompt para Claude:** *"Mi plugin congela QGIS aunque uso QgsTask. ¿Qué estoy haciendo mal? Aquí mi código: [pega código]"*

---

## Notas finales

**Fechas estimadas:** 10-12 días de trabajo a tiempo parcial (2-3 horas diarias)

**Métrica de éxito:** Un plugin funcional que, al seleccionar "Parcelas" y "MADRID", descargue y muestre las parcelas en menos de 5 minutos (depende del servidor).

**Mejor consejo para principiante:** Cuando algo no funcione, no pidas a Claude "arregla esto". Mejor pregunta: *"¿Por qué ocurre este error? Dame 3 posibles causas."* Así aprendes en lugar de solo parchear.

---

*Plan generado el [fecha actual]. Modifícalo según tu progreso real.*

---

**Para guardar este plan:** Selecciona todo el texto desde `# Plan de Trabajo` hasta el final, cópialo (Ctrl+C), abre VSCode o cualquier editor de texto, crea un nuevo archivo llamado `PLAN_TRABAJO.md` y pega el contenido (Ctrl+V). Ya tienes tu plan listo para usar.