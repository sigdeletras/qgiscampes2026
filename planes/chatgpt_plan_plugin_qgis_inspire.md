# Plan de trabajo -- Plugin QGIS INSPIRE Catastro (ATOM Downloader)

## Fase 0 --- Configuración del entorno

### Objetivo

Preparar un entorno reproducible para desarrollo de plugins QGIS con IA
asistida.

### Herramientas

-   VS Code
-   Python (compatible con QGIS 3.x)
-   QGIS instalado (versión LTR recomendada)
-   Claude en VS Code
-   Git

### Extensiones VS Code

-   Python (Microsoft)
-   Pylance
-   Ruff
-   XML Tools
-   GitLens

### Prompt IA (configuración)

> Diseña una configuración óptima de VS Code para desarrollo de plugins
> QGIS en Python. Incluye estructura de proyecto, linters y buenas
> prácticas específicas de PyQGIS.

------------------------------------------------------------------------

## Fase 1 --- Exploración del sistema ATOM INSPIRE

### Objetivo

Entender la estructura real de los feeds ATOM del Catastro.

### Tareas

-   Analizar feeds:
    -   Parcelas
    -   Direcciones
    -   Edificios
-   Identificar jerarquía provincia → municipio → dataset
-   Detectar patrones de URLs GML

### Prompt IA

> Analiza estos feeds ATOM INSPIRE del Catastro y explica la estructura
> de navegación hasta llegar al GML municipal. Devuelve un esquema
> técnico y posibles casos de inconsistencia.

------------------------------------------------------------------------

## Fase 2 --- Diseño de arquitectura del plugin

### Objetivo

Definir arquitectura modular mantenible.

### Tareas

-   Separar UI, lógica ATOM y carga QGIS
-   Definir servicios
-   Definir flujo de datos

### Prompt IA

> Diseña la arquitectura de un plugin QGIS para descarga de datos
> INSPIRE del Catastro. Debe incluir módulos, responsabilidades, flujo
> de datos y separación estricta entre UI y lógica GIS.

------------------------------------------------------------------------

## Fase 3 --- MVP: descarga y carga de una capa

### Objetivo

Obtener un plugin funcional mínimo.

### Tareas

-   Resolver URL GML desde ATOM
-   Descargar fichero
-   Cargar en QGIS con QgsVectorLayer

### Prompt IA

> Implementa un servicio Python que resuelva la URL GML de un municipio
> desde un feed ATOM INSPIRE del Catastro. El código debe ser modular y
> reutilizable.

### Prompt IA (carga QGIS)

> Genera una función PyQGIS que cargue un fichero GML como capa
> vectorial y gestione errores comunes de CRS y geometría.

------------------------------------------------------------------------

## Fase 4 --- Interfaz de usuario (PyQt)

### Objetivo

Crear UI funcional en QGIS.

### Tareas

-   Selector de municipio
-   Selector de tipo de capa
-   Botón de descarga

### Prompt IA

> Diseña un diálogo PyQt para un plugin QGIS con selección de municipio
> y tipo de dato INSPIRE. Debe seguir buenas prácticas PyQGIS y
> separación MVC.

------------------------------------------------------------------------

## Fase 5 --- Robustez y asincronía

### Objetivo

Evitar bloqueos en QGIS.

### Tareas

-   QThread o QgsTask
-   barra de progreso
-   cancelación de descarga
-   logging

### Prompt IA

> Refactoriza este flujo de descarga para ejecutarse de forma asíncrona
> en QGIS usando QgsTask. Incluye barra de progreso y manejo de
> cancelación.

------------------------------------------------------------------------

## Fase 6 --- Optimización y almacenamiento local

### Objetivo

Mejorar rendimiento y reutilización.

### Tareas

-   Cache local
-   Conversión a GeoPackage
-   Evitar descargas repetidas

### Prompt IA

> Diseña un sistema de cache para un plugin QGIS que almacene datos
> INSPIRE descargados localmente en GeoPackage y evite descargas
> redundantes.

------------------------------------------------------------------------

## Fase 7 --- Empaquetado y publicación

### Objetivo

Preparar plugin para distribución.

### Tareas

-   metadata.txt
-   estructura plugin QGIS
-   validación
-   documentación

### Prompt IA

> Revisa este plugin QGIS y genera la estructura lista para publicación
> en el repositorio oficial. Incluye metadata.txt y checklist de
> validación.

------------------------------------------------------------------------

## Fase 8 --- Documentación y aprendizaje

### Objetivo

Convertir el desarrollo en conocimiento reutilizable.

### Tareas

-   docs técnicos
-   decisiones de diseño
-   log de errores

### Prompt IA

> Convierte este módulo del plugin QGIS en documentación técnica
> explicativa para un desarrollador GIS junior. Incluye flujo de datos,
> decisiones y posibles mejoras.
