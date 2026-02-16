---
description: "Auditoría SEO completa - Técnica + Renderizado en paralelo"
argument-hint: "[dominio] [ruta carpeta crawl SF+GSC]"
---

> **Configuración requerida:** en tu `CLAUDE.md`, define la ruta a Python 3.10+:
> - Windows: `PYTHON="C:\ruta\a\python.exe"`
> - Linux/Mac: `PYTHON="python3"`
> Todos los comandos de este skill usan `$PYTHON` como referencia.

# Auditoría SEO completa

Este workflow ejecuta en paralelo las dos auditorías SEO fundamentales y genera un informe unificado con todas las conclusiones.

## Qué ejecuta

1. **`/seo-tecnico`** — Auditoría técnica basada en crawl de Screaming Frog + GSC: indexación, rastreo, arquitectura, contenido, enlazado, errores
2. **`/seo-render-audit`** — Auditoría de renderizado JavaScript: cómo ve Googlebot el HTML vs el DOM renderizado, por tipo de plantilla

Ambas se lanzan en paralelo usando el Task tool con subagent_type=Bash o general-purpose para maximizar velocidad.

## Input esperado

El usuario proporciona:

1. **Dominio** (obligatorio): URL del sitio a auditar (ej: `https://inoxamedida.com/`)
2. **Ruta carpeta crawl** (obligatorio): carpeta con los exports de Screaming Frog + GSC

```
carpeta-cliente/
├── issues_reports/              # ~60 .xlsx de Screaming Frog
│   └── issues_overview_report.xlsx
├── internos_todo.xlsx           # Crawl completo SF con datos GSC
└── *backlinks*.csv              # Export de Ahrefs (opcional)
```

Si el usuario solo da el dominio sin carpeta de crawl, ejecutar solo `/seo-render-audit` y avisar que para la auditoría técnica necesita proporcionar los datos de crawl.

## Flujo obligatorio

### Paso 0: validar inputs

Antes de lanzar nada:

1. Confirmar que la carpeta de crawl existe y tiene los archivos necesarios:
   - Buscar `*internos*` (obligatorio)
   - Buscar `*issues_overview*` (obligatorio)
   - Buscar `issues_reports/` o xlsx sueltos
   - Buscar `*backlinks*` (opcional)

2. Confirmar que el dominio es accesible:
   ```bash
   curl -sL --max-time 10 -o /dev/null -w "%{http_code}" "DOMINIO"
   ```

Si falta algún archivo obligatorio, avisar al usuario y detenerse. NO continuar sin validar.

### Paso 1: lanzar ambas auditorías en paralelo

Usar el Task tool para lanzar DOS agentes simultáneos:

**Agente 1 — Auditoría técnica:**
```
Ejecutar /seo-tecnico con la carpeta de crawl proporcionada.
Seguir el flujo completo del skill: validar carpeta → preprocesar → analizar JSON → generar MD + Excel de evidencias.
Guardar outputs en la carpeta del crawl:
- auditoria_seo_tecnica.md
- evidencia_auditoria.xlsx
- resumen_auditoria.json
```

**Agente 2 — Auditoría de renderizado:**
```
Ejecutar /seo-render-audit en modo multi-plantilla (modo 2) con el dominio.
Auto-detectar tipos de plantilla navegando la web.
Guardar outputs en render_audit_[dominio]/ dentro de la carpeta del proyecto.
Generar reportes individuales + reporte_general.md
```

**IMPORTANTE:** Ambos agentes deben ejecutarse SIMULTÁNEAMENTE usando múltiples llamadas al Task tool en un solo mensaje.

### Paso 2: esperar y recopilar resultados

Esperar a que ambos agentes completen. Si uno falla, continuar con el que haya terminado.

Verificar que existen los archivos generados:
- De seo-tecnico: `auditoria_seo_tecnica.md`, `evidencia_auditoria.xlsx`
- De seo-render-audit: `render_audit_[dominio]/reporte_general.md`

