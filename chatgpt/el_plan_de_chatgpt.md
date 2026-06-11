# Plan de trabajo para desarrollo de plugin QGIS con IA

## Fase 0 — Configuración del entorno

### 0.1 Instalación base
- Instalar QGIS (versión estable)
- Instalar Python compatible con QGIS
- Instalar Visual Studio Code

### 0.2 Configuración VS Code
- Instalar extensiones:
  - Python (Microsoft)
  - Pylance
  - Ruff
  - Black Formatter
  - isort
  - GitLens
  - XML Tools
  - REST Client
  - Codeium

### 0.3 Configurar intérprete Python QGIS
- Seleccionar Python de instalación QGIS en VS Code:
  - Windows ejemplo:
    `C:\OSGeo4W\apps\Python39\python.exe`

### 0.4 Control de versiones
- Inicializar repositorio Git
- Crear estructura base del proyecto

### 0.5 Herramientas IA gratuitas
- Codeium (autocompletado)
- Continue.dev (chat con código)
- ChatGPT (diseño y revisión)
- Opcional: Ollama (modelos locales)

---

## Fase 1 — Diseño de arquitectura del plugin

### Objetivo
Definir estructura modular del plugin antes de programar.

### Interacción con IA
**Prompt:**
> Diseña la arquitectura de un plugin QGIS en Python que consuma servicios ATOM del Catastro (parcelas, direcciones y edificios).  
> Devuélvelo como arquitectura modular con separación de responsabilidades (UI, lógica, datos, GIS).  
> Incluye estructura de carpetas y clases principales.

### Resultado esperado
- Documento `docs/01_architecture.md`
- Estructura de carpetas del plugin
- Definición de clases base

---

## Fase 2 — Diseño del flujo ATOM

### Objetivo
Definir cómo se consumen los servicios INSPIRE ATOM.

### Interacción con IA
**Prompt:**
> Explica el flujo completo de consumo de un feed ATOM del Catastro (INSPIRE): desde la selección de municipio hasta la carga en QGIS como capa vectorial.  
> Incluye pasos técnicos, formatos de datos y posibles problemas.

### Resultado esperado
- Documento `docs/02_atom_flow.md`
- Diagrama lógico del proceso

---

## Fase 3 — Implementación del servicio ATOM

### Objetivo
Crear módulo de descarga y parsing.

### Interacción con IA
**Prompt:**
> Genera una clase Python llamada AtomService con responsabilidad de:  
> - descargar un feed ATOM  
> - parsear XML  
> - extraer enlaces de descarga  
> Usa buenas prácticas (PEP8, type hints, funciones pequeñas, docstrings).  
> Mantén el código desacoplado de QGIS.

### Resultado esperado
- `core/atom_service.py`
- Documentación asociada

---

## Fase 4 — Descarga de datos

### Interacción con IA
**Prompt:**
> Diseña una clase Downloader en Python para descargar archivos GML/ZIP desde URLs.  
> Debe incluir control de errores, reintentos y progreso básico.

### Resultado esperado
- `core/downloader.py`

---

## Fase 5 — Carga en QGIS

### Interacción con IA
**Prompt:**
> Genera una clase QgisLayerLoader que cargue archivos GML en QGIS usando QgsVectorLayer.  
> Debe manejar rutas locales y validar si la capa es válida.

### Resultado esperado
- `core/qgis_loader.py`

---

## Fase 6 — Interfaz del plugin

### Interacción con IA
**Prompt:**
> Diseña una interfaz Qt para un plugin QGIS con:  
> - selector de municipio  
> - selector de tipo de datos (parcelas, direcciones, edificios)  
> - botón de descarga  
> Devuélvelo en estructura PyQt compatible con QGIS.

### Resultado esperado
- UI básica del plugin

---

## Fase 7 — Integración del flujo completo

### Interacción con IA
**Prompt:**
> Integra los módulos AtomService, Downloader y QgisLayerLoader en un flujo único dentro de un plugin QGIS.  
> Mantén separación de responsabilidades y evita código monolítico.

### Resultado esperado
- Plugin funcional MVP

---

## Fase 8 — Documentación del sistema

### Interacción con IA
**Prompt:**
> Documenta este módulo de software como si fuera un proyecto profesional GIS.  
> Incluye: arquitectura, flujo de datos, decisiones técnicas y cómo ampliarlo.

### Resultado esperado
- `docs/03_system_overview.md`

---

## Fase 9 — Revisión y refactor

### Interacción con IA
**Prompt:**
> Analiza este código como un revisor senior de software GIS.  
> Detecta problemas de diseño, acoplamiento, legibilidad y rendimiento.  
> Propón mejoras concretas.

### Resultado esperado
- Código más limpio y mantenible

---

## Fase 10 — Evolución del plugin

### Interacción con IA
**Prompt:**
> Propón mejoras futuras para un plugin QGIS que consume datos ATOM del Catastro, incluyendo escalabilidad, nuevas fuentes INSPIRE y mejoras de rendimiento.

### Resultado esperado
- Roadmap del proyecto

