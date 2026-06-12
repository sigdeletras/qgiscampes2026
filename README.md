# QGIS CAMP ESPAÑA 2026

[https://www.qgis.es/talk/2026-05-qgis-camp-espana-2026-madrid/](https://www.qgis.es/talk/2026-05-qgis-camp-espana-2026-madrid/)

## Materiales de la presentación "Vibe coding para QGIS: oportunidades y riesgos de crear plugins con IA"

### Tips sobre contrucciones de instrucciones
- Añadir contexto y hacer referencia en vez de repetirlo
- Evitar ser redundante y omitir frases de relleno
- Fusionar condiciones
- Evitar ambigüedades
- Reformular preguntas muy abiertas abiertas y añadir ejemplos concretos
- Normalizar terminología
- Estructurar consultas
- Definir formatos exactos de salida

### Prompt 01 o "Sujétame el cubata…"

```
Soy usuario de QGIS con conocimientos básicos de Python (más experiencia en web). Quiero crear mi primer complemento para descargar datos INSPIRE de Catastro por municipio y cargarlos en QGIS.

Fuentes ATOM:
- Parcelas: https://www.catastro.hacienda.gob.es/INSPIRE/CadastralParcels/ES.SDGC.CP.atom.xml
- Direcciones: https://www.catastro.hacienda.gob.es/INSPIRE/Addresses/ES.SDGC.AD.atom.xml
- Edificios: https://www.catastro.hacienda.gob.es/INSPIRE/buildings/ES.SDGC.BU.atom.xml

El usuario elegiría municipio y tipo de datos; el plugin descarga y carga la capa.

Valora su viabilidad de forma concisa, adaptada a mi nivel.
```

### Prompt 02 o "La caja de herramientas"
```
¿Qué herramientas de programación asistida por IA me recomiendas para desarrollar un complemento QGIS en Python? Diferencia entre gratuitas y de pago.
```

### Prompt 03 o "Sois todos iguales"
```
¿Qué LLMs recomiendas para desarrollar este complemento QGIS? ¿O conviene alternar varios según la tarea?
```
### Prompt 04 o "¿Código de KK?"
```
Siendo principiante, ¿cómo puedo asegurar que el código que genere con IA sea legible, mantenible y de calidad aceptable para otros desarrolladores?
```
### Prompt 05 o "Los preliminares"

(Claude) 

```
Voy a usar VSCode con Claude. Dame una guía de configuración. Dime también extensiones de utilidad
```

(Cursor)

```
Voy a usar Cursor. Dame una guía de configuración
```

### Prompt 06 o "Es duro pedir…pero mejor es delegar"
```
¿Qué son los "agentic coding" o agent skills en IA? ¿Vale la pena usarlos para este proyecto y cómo?
```
### Prompt 07 o "Documentar ≠ Sufrir"
```
¿Cómo puedo usar la IA para documentar el complemento y el proceso de desarrollo con VSCode, de forma que también me sirva para aprender?
```
### Prompt 08 o "¿Qué se me está escapando?"
```
Antes de continuar ¿qué mas debería tener en cuenta?
```
### Prompt 09 o "*Habemus* un plan"
```
Con todo lo anterior, genera un plan de trabajo en Markdown para desarrollar el complemento. Incluye:
- Fase 0: configuración del entorno de la herramienta
- Una fase por etapa de desarrollo
- Para cada interacción con la IA, el prompt exacto a lanzar

Entrégalo como archivo .md descargable.
```

## Enlaces a la respuesta de los asistentes

**Usando suscripciones freemium** 

- [Gemini - Gemini 3.5 Flash](https://gemini.google.com/share/5bccfa83d1d9)
- [Deepseek - DeepSeek-V3](https://chat.deepseek.com/share/uya53gopxq7zf5rwtk)
- [ChatGPT - GPT-4o mini](https://chatgpt.com/share/6a2aedfb-981c-83eb-987d-a01952e9aded)
- [Claude - Sonet 4.5 Bajo](https://claude.ai/share/29b0c6f7-3913-47e7-90cb-f4ca216b5bdc)

## Recursos

- VSCODE
- Winsurf
- Cursor
- Codemium
- Plging Builder
- Pluging Reload