### Paso 3: generar informe unificado

Leer ambos reportes MD y crear un informe conjunto:

```
carpeta-cliente/auditoria_seo_completa.md
```

Seguir la plantilla de abajo.

### Paso 4: presentar resultados

Mostrar al usuario:
- Estado general del sitio (combinando ambas auditorías)
- Top 5 problemas más críticos (de ambas auditorías)
- Rutas de todos los archivos generados

## Plantilla del informe unificado

El archivo `auditoria_seo_completa.md` debe seguir esta estructura:

```markdown
# Auditoría SEO completa - [DOMINIO]

- **Dominio:** [dominio]
- **Fecha:** [fecha]
- **Auditorías ejecutadas:** Técnica (crawl SF + GSC) + Renderizado (multi-plantilla)
- **URLs rastreadas:** [total del crawl]
- **Plantillas de renderizado auditadas:** [número]

---

## Estado general: [CRITICAL / WARNING / OK]

[3-4 frases que combinen el diagnóstico técnico y de renderizado. El lector debe entender inmediatamente si su web tiene problemas graves o no.]

---

## Resumen ejecutivo

### Auditoría técnica (crawl)

| Bloque | Estado | Problemas principales |
|--------|--------|----------------------|
| Indexación | [estado] | [resumen 1 línea] |
| Rastreo | [estado] | [resumen 1 línea] |
| Arquitectura | [estado] | [resumen 1 línea] |
| Errores | [estado] | [resumen 1 línea] |
| Enlazado | [estado] | [resumen 1 línea] |
| Contenido | [estado] | [resumen 1 línea] |

### Auditoría de renderizado

| Plantilla | Estado | Framework | Contenido en HTML crudo |
|-----------|--------|-----------|------------------------|
| Home | [estado] | [framework] | [sí/parcial/no] |
| Categoría | [estado] | [framework] | [sí/parcial/no] |
| Producto | [estado] | [framework] | [sí/parcial/no] |
| Blog | [estado] | [framework] | [sí/parcial/no] |

---

## Hallazgos cruzados

[Esta sección es el valor añadido del informe unificado. Identificar problemas que se manifiestan en AMBAS auditorías o donde una auditoría explica la otra.]

Ejemplos de hallazgos cruzados:
- "El crawl detecta 3.276 URLs con canonical duplicado. La auditoría de renderizado confirma que esto se debe a que el theme de PrestaShop inyecta un canonical por PHP y el módulo SEO inyecta otro por JavaScript."
- "El 93% del blog no tiene meta description en el crawl. La auditoría de renderizado muestra que la plantilla de blog no incluye meta description ni en HTML crudo ni tras renderizado: es un problema de plantilla, no de JS."
- "Hay URLs de filtros indexadas sin canonical. La auditoría de renderizado muestra que los filtros se generan con JavaScript (#fragmentos), pero el servidor también genera versiones con parámetros GET que sí se indexan."

### Hallazgo cruzado 1: [título]

- **Evidencia técnica:** [dato del crawl]
- **Evidencia de renderizado:** [dato del render audit]
- **Diagnóstico combinado:** [explicación que conecta ambos]
- **Impacto:** [consecuencia SEO]

### Hallazgo cruzado 2: [título]

[mismo formato]

---

## Plan de acción priorizado (unificado)

[Combinar las acciones de ambas auditorías en un solo plan priorizado. No duplicar acciones que se solapen.]

### Prioridad 1 - resolver inmediatamente

1. **[Acción]**
   - Origen: [Técnica / Renderizado / Ambas]
   - Problema: [descripción]
   - URLs afectadas: [N]
   - Solución: [pasos concretos]

2. **[Acción]**
   [...]

### Prioridad 2 - resolver a corto plazo

[...]

### Prioridad 3 - mejoras recomendadas

[...]

---

## Verificaciones manuales pendientes

| Verificación | Origen | Herramienta |
|-------------|--------|-------------|
| Core Web Vitals | Renderizado | PageSpeed Insights |
| Datos estructurados | Renderizado | Rich Results Test |
| Robots.txt (bloqueos) | Técnica | Inspección manual |
| GSC cobertura | Técnica | Google Search Console |
| EEAT completo | Técnica | Revisión manual |
| Sitemap en GSC | Técnica | Google Search Console |

---

## Archivos generados

### Auditoría técnica
| Archivo | Contenido |
|---------|-----------|
| `auditoria_seo_tecnica.md` | Informe técnico detallado (11 bloques) |
| `evidencia_auditoria.xlsx` | URLs filtradas por problema ([N] hojas) |
| `resumen_auditoria.json` | Métricas agregadas (uso interno) |

### Auditoría de renderizado
| Archivo | Contenido |
|---------|-----------|
| `render_audit_[dominio]/reporte_general.md` | Informe comparativo de renderizado |
| `render_audit_[dominio]/[plantilla]/reporte.md` | Reportes individuales por plantilla |
| `render_audit_[dominio]/[plantilla]/screenshot_*.png` | Capturas desktop y mobile |
| `render_audit_[dominio]/[plantilla]/*.html` | HTML crudo, Googlebot y renderizado |

### Informe unificado
| Archivo | Contenido |
|---------|-----------|
| `auditoria_seo_completa.md` | Este informe (resumen ejecutivo + hallazgos cruzados + plan unificado) |

---

*Auditoría generada con /auditoria-seo-completa — Workflow de auditoría SEO técnica + renderizado*
```

## Análisis obligatorios que NO pueden faltar

La auditoría técnica DEBE incluir estos análisis específicos. Si faltan, la auditoría está incompleta:

### 1. Robots.txt
- Descargar y analizar el robots.txt actual del dominio
- Listar qué bloquea y qué falta bloquear
- Proponer versión mejorada con reglas concretas

### 2. Parámetros de URL
- Inventario completo de todos los parámetros encontrados en el crawl
- Clasificación: paginación / filtros / tracking / sistema
- Cuántas URLs por parámetro, cuántas indexables, cuántas con GSC data
- Propuesta de bloqueo por parámetro
- **REGLA CRÍTICA — NUNCA bloquear paginaciones:** Los parámetros de paginación (`?page=`, `?p=`, `?paged=`, `/page/N`) NUNCA deben bloquearse en robots.txt ni marcarse con noindex. Las paginaciones contienen enlaces a productos/contenidos reales; bloquearlas impide que Google los rastree. SÍ se puede recomendar: validar el rango (404 si page > max), aumentar productos por página, mejorar enlazado de paginación.

### 3. Arquitectura y oportunidades
- Segmentos de URL agrupados con métricas (URLs, indexables, GSC)
- Análisis de profundidad: causa raíz y propuesta de aplanamiento
- Top URLs por tráfico vs su profundidad (deben estar a depth < 3)
- Propuesta de consolidación de categorías débiles
- Detección de canibalización (titles/H1 idénticos entre URLs diferentes)

### 4. Correlaciones entre bloques
- Conectar problemas entre bloques (ej: "las 1.403 URLs con H1 duplicado son las mismas que generan la profundidad > 50")
- No listar problemas aislados: buscar la causa raíz común

## Notas

- Si el script `preprocesar_auditoria.py` falla, el agente de seo-tecnico debe ejecutar el análisis manualmente (está documentado en el skill)
- Si Playwright no está instalado, el agente de seo-render-audit lo instalará automáticamente
- El informe unificado NO repite todo el contenido de los informes individuales; solo extrae resúmenes, hallazgos cruzados y el plan de acción combinado
- Los informes detallados están en sus archivos respectivos para quien quiera profundizar
- La carpeta `render_audit_[dominio]/` se crea dentro de la misma carpeta donde están los datos del crawl
- **Python**: usar siempre `$PYTHON` (la ruta configurada en tu CLAUDE.md)

$ARGUMENTS
