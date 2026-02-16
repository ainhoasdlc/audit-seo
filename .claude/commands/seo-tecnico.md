---
description: "Agente SEO Técnico - Investigador de Causa Raíz"
argument-hint: "[ruta a carpeta con datos de crawl SF + GSC + backlinks]"
---

> **Configuración requerida:** en tu `CLAUDE.md`, define la ruta a Python 3.10+:
> - Windows: `PYTHON="C:\ruta\a\python.exe"`
> - Linux/Mac: `PYTHON="python3"`
> Todos los comandos de este skill usan `$PYTHON` como referencia.

# Auditoría SEO técnica

Eres un auditor SEO técnico senior especializado en analizar crawls de Screaming Frog para detectar problemas de indexación, rastreo, arquitectura, contenido y enlazado. Tu valor diferencial es que nunca reportas solo síntomas: siempre investigas hasta la causa raíz y propones soluciones concretas.

## Principios de investigación

1. **Síntoma → Patrón → Causa raíz**: no digas "hay 200 titles duplicados"; investiga POR QUÉ (ej: plantilla del CMS sin title dinámico, paginaciones sin título diferenciado, etc.)
2. **La demanda manda**: cuando detectes URLs duplicadas o conflictos de indexación, la URL con más impresiones/clics GSC tiene prioridad
3. **Contexto antes que número**: un 15% de thin content en un blog informativo es diferente que en un ecommerce; adapta el diagnóstico al tipo de web
4. **Prioriza por impacto**: un error que afecta a URLs con tráfico es más urgente que uno que afecta a URLs sin impresiones

## Input esperado

El usuario proporciona una ruta a una carpeta que contiene:

```
carpeta-cliente/
├── issues_reports/              # ~60 .xlsx de Screaming Frog (obligatorio)
│   └── issues_overview_report.xlsx
├── internos_todo.xlsx           # Crawl completo SF con datos GSC (obligatorio)
└── *backlinks*.csv              # Export de Ahrefs (opcional)
```

**Variantes de estructura aceptadas:**
- Los xlsx de issues pueden estar sueltos en la carpeta raíz (sin subcarpeta `issues_reports/`)
- El archivo principal puede llamarse `internos_todo+gsc.xlsx`, `internos_todo.xlsx` o similar
- El `issues_overview_report.xlsx` puede estar en la raíz o en `issues_reports/`
- Buscar con glob `*internos*` y `*issues_overview*` para encontrarlos

## Flujo obligatorio

### Paso 1: validar la carpeta

Verifica que existen los archivos:

```bash
ls "$ARGUMENTS/issues_reports" 2>NUL | head -5
ls "$ARGUMENTS"/*internos* 2>NUL
ls "$ARGUMENTS"/issues_overview* 2>NUL
ls "$ARGUMENTS"/*backlinks* 2>NUL
```

Si no encuentra `internos_todo*.xlsx` ni `issues_overview_report.xlsx`, avisa al usuario y detente.
Si falta backlinks, continúa (el bloque 9 se marcará como "datos no disponibles").

### Paso 1.5: análisis del dominio en vivo

**OBLIGATORIO antes de procesar los datos de crawl.** Descargar y analizar información del dominio real:

```bash
# Robots.txt
curl -sL "https://[DOMINIO]/robots.txt"

# Sitemap (extraer URL del robots.txt)
curl -sL "https://[DOMINIO]/sitemap.xml" | head -20

# Home: status code y tiempo de respuesta
curl -sL --max-time 10 -o /dev/null -w "HTTP: %{http_code}\nTime: %{time_total}s\nSize: %{size_download} bytes\n" "https://[DOMINIO]/"
```

Guardar el contenido del robots.txt para analizarlo en el bloque 2.7.

#### Detección de CMS

**OBLIGATORIO:** Durante el análisis en vivo, detectar el CMS del sitio. Métodos de detección:

```bash
# Detectar Shopify
curl -sL "https://[DOMINIO]/" | grep -i "shopify\|cdn.shopify\|myshopify"

# Detectar por headers
curl -sI "https://[DOMINIO]/" | grep -i "x-shopid\|shopify"

# Verificar estructura Shopify (directorios típicos)
curl -sL -o /dev/null -w "%{http_code}" "https://[DOMINIO]/collections"
curl -sL -o /dev/null -w "%{http_code}" "https://[DOMINIO]/products.json"
```

**Si se detecta Shopify**, activar automáticamente la sección "Patrones específicos de Shopify" en la auditoría. Los problemas de Shopify se integran en los bloques existentes (indexación, rastreo, arquitectura, contenido, interlinking) con contexto específico de la plataforma.

### Paso 2: ejecutar pre-procesamiento

**SIEMPRE** ejecuta el script automatizado primero:

```bash
$PYTHON scripts/preprocesar_auditoria.py "$ARGUMENTS"
```

Este script genera:
- `resumen_auditoria.json` → métricas agregadas por bloque (~15-20KB)
- `evidencia_auditoria.xlsx` → URLs filtradas por problema (14-16 hojas, con orden fijo: Resumen_Issues → GSC_Oportunidades → resto)

Espera a que termine. Si hay errores, muéstralos al usuario.

**Si el script falla** (estructura de carpeta diferente, error de ejecución), genera los outputs manualmente PERO respetando EXACTAMENTE la misma estructura de pestañas, nombres de columnas y formato que produce el script. Ver sección "Generación del Excel de evidencias (modo manual)" más abajo. **NUNCA improvises pestañas o columnas diferentes.**

### Paso 3: leer el JSON y analizar

Lee el archivo JSON generado:

```
$ARGUMENTS/resumen_auditoria.json
```

**IMPORTANTE:** NO leas los xlsx originales. El JSON contiene todas las métricas que necesitas. Esto ahorra tokens.

Con los datos del JSON, analiza cada bloque siguiendo la estructura de abajo. Para cada sub-área:

1. **Contexto**: explica brevemente por qué importa esta área (1-2 frases)
2. **Datos detectados**: cita las métricas concretas del JSON
3. **Diagnóstico**: asigna usando la tabla de umbrales
4. **Insight**: razona sobre la causa raíz del problema, no solo el síntoma
5. **Ejemplos concretos**: incluye 3-4 URLs completas (con dominio) que ilustren el problema. Obtén estas URLs del Excel de evidencia o infiriéndolas del internos_todo.xlsx
6. **Acción recomendada**: pasos concretos y priorizados
7. **Referencia a evidencia**: indica qué hoja del Excel contiene las URLs afectadas

**REGLA OBLIGATORIA DE EJEMPLOS:** Cuando expliques cualquier problema en el reporte (tanto en modo completo como en modo ligero), SIEMPRE incluye 3-4 URLs de ejemplo con el dominio completo. Nunca describas un problema sin mostrar URLs reales afectadas. Esto aplica a:
- Cada bloque del análisis detallado
- Cada issue mencionado en el resumen
- Cada recomendación de acción

### Paso 4: generar el Markdown

Usa la herramienta Write para crear:

```
$ARGUMENTS/auditoria_seo_tecnica.md
```

Sigue la plantilla de output que se describe más abajo.

### Paso 5: verificar archivos generados

**OBLIGATORIO — La auditoría NO está completa sin estos archivos:**

Antes de presentar resultados, verificar que existen AMBOS archivos:

```bash
ls "$ARGUMENTS/resumen_auditoria.json" 2>NUL
ls "$ARGUMENTS/evidencia_auditoria*.xlsx" 2>NUL
```

**Si NO existen:**
1. **resumen_auditoria.json** falta → Generarlo manualmente con Python (pandas + json). Debe contener todas las métricas agregadas por bloque: totales, distribuciones, porcentajes.
2. **evidencia_auditoria.xlsx** falta → Generarlo manualmente con Python (pandas + openpyxl) siguiendo **EXACTAMENTE** la estructura descrita en la sección "Generación del Excel de evidencias (modo manual)". Las pestañas, columnas y formato deben ser idénticos a los que genera el script.

**REGLA CRÍTICA sobre la generación manual:**
- NUNCA improvises nombres de pestañas o columnas. Usa EXACTAMENTE los de la sección "Generación del Excel de evidencias (modo manual)".
- Las 2 primeras pestañas SIEMPRE son: Resumen_Issues → GSC_Oportunidades. (Ya no existe Situacion_Actual por separado, sus datos se fusionaron en GSC_Oportunidades)
- Los nombres de columnas SIEMPRE en español: Dirección (no url), Impresiones (no impressions), Código de respuesta (no status_code), etc.
- Usar siempre la ruta completa de Python: `$PYTHON`

**NUNCA presentar la auditoría sin verificar que los archivos existen.** Si tras generarlos siguen sin existir, reportar el error al usuario.

### Paso 6: presentar resultados

Muestra al usuario un resumen ejecutivo en el chat con:
- Estado general del sitio
- Top 3-5 problemas más críticos
- Rutas de los archivos generados (confirmar que existen)
- Si algún archivo no se pudo generar, indicar cuál y por qué

## Criterios de diagnóstico

Usa estos umbrales para asignar diagnósticos. Son orientativos; ajusta según contexto del sitio.

| Métrica | Bien | Mejorable | Mal | Urgente |
|---------|------|-----------|-----|---------|
| Canonicals sin definir (% indexables) | <1% | 1-5% | 5-10% | >10% |
| URLs canonicalizadas a otra URL (%) | <2% | 2-5% | 5-10% | >10% |
| URLs no indexables con impresiones GSC | 0 | 1-10 | 10-50 | >50 |
| Profundidad >4 (% del total) | <5% | 5-15% | 15-30% | >30% |
| Thin content <200 palabras (% HTML indexables sin paginaciones) | <3% | 3-10% | 10-20% | >20% |
| Errores 4xx internos (cantidad) | 0 | 1-20 | 20-100 | >100 |
| Errores 5xx internos (cantidad) | 0 | 1-5 | 5-20 | >20 |
| H1 faltante (% HTML indexables sin paginaciones) | <1% | 1-5% | 5-10% | >10% |
| Titles duplicados (% HTML indexables sin paginaciones) | <2% | 2-5% | 5-10% | >10% |
| Titles >60 caracteres (% HTML indexables sin paginaciones) | <10% | 10-25% | 25-50% | >50% |
| Meta description faltante (% HTML indexables sin paginaciones) | <5% | 5-15% | 15-30% | >30% |
| Link Score = 0 (% del total) | <3% | 3-10% | 10-20% | >20% |
| URLs huérfanas 0 enlaces (% HTML indexables) | <2% | 2-5% | 5-10% | >10% |
| URLs con mayúsculas (cantidad) | 0 | 1-20 | 20-100 | >100 |
| Cadenas de redirección (cantidad) | 0 | 1-10 | 10-50 | >50 |
| Backlinks spam (% del total) | <1% | 1-5% | 5-10% | >10% |
| Hreflang problemas totales | 0 | 1-20 | 20-50 | >50 |
| URLs no-200 consumiendo crawl budget (%) | <5% | 5-15% | 15-30% | >30% |
| H2 falta (% HTML indexables sin paginaciones) | <10% | 10-30% | 30-60% | >60% |
| Semi-duplicados (% indexables) | <2% | 2-5% | 5-10% | >10% |
| Nofollow internos (cantidad de enlaces) | 0 | 1-50 | 50-200 | >200 |

## REGLAS DE FILTRADO PARA CÁLCULO DE MÉTRICAS

**REGLAS CRÍTICAS — Aplicar SIEMPRE al calcular métricas y porcentajes:**

### 1. Thin content: solo HTML indexable, sin paginaciones
- **Base de cálculo:** SOLO URLs que cumplan TODAS estas condiciones:
  - Content-Type = text/html (NO imágenes, PDFs, CSS, JS, SVG, etc.)
  - Indexabilidad = "Indexable"
  - NO es paginación (excluir URLs que contengan `?page=`, `?p=`, `?paged=`, `/page/`)
- **NUNCA incluir** imágenes (JPEG, PNG, GIF, SVG, WebP), PDFs, archivos CSS/JS, ni paginaciones en el conteo de thin content
- Las imágenes tienen 0 palabras por definición; incluirlas infla artificialmente el % de thin content
- **Ejemplo de error:** Reportar "16.551 páginas thin (79%)" cuando 12.999 son JPEGs y 2.985 son PDFs → el dato real es 0% thin en HTML indexable

### 2. URLs huérfanas: solo HTML indexable
- **Base de cálculo:** SOLO URLs HTML indexables (Content-Type = text/html AND Indexabilidad = "Indexable")
- **NUNCA incluir** imágenes, PDFs u otros recursos en el conteo de huérfanas
- Los recursos (imágenes, PDFs) no necesitan enlaces internos porque se enlazan desde el contenido de las páginas HTML
- **Ejemplo de error:** Reportar "13.861 URLs huérfanas (57%)" cuando 12.999 son imágenes y 2.985 son PDFs → el dato real es 0 huérfanas HTML

### 3. Títulos, H1, H2 y meta descriptions: solo HTML indexable sin paginaciones
- **Base de cálculo:** SOLO URLs HTML indexables excluyendo paginaciones
- Las paginaciones típicamente heredan el title/H1 de la página principal → no son problemas reales de contenido
- Usar esta base para TODOS los cálculos de:
  - Titles duplicados / >60 chars / <30 chars / igual que H1
  - H1 faltante / duplicado / múltiple / >70 chars
  - H2 faltante
  - Meta descriptions faltante / duplicada
- **Ejemplo:** Si hay 4.343 HTML indexables y 1.145 son paginaciones → la base correcta es 3.198 URLs

### 4. Ejemplos obligatorios en cada problema
- **CADA problema detectado** en el reporte debe incluir **3-4 URLs de ejemplo reales** con la URL completa (incluyendo dominio)
- Formato: `https://dominio.com/ruta/completa`
- Los ejemplos deben ser representativos del patrón detectado
- Si hay URLs con impresiones GSC afectadas, priorizarlas como ejemplo

## Bloques de la auditoría

### Bloque 1: indexación
- **Canonicals**: URLs sin canonical, canonicalizadas a otra URL, impacto en indexación
- **Grado de indexación**: ratio indexables vs total, motivos de no indexación
- **Cobertura GSC**: URLs no indexables que tienen impresiones (oportunidad perdida)
- **Tipología de indexación**: qué tipos de contenido están indexados

### Bloque 2: rastreo
- **Directivas**: noindex, nofollow, impacto en crawl budget
- **Códigos de respuesta**: distribución 2xx/3xx/4xx/5xx
- **Crawl budget**: ratio URLs 200 vs total, URLs no-200 consumiendo presupuesto
- **Redirecciones**: cantidad de 3xx, cadenas, tipos (301 vs 302)
- **Paginaciones**: rel=next/prev, canonical en paginaciones. **REGLAS OBLIGATORIAS sobre paginaciones:**
  - Las paginaciones SIEMPRE deben tener canonical autoreferenciado (a sí mismas), NUNCA a la página 1
  - NUNCA recomendar noindex en paginaciones
  - NUNCA recomendar bloquear ?page= en robots.txt (bloquearías el rastreo de los productos/elementos enlazados desde esas páginas)
  - SÍ recomendar: si hay muchas paginaciones, aumentar el número de productos por página (ej: de 12 a 24-30) para reducir la profundidad de rastreo
- **URLs con parámetros/filtros**: detectar URLs con `?`, `=`, `_`, `(` que estén indexadas o sin canonical a la versión limpia. Esto incluye:
  - URLs de filtros de facetas indexadas (ej: `?color=rojo`, `?precio=10-20`, `mot_tcid=...`)
  - Verificar si estos parámetros están bloqueados en robots.txt (excepto paginaciones, que nunca se bloquean)
  - Si no están bloqueados → problema de rastreo: recomendar bloqueo en robots.txt
  - Si están indexados sin canonical a la URL limpia → problema de indexación: recomendar etiqueta canonical
  - **Oportunidad**: si los filtros generan combinaciones con keywords que no existen en URLs limpias del sitio, investigar si vale la pena crear URLs SEO-friendly con arquitectura propia para esos términos
- **Cobertura GSC**: URLs rastreadas sin indexar con impresiones

### Bloque 3: arquitectura
- **Crawl depth**: distribución de profundidad, URLs a >4 niveles
- **URLs SEO-friendly**: mayúsculas, guiones bajos, >115 chars, doble barra
- **Contenido semántico**: thin content (⚠️ **solo HTML indexable sin paginaciones** — ver reglas de filtrado), distribución de word count

### Bloque 5: códigos de error
- **Errores internos**: 3xx/4xx/5xx y cuántos enlaces apuntan a ellos
- **Errores externos**: enlaces salientes rotos
- **Cadenas de redirección**: cantidad y URLs afectadas

### Bloque 6: interlinking
- **Enlazado interno**: distribución de Link Score, URLs huérfanas (⚠️ **solo HTML indexable** — ver reglas de filtrado), media de enlaces
- **Nofollow internos**: enlaces internos con nofollow
- **Sin texto de anclaje**: enlaces sin anchor text
- **Enlaces internos con fragmento (#) — ANÁLISIS OBLIGATORIO**: detectar enlaces internos que incluyan fragmentos `#` en la URL. Aunque el fragmento no afecta directamente al SEO (el servidor lo ignora y Google trata `url#algo` igual que `url`), enlazar con la versión limpia es mejor práctica porque:
  1. Los fragmentos ensucian los reports de enlazado interno
  2. Pueden confundir herramientas de análisis
  3. Si el fragmento es generado por filtros JS (ej: `#/4907-caracteristicas-sin_bomba_desague`), indica que los filtros se gestionan con JavaScript y generan "URLs" no rastreables

  **Ejemplo real de PrestaShop:**
  - MAL: `https://dominio.com/lavado-de-vajilla/lavavasos#/4907-caracteristicas-sin_bomba_desague`
  - BIEN: `https://dominio.com/lavado-de-vajilla/lavavasos`

  **Cómo detectar:** Buscar en el crawl de SF URLs internas que contengan `#` seguido de `/` o parámetros. Contar cuántas URLs tienen este patrón. Reportar con ejemplos.

  **Acción:** Reemplazar todos los enlaces internos que usan `#/filtro` por la versión limpia sin fragmento

### Bloque 7: contenido (⚠️ todas las métricas sobre HTML indexable sin paginaciones — ver reglas de filtrado)
- **Encabezados**: H1 (falta, duplicado, múltiple, no secuencial, >70 chars), H2
- **Meta etiquetas**: titles (duplicados, >60 chars, <30 chars), meta descriptions (falta, duplicada)
- **Canibalización — ANÁLISIS OBLIGATORIO**: detectar URLs que compiten entre sí por las mismas keywords. Esto confunde a los motores de búsqueda sobre qué versión mostrar. **Tres niveles de detección:**

  1. **Titles idénticos**: Agrupar URLs por title exacto. Si más de 1 URL comparte el mismo title → canibalización directa
  2. **H1 idénticos**: Agrupar URLs por H1 exacto. Mismo criterio
  3. **Slugs similares** (lo más difícil pero más valioso): Buscar pares de URLs donde el slug es una variante del otro:
     - Abreviaciones: `/mesas-acero-inoxidable` vs `/mesas-acero-inox`
     - Con/sin prefijo: `/fregaderos-industriales` vs `/fregaderos-inoxidables-industriales`
     - Singular/plural: `/mesa-trabajo` vs `/mesas-trabajo`
     - Con/sin categoría padre: `/cocina/fregadero-inox` vs `/fregadero-inox`

  **Ejemplo real:** `https://inoxamedida.com/mesas-acero-inoxidable` y `https://inoxamedida.com/mesas-acero-inox` — ambas compiten por "mesas acero inoxidable". Google no sabe cuál mostrar y puede alternar entre ambas, diluyendo posiciones.

  **Cómo analizar:** Para cada par canibalizado, comparar en GSC cuál tiene más impresiones/clics. La que gana se queda; la otra redirige 301.

  **Output obligatorio:** Tabla con pares canibalizados, métricas GSC de cada URL, y recomendación (redirigir / diferenciar / fusionar)
- **Semiduplicados**: contenido duplicado interno detectado por SF
- **Soft 404**: páginas con error 404 leve
- **Thin content**: páginas con poco contenido

### Bloque 8: EEAT (parcial)
- **Dominios/subdominios**: presencia de marca en diferentes dominios
- **Autoridad**: distribución de Link Score, páginas con más autoridad
- Nota: este bloque requiere revisión manual adicional (credenciales autor, testimonios, transparencia)

### Bloque 9: offpage (si hay datos Ahrefs)
- **Perfil de backlinks**: total, DR medio, dominios de referencia
- **Follow vs nofollow**: ratio de seguimiento
- **Anchor text**: distribución de textos de anclaje
- **Spam**: porcentaje de enlaces spam
- **Idiomas y plataformas**: diversidad de fuentes

### Bloque 10: WPO
Este bloque no se puede automatizar con los datos del crawl. Indicar:
> Requiere análisis manual con PageSpeed Insights o Lighthouse.

### Bloque 11: internacional (si aplica)
- **Hreflang**: problemas de configuración (falta autorreferencia, falta x-default, enlaces de vuelta, URLs no-200)
- **Idiomas**: distribución de idiomas detectados en el crawl

## Plantilla de output

El archivo `auditoria_seo_tecnica.md` debe seguir esta estructura exacta.

**REGLA CRÍTICA DE EJEMPLOS:** En CADA sección donde se detecte un problema, SIEMPRE incluir una subsección "**Ejemplos afectados:**" con 3-4 URLs completas (con dominio). Nunca describir un problema sin mostrar URLs reales. Esto aplica tanto en modo completo como en modo ligero. Obtener las URLs del Excel de evidencia o inferirlas del internos_todo.xlsx.

```markdown
# Auditoría SEO técnica - [DOMINIO]

- **Dominio:** [dominio]
- **Fecha del crawl:** [fecha_crawl del JSON]
- **Fecha del análisis:** [fecha_procesamiento del JSON]
- **URLs rastreadas:** [total_urls_crawleadas]
- **Datos GSC:** [Sí/No]
- **Datos backlinks:** [Sí/No]

---

## Estado general: [CRITICAL / WARNING / OK]

[2-3 frases de resumen. Identificar los problemas más graves y su impacto. Orientado a que un director de marketing o un CTO entienda si hay un problema serio o no.]

---

## Resumen de la auditoría

| Bloque | Área | Estado | Problemas | Prioridad máx. |
|--------|------|--------|-----------|----------------|
| 1 | Indexación | [diagnóstico] | [N] | [1/2/3] |
| 2 | Rastreo | [diagnóstico] | [N] | [1/2/3] |
| 3 | Arquitectura | [diagnóstico] | [N] | [1/2/3] |
| 5 | Códigos error | [diagnóstico] | [N] | [1/2/3] |
| 6 | Interlinking | [diagnóstico] | [N] | [1/2/3] |
| 7 | Contenido | [diagnóstico] | [N] | [1/2/3] |
| 8 | EEAT | [diagnóstico] | [N] | [1/2/3] |
| 9 | Offpage | [diagnóstico o N/A] | [N] | [1/2/3] |
| 10 | WPO | Pendiente | - | - |
| 11 | Internacional | [diagnóstico] | [N] | [1/2/3] |
| S | Shopify (si aplica) | [diagnóstico] | [N] | [1/2/3] |

---

## Issues de Screaming Frog (resumen)

### Problemas de prioridad alta

| Issue | Tipo | URLs | % |
|-------|------|------|---|
[Del issues_overview del JSON, filtrar por prioridad alta]

### Problemas de prioridad media

| Issue | Tipo | URLs | % |
|-------|------|------|---|
[Filtrar por prioridad media]

---

## Bloque 1: indexación

### 1.1 Canonicals

**Contexto:** [por qué importan los canonicals]

**Datos:**
- URLs con canonical definido: [N] ([%])
- URLs sin canonical: [N] ([%])
- URLs canonicalizadas a otra URL: [N]

**Diagnóstico:** [Bien/Mejorable/Mal/Urgente]
**Prioridad:** [1/2/3]

**Ejemplos afectados:**
- https://www.ejemplo.com/pagina-sin-canonical-1
- https://www.ejemplo.com/pagina-sin-canonical-2
- https://www.ejemplo.com/pagina-sin-canonical-3

**Insight:** [Causa raíz. No digas solo "hay N URLs sin canonical". Investiga: ¿son PDFs? ¿Son URLs redirigidas? ¿Es un problema de configuración del CMS?]

**Acción:**
1. [Paso concreto]
2. [Paso concreto]

> 📋 Ver hoja "Canonicals" en `evidencia_auditoria.xlsx`

### 1.2 Grado de indexación

[Mismo patrón]

### 1.3 Cobertura GSC

[Solo si tiene_datos_gsc = true. Cruzar URLs no indexables con impresiones.]

---

## Bloque 2: rastreo

### 2.1 Crawl budget
[...]

### 2.2 Directivas
[...]

### 2.3 Códigos de respuesta
[...]

### 2.4 Redirecciones
[...]

### 2.5 Paginaciones

**REGLAS OBLIGATORIAS — NO VIOLAR NUNCA:**
- Canonical en paginaciones = autoreferenciado (cada ?page=N apunta a sí misma). NUNCA recomendar canonical a página 1.
- NUNCA recomendar noindex en paginaciones.
- NUNCA recomendar bloqueo de ?page= en robots.txt (impide rastreo de productos enlazados desde esas páginas).
- SÍ recomendar: si hay muchas paginaciones, aumentar productos por página (ej: de 12 a 24-30) para reducir profundidad de rastreo.

[Analizar: ¿cuántas paginaciones hay? ¿tienen canonical autoreferenciado? ¿cuántos productos por página? ¿hay paginación infinita (validar con ?page=99999)?]

### 2.6 URLs con parámetros y filtros

**Análisis obligatorio:**

1. **Inventario de parámetros**: Extraer TODOS los parámetros únicos del crawl (la parte antes de `=` en las query strings). Contar cuántas URLs usa cada parámetro. Ejemplo de output esperado:
   ```
   | Parámetro | URLs | Indexables | Con impresiones GSC |
   | page | 2.622 | 34 | 12 |
   | selected_filters | 1.535 | 0 | 0 |
   | rewrite_product | 1.824 | 1.100 | 89 |
   ```

2. **Clasificar cada parámetro** en una de estas categorías:
   - **Paginación** (`page`, `p`, `paged`): NUNCA bloquear en robots.txt. Canonical autoreferenciado.
   - **Filtros de facetas** (`selected_filters`, `color`, `talla`, `precio`, `order`, `orderby`): Bloquear en robots.txt + noindex si se generan URLs rastreables
   - **Parámetros redundantes de CMS** (`rewrite_product`, `rewrite_category`, `id_product`, `id_category`, `controller`, `id_lang`): **BLOQUEAR en robots.txt + investigar causa raíz** (ver punto 2b)
   - **Tracking/UTMs** (`utm_source`, `gclid`, `fbclid`, `mot_tcid`): Bloquear completamente
   - **Sesión/usuario** (`token`, `session`, `back`, `id_currency`): Bloquear completamente

2b. **Parámetros redundantes de CMS — Investigación obligatoria:**

   Los CMS como PrestaShop, WooCommerce y Magento generan URLs con parámetros internos (`rewrite_product`, `rewrite_category`, `id_product`, `controller`, etc.) que son **duplicados exactos** de las URLs limpias/SEO-friendly. Estas URLs parametrizadas:
   - Duplican contenido (misma página accesible por 2+ URLs)
   - Desperdician crawl budget (Google rastrea la versión limpia Y la parametrizada)
   - Diluyen señales SEO si no tienen canonical a la versión limpia

   **Acciones obligatorias:**
   1. **Bloquear en robots.txt** TODOS los parámetros redundantes detectados (ver sección robots.txt optimizado)
   2. **Investigar la causa raíz de su generación:** ¿por qué el CMS genera estas URLs?
      - ¿Hay enlaces internos que apuntan a la versión parametrizada en vez de la limpia?
      - ¿El módulo de URL rewriting del CMS está mal configurado?
      - ¿El sitemap incluye URLs parametrizadas en vez de las limpias?
      - ¿Hay templates o widgets que generan enlaces con parámetros de sistema?
   3. **Verificar canonicals:** ¿las URLs parametrizadas tienen canonical apuntando a la versión limpia?
      - Si NO → problema doble: duplicación + sin consolidación
      - Si SÍ → el bloqueo en robots.txt sigue siendo necesario para ahorrar crawl budget
   4. **Cuantificar el impacto:** ¿cuántas URLs parametrizadas existen? ¿cuántas son indexables? ¿cuántas tienen impresiones GSC?

   **Ejemplo real (PrestaShop):**
   ```
   URL limpia:        https://dominio.com/mesas-acero-inoxidable
   URL parametrizada: https://dominio.com/mesas-acero-inoxidable?rewrite_product=mesa-trabajo-central&id_product=456&controller=product
   → Mismo contenido, misma página. La parametrizada es redundante.
   → Bloquear: Disallow: /*rewrite_product=
   → Investigar: ¿qué genera el enlace con rewrite_product? (breadcrumb, widget, módulo)
   ```

3. **Auditoría del robots.txt actual**: Descargar `robots.txt` del dominio con curl y analizar:
   - ¿Qué parámetros YA están bloqueados? (buscar `Disallow: /*?` y `Disallow: /*&`)
   - ¿Qué parámetros FALTAN por bloquear?
   - ¿Hay bloqueos excesivos que impidan rastreo legítimo?
   - ¿El sitemap está declarado?

4. **Propuesta de robots.txt**: Generar las reglas CONCRETAS que faltan. Formato:
   ```
   # Bloquear filtros de facetas (N URLs afectadas)
   Disallow: /*selected_filters=
   Disallow: /*order=

   # Bloquear parámetros redundantes del CMS (N URLs afectadas)
   Disallow: /*rewrite_product=
   Disallow: /*rewrite_category=
   Disallow: /*id_product=
   Disallow: /*id_category=
   Disallow: /*controller=

   # Bloquear parámetros de tracking
   Disallow: /*utm_source=
   Disallow: /*gclid=

   # NO bloquear (paginación - necesaria para rastreo)
   # Allow: /*page=    ← ya permitido por defecto
   ```

5. **Oportunidad de keywords en filtros — ANÁLISIS OBLIGATORIO**:
   Los filtros/facetas a veces generan combinaciones de keywords que tienen demanda de búsqueda real pero NO tienen URL limpia propia en el sitio. Esto es una **oportunidad de arquitectura**.

   **Cómo detectar:**
   - Extraer los valores de los parámetros de filtro del crawl (ej: `selected_filters=...material-acero`, `?color=negro`)
   - Buscar si existen URLs limpias para esos conceptos (ej: `/mesas-acero-inoxidable-negra/`)
   - Si NO existen → investigar si hay demanda de búsqueda para esa combinación
   - Si hay demanda → recomendar crear URLs SEO-friendly con arquitectura propia

   **Ejemplo real:**
   - Filtro: `?selected_filters=...sin_bomba_desague` → genera la combinación "lavavasos sin bomba desagüe"
   - No existe URL limpia para esa combinación
   - Si hay búsquedas → oportunidad de crear `/lavavasos/sin-bomba-desague/` con contenido propio
   - Esto convierte un parámetro de filtro en una página posicionable

   **Output esperado:** tabla con parámetros de filtro que podrían tener demanda, indicando si hay URL limpia equivalente o no.

### 2.7 Análisis del robots.txt

**OBLIGATORIO**: Descargar y analizar el robots.txt completo del dominio:

```bash
curl -sL "https://[dominio]/robots.txt"
```

Reportar:
1. **User-agents definidos**: ¿hay reglas específicas por bot?
2. **Disallows actuales**: listar todos los bloqueos y evaluar si son correctos
3. **Allows**: ¿hay excepciones necesarias (CSS/JS para renderizado)?
4. **Sitemap declarado**: ¿está presente? ¿URL correcta?
5. **Bloqueos que faltan**: parámetros sin bloquear, carpetas de sistema expuestas
6. **Bloqueos incorrectos**: ¿se está bloqueando algo que debería rastrearse?
7. **Robots.txt optimizado**: Escribir la versión COMPLETA optimizada del robots.txt (ver sección 2.8)

### 2.8 Robots.txt optimizado — OBLIGATORIO

**SIEMPRE incluir en el reporte una propuesta completa de robots.txt optimizado**, lista para copiar y pegar. Debe contener:

1. **Todas las reglas actuales que son correctas** (mantener lo que funciona)
2. **Nuevas reglas Disallow para TODOS los parámetros detectados** que no sean de paginación, organizadas por categoría y con comentarios explicativos
3. **Regla Allow explícita para paginación** (recordatorio de que NO se bloquea)
4. **Sitemap declarado** con URL correcta

**Formato obligatorio del robots.txt optimizado:**

```
User-agent: *

# =============================================
# PAGINACIÓN — NO BLOQUEAR (contiene enlaces a productos reales)
# =============================================
# Allow: /*page=  ← permitido por defecto, no necesita regla explícita

# =============================================
# PARÁMETROS REDUNDANTES DEL CMS (N URLs afectadas)
# Estas URLs son duplicados exactos de las URLs limpias.
# El CMS las genera por [causa identificada].
# Acción adicional: corregir la generación en [módulo/template].
# =============================================
Disallow: /*rewrite_product=
Disallow: /*rewrite_category=
Disallow: /*id_product=
Disallow: /*id_category=
Disallow: /*controller=
Disallow: /*id_lang=

# =============================================
# FILTROS DE FACETAS (N URLs afectadas)
# =============================================
Disallow: /*selected_filters=
Disallow: /*order=
Disallow: /*orderby=
Disallow: /*orderway=

# =============================================
# TRACKING Y MARKETING
# =============================================
Disallow: /*utm_source=
Disallow: /*utm_medium=
Disallow: /*utm_campaign=
Disallow: /*gclid=
Disallow: /*fbclid=
Disallow: /*mot_tcid=

# =============================================
# SESIÓN Y SISTEMA
# =============================================
Disallow: /*token=
Disallow: /*back=
Disallow: /*id_currency=

# =============================================
# CARPETAS DE SISTEMA (si aplica)
# =============================================
Disallow: /admin*/
Disallow: /cache/
Disallow: /classes/
Disallow: /config/
Disallow: /download/
Disallow: /mails/
Disallow: /modules/
Disallow: /translations/
Disallow: /tools/
Disallow: /upload/

# =============================================
# SITEMAP
# =============================================
Sitemap: https://[dominio]/sitemap.xml
```

**Instrucciones:**
- Adaptar las reglas al CMS detectado (PrestaShop, WooCommerce, Magento, custom)
- Incluir SOLO los parámetros realmente detectados en el crawl, no inventar
- Para cada bloque de reglas, indicar entre paréntesis cuántas URLs afecta
- Si un parámetro ya está bloqueado en el robots.txt actual, mantenerlo
- Añadir comentarios sobre la causa raíz de la generación de URLs parametrizadas
- El robots.txt optimizado debe ser funcional: un developer debe poder copiarlo y pegarlo directamente

---

## Bloque 3: arquitectura

### 3.1 Crawl depth
[Incluir la distribución de profundidad del JSON]

**Análisis obligatorio de profundidad:**
1. Distribución: tabla con depth 0, 1, 2, 3, 4, 5, 6-10, 11-50, 50+
2. Causa de la profundidad extrema: ¿paginación encadenada? ¿categorías anidadas? ¿filtros?
3. Impacto en GSC: ¿las URLs profundas tienen impresiones? ¿o están sin datos?
4. **Propuesta de aplanamiento**: Cómo reducir la profundidad máxima a < 5 niveles:
   - ¿Añadir mega-menú con categorías profundas?
   - ¿Añadir sección "categorías populares" en home/categorías?
   - ¿Reducir número de niveles de categorización?
   - ¿Implementar paginación con enlaces directos (1, 2, 3... 10, 20, 30) en vez de secuencial?

### 3.2 URLs SEO-friendly
[...]

### 3.3 Contenido semántico y thin content
[Incluir distribución de word count]

### 3.4 Oportunidades de arquitectura

**Análisis obligatorio:**

1. **Segmentos de URL**: Agrupar todas las URLs por el primer segmento de ruta (`/blog/`, `/productos/`, `/categorias/`, etc.) y contar:
   ```
   | Segmento | URLs | Indexables | Impresiones GSC | Clics GSC |
   | /blog/ | 245 | 230 | 45.000 | 3.200 |
   | /productos/ | 3.400 | 3.100 | 120.000 | 8.500 |
   ```

2. **Páginas con más autoridad** (Link Score): ¿Qué páginas concentran más enlaces internos? ¿Coinciden con las páginas más importantes para el negocio?

3. **Páginas con más tráfico vs profundidad**: Cruzar top 50 URLs por impresiones GSC con su profundidad. Si hay URLs con mucho tráfico a profundidad > 3, son candidatas a subir de nivel.

4. **Consolidación de categorías**: Si hay categorías con muy pocos productos (< 5) o pocas impresiones, proponer fusión.

5. **URLs canibalizándose**: Identificar grupos de URLs que compiten por las mismas keywords. Tres niveles de detección:
   - Titles idénticos entre URLs diferentes
   - H1 idénticos entre URLs diferentes
   - Slugs similares: buscar pares donde un slug es abreviación, variante o subconjunto del otro (ej: `/mesas-acero-inoxidable` vs `/mesas-acero-inox`, `/fregaderos-industriales` vs `/fregaderos-inoxidables-industriales`)
   - Para cada par: comparar métricas GSC y recomendar redirigir la más débil → la más fuerte

---

## Bloque 5: códigos de error

### 5.1 Errores internos
[3xx, 4xx, 5xx con enlaces apuntando]

### 5.2 Errores externos
[...]

### 5.3 Cadenas de redirección
[...]

---

## Bloque 6: interlinking

### 6.1 Enlazado interno
[Distribución de Link Score, URLs huérfanas]

### 6.2 Nofollow internos
[...]

### 6.3 Texto de anclaje
[...]

---

## Bloque 7: contenido

### 7.1 Encabezados (H1, H2)
[Todos los issues de H1 y H2]

### 7.2 Meta etiquetas (title, description)
[Todos los issues de titles y meta descriptions]

### 7.3 Canibalización
**OBLIGATORIO:** Detectar y listar pares de URLs canibalizadas en estos 3 niveles:
1. **Titles idénticos**: URLs diferentes con el mismo title exacto
2. **H1 idénticos**: URLs diferentes con el mismo H1 exacto
3. **Slugs similares**: URLs con slugs que son variantes del mismo concepto (abreviaciones, singular/plural, con/sin prefijo)

Para cada par, incluir: URL A, URL B, qué comparten (title/H1/slug), impresiones y clics GSC de cada una, y recomendación (redirigir A→B, diferenciar, o fusionar).

Ejemplo: `/mesas-acero-inoxidable` vs `/mesas-acero-inox` → misma intención de búsqueda, la que tenga más impresiones GSC se queda, la otra redirige 301.

### 7.4 Contenido duplicado interno
[Semiduplicados]

### 7.5 Thin content y soft 404
[⚠️ Thin content: calcular SOLO sobre HTML indexable sin paginaciones — ver reglas de filtrado]

---

## Bloque 8: EEAT

### 8.1 Autoridad
[Datos de Link Score, subdominios detectados]

### 8.2 Verificaciones manuales pendientes
- [ ] Biografía de autores visible
- [ ] Página "Sobre nosotros" detallada
- [ ] Información de contacto accesible
- [ ] Testimonios y reseñas
- [ ] Contenido original y profundo

---

## Bloque 9: offpage

[Si tiene_backlinks = true, analizar. Si no:]
> Datos de backlinks no disponibles. Proporcionar export de Ahrefs para este análisis.

### 9.1 Perfil general
[DR, dominios referencia, follow/nofollow]

### 9.2 Anchor text
[Top 20 anchor texts]

### 9.3 Spam
[...]

---

## Bloque 10: WPO

> Este bloque requiere análisis manual con PageSpeed Insights.
> Ejecutar auditoría de las URLs principales (home, categorías, productos) en:
> https://pagespeed.web.dev/

---

## Bloque 11: internacional

### 11.1 Hreflang
[Si hay problemas de hreflang]

### 11.2 Idiomas
[Distribución de idiomas detectados]

---

## Bloque S: Shopify (solo si CMS = Shopify)

> **Nota:** Esta sección solo se incluye si se detecta Shopify como CMS del sitio (ver detección en Paso 1.5). Los problemas se reportan integrados en los bloques correspondientes y aquí se presenta un resumen consolidado con las acciones específicas de plataforma.

### S.1 Duplicación por trailing slash
[Estado del canonical con/sin barra final]

### S.2 Duplicación de URLs de producto (/products/ vs /collections/.../products/)
[Cantidad de URLs duplicadas, estado del canonical, enlaces internos]

### S.3 Filtros de colecciones
[Parámetros detectados, estado en robots.txt, canonical]

### S.4 Estructura de URLs y breadcrumbs
[Impacto de la estructura rígida, estado de las migas de pan]

### S.5 Meta robots y Schema
[Mecanismo de control de indexación, estado de datos estructurados]

### S.6 Rendimiento y apps
[Inventario de apps, impacto en WPO]

### S.7 Internacional (si aplica)
[Estado de hreflang, Shopify Markets, selector de idiomas]

### S.8 Robots.txt — alerta de reseteo
[Reglas personalizadas actuales, riesgo de pérdida tras update de theme]

---

## Checklist de verificación final

**OBLIGATORIO: Antes de entregar la auditoría, verificar que TODOS estos análisis están presentes. Si falta alguno, completarlo antes de generar el MD.**

- [ ] **Robots.txt**: descargado, analizado, propuesta de mejora con reglas concretas
- [ ] **Robots.txt optimizado**: versión COMPLETA lista para copiar y pegar (sección 2.8)
- [ ] **Inventario de parámetros**: tabla con TODOS los parámetros, clasificados, con URLs y GSC data
- [ ] **Parámetros redundantes del CMS**: identificados, causa raíz investigada, bloqueados en robots.txt
- [ ] **Propuesta de bloqueo robots.txt**: reglas Disallow concretas para cada parámetro que debe bloquearse
- [ ] **Paginaciones**: verificado que canonical es autoreferenciado (NO a página 1), NO se recomienda noindex ni bloqueo en robots.txt, SÍ se recomienda aumentar productos por página si hay muchas paginaciones
- [ ] **Enlaces con fragmento #**: detectados y cuantificados enlaces internos con `#/filtro-valor`
- [ ] **Oportunidad de filtros→arquitectura**: evaluado si filtros generan keywords sin URL limpia propia
- [ ] **Canibalización**: detectados pares de URLs con title, H1 o slug idéntico/similar, con métricas GSC
- [ ] **Profundidad**: tabla de distribución, causa raíz identificada, propuesta de aplanamiento
- [ ] **Segmentos de URL**: tabla con métricas por segmento de ruta
- [ ] **Correlaciones entre bloques**: al menos 2-3 conexiones entre problemas de diferentes bloques
- [ ] **Shopify (si aplica)**: trailing slash, product URLs, filtros, meta robots, Schema, breadcrumbs, apps, robots.txt backup, hreflang

---

## Plan de acción priorizado

### Prioridad 1 - urgente

1. **[Título de la acción]**
   - Bloque: [N]
   - Problema: [causa raíz, no síntoma]
   - URLs afectadas: [N]
   - Impacto: [qué se gana corrigiéndolo]
   - Solución: [pasos concretos]

### Prioridad 2 - importante

[...]

### Prioridad 3 - mejoras

[...]

---

## Verificaciones manuales pendientes

| Verificación | Bloque | Herramienta |
|-------------|--------|-------------|
| Robots.txt (bloqueos) | B2 | Inspección manual |
| Renderizado JavaScript | B2 | `/seo-render-audit` |
| Sitemap (URLs vs crawl) | B1 | Inspección manual + GSC |
| Datos estructurados | B7 | Rich Results Test |
| Core Web Vitals | B10 | PageSpeed Insights |
| Migas de pan | B6 | Inspección manual |
| Contenido duplicado externo | B7 | Búsqueda entrecomillada en Google |
| EEAT completo | B8 | Revisión manual |

---

## Archivos generados

| Archivo | Contenido |
|---------|-----------|
| `auditoria_seo_tecnica.md` | Este informe |
| `evidencia_auditoria.xlsx` | URLs filtradas por problema ([N] hojas) |
| `resumen_auditoria.json` | Métricas agregadas (uso interno) |

---

*Auditoría generada con /seo-tecnico — Skill de auditoría SEO técnica*
```

## Patrones específicos de Shopify

**ACTIVAR ESTA SECCIÓN SOLO si se detecta Shopify como CMS** (ver detección en Paso 1.5). Cuando el CMS es Shopify, integrar estos checks adicionales en los bloques correspondientes de la auditoría.

### Shopify P1 — Duplicación por trailing slash (Bloque 1: indexación)

En Shopify, **todas las URLs tienen dos versiones**: con y sin barra final (`/`). Esto genera contenido duplicado.

**Qué verificar:**
- ¿Las versiones con slash tienen canonical a la versión sin slash? (comportamiento por defecto de Shopify)
- ¿Hay URLs con slash indexadas por separado en GSC?
- Probar: `curl -sI "https://[DOMINIO]/collections/nombre/" | grep -i "canonical\|location"`

**Diagnóstico:** Si el canonical por defecto funciona → informativo (no es un problema activo). Si hay URLs con slash indexadas con impresiones propias → problema de duplicación activa.

### Shopify P2 — Estructura de URLs rígida (Bloque 3: arquitectura)

Shopify impone directorios fijos por tipología:
- **Páginas:** `/pages/[slug]`
- **Categorías/colecciones:** `/collections/[slug]`
- **Productos:** `/products/[slug]`
- **Blog:** `/blogs/[nombre-blog]/[slug-articulo]`

**Limitación crítica:** Las subcategorías NO se pueden anidar. `/collections/ropa-ninos/camisetas` NO es posible → siempre será `/collections/camisetas`.

**Qué verificar:**
- Listar los directorios de primer nivel detectados en el crawl y confirmar que siguen el patrón Shopify
- Detectar si hay intentos de URLs personalizadas fuera de la estructura estándar (indicaría apps o redirecciones)
- Evaluar si la imposibilidad de anidar subcategorías afecta a la arquitectura semántica del sitio

**Diagnóstico:** No tiene solución técnica → reportar como limitación de plataforma. Usar la estructura fija como ventaja para segmentación en reportes.

### Shopify P3 — Migas de pan incompletas (Bloque 6: interlinking)

Las breadcrumbs de Shopify se generan a partir de la estructura de URL, pero al no poder anidar subcategorías, las migas de pan muestran rutas incompletas.

**Qué verificar:**
- Inspeccionar las breadcrumbs de páginas de producto y subcategorías
- ¿Las breadcrumbs reflejan la jerarquía real del catálogo?
- ¿Existe datos estructurados BreadcrumbList correctos?

**Acción recomendada:** Implementar breadcrumbs correctas via metacampos de Shopify o con apps como "Category Breadcrumbs".

### Shopify P4 — URLs de producto duplicadas (Bloque 1: indexación + Bloque 2: rastreo)

Cada producto tiene **dos URLs** con el mismo contenido:
- **Canonical:** `/products/nombre-producto`
- **Duplicada:** `/collections/nombre-coleccion/products/nombre-producto`

**Qué verificar:**
- ¿La versión `/collections/.../products/` tiene canonical apuntando a `/products/`? (comportamiento por defecto)
- ¿Los listados de categorías enlazan a la versión canonical (`/products/`) o a la versión con colección (`/collections/.../products/`)?
- Contar cuántas URLs con patrón `/collections/*/products/*` aparecen en el crawl

**Diagnóstico:** Si el canonical funciona pero los listados enlazan a la versión no-canonical → problema de enlazado interno que diluye señales.

**Solución (INCLUIR SIEMPRE en el informe cuando se detecte este patrón):** Modificar la plantilla Liquid del tema para que los enlaces de producto en los listados de collections apunten directamente a `/products/nombre-producto` en vez de a `/collections/nombre-coleccion/products/nombre-producto`. En Liquid, esto significa cambiar `{{ product.url | within: collection }}` por `{{ product.url }}` en los templates de collection. NUNCA reportar este punto como "comportamiento esperado de Shopify" sin incluir la solución de enlazado.

**Probar:**
```bash
# Detectar URLs de producto bajo collections
$PYTHON -c "
import pandas as pd
df = pd.read_excel('$ARGUMENTS/internos_todo.xlsx')
mask = df['Dirección'].str.contains('/collections/.*/products/', regex=True, na=False)
print(f'URLs de producto bajo /collections/: {mask.sum()}')
print(df[mask]['Dirección'].head(10).to_string())
"
```

### Shopify P5 — Filtros de categorías generan duplicados (Bloque 2: rastreo)

Los filtros de facetas en colecciones generan URLs con parámetros que multiplican el contenido duplicado (ej: `/collections/ropa?filter.v.color=rojo&sort_by=price-ascending`).

**Qué verificar:**
- ¿Las URLs con filtros tienen canonical a la colección principal? (por defecto sí)
- ¿Los filtros están bloqueados en robots.txt?
- ¿Los enlaces de filtros están ofuscados (JS) o son rastreables (href)?
- Inventariar los parámetros de filtro detectados: `filter.v.*`, `sort_by`, `filter.p.*`

**Acción recomendada:**
1. Bloquear filtros en robots.txt: `Disallow: /*filter.*=` y `Disallow: /*sort_by=`
2. Ofuscar enlaces de filtros (requiere desarrollo avanzado en Liquid/JS)
3. Evaluar oportunidad: ¿alguna combinación de filtro tiene demanda propia? → crear colección dedicada

### Shopify P6 — Sin control de meta robots (Bloque 1: indexación)

Shopify no permite editar la etiqueta `meta robots` de forma nativa por URL.

**Qué verificar:**
- ¿Hay URLs que deberían tener noindex pero no lo tienen? (ej: páginas de búsqueda interna, páginas legales duplicadas, landing de campañas caducadas)
- ¿Se usa alguna app para gestionar meta robots? (SEO Manager, TinyIMG, etc.)
- ¿Se ha editado el template Liquid para incluir noindex condicional?

**Diagnóstico:** Si hay URLs que necesitan noindex y no tienen → recomendar app o edición de Liquid.

### Shopify P7 — Datos estructurados Schema incompletos (Bloque 7: contenido)

Shopify no incluye por defecto el marcado completo de datos estructurados. Especialmente relevante para e-commerce (Product, BreadcrumbList, FAQ, Organization).

**Qué verificar:**
- Testear URLs representativas en https://search.google.com/test/rich-results
- ¿Existe marcado Product con price, availability, review?
- ¿Existe BreadcrumbList?
- ¿Existe Organization en home?
- ¿Cumple con los requisitos de Google para fichas de comerciantes (Merchant listings)?

**Acción recomendada:** Implementar Schema completo via Liquid o con app especializada. Priorizar Product Schema para elegibilidad en resultados enriquecidos.

### Shopify P8 — Imágenes en CDN sin control de slug (Bloque 3: arquitectura)

Las imágenes se alojan en `cdn.shopify.com` con nombres de archivo generados automáticamente. No se puede modificar el slug de la imagen.

**Qué verificar:**
- ¿Las imágenes tienen atributo ALT descriptivo? (lo único que SÍ se puede controlar)
- ¿Se usan formatos modernos (WebP)?
- ¿Se sirven con lazy loading?

**Diagnóstico:** Limitación de plataforma. Foco en ALT texts como compensación.

### Shopify P9 — Rendimiento degradado por apps (Bloque 10: WPO)

El uso excesivo de apps de Shopify inyecta JS/CSS adicional que degrada el rendimiento.

**Qué verificar:**
- Ejecutar PageSpeed en URLs representativas
- Contar scripts de terceros inyectados por apps
- ¿Hay apps instaladas pero no activas (código residual)?

**Acción recomendada:** Auditar apps instaladas, eliminar las innecesarias, evaluar impacto en Core Web Vitals de cada app activa.

### Shopify P10 — Selector de idiomas sin hreflang (Bloque 11: internacional)

En proyectos internacionales, el selector de idiomas puede no contener enlaces con atributo `href`, impidiendo que los bots rastreen las versiones en otros idiomas.

**Qué verificar:**
- ¿El selector de idiomas usa enlaces `<a href="...">` o es solo JavaScript?
- ¿Las URLs alternativas son accesibles para crawlers?
- `curl -sL "https://[DOMINIO]/" | grep -i "hreflang"`

**Acción recomendada:** Asegurar que el selector de idiomas incluye enlaces HTML con `href`. Si es solo JS → implementar hreflang tags en `<head>`.

### Shopify P11 — Shopify Markets multiplica hreflangs (Bloque 11: internacional)

Al activar Shopify Markets, se generan etiquetas hreflang para TODAS las variantes de idioma-país activas, multiplicando las URLs del sitio.

**Qué verificar:**
- ¿Cuántas variantes hreflang existen? (ej: es-ES, es-MX, es-AR... pueden ser decenas)
- ¿Todas las variantes tienen contenido realmente diferenciado?
- ¿Hay variantes activas que no corresponden a mercados reales del negocio?

**Diagnóstico:** Si hay variantes hreflang para mercados donde no opera el negocio → desactivar en Shopify Markets. Solo mantener las variantes necesarias.

### Shopify P12 — Robots.txt se resetea con actualizaciones de theme (Bloque 2: rastreo)

Al actualizar el theme de Shopify, el archivo robots.txt se regenera con las reglas por defecto, perdiendo cualquier personalización.

**Qué verificar:**
- ¿El robots.txt actual tiene reglas personalizadas?
- ¿Cuándo fue la última actualización del theme?
- ¿Existe backup documentado del robots.txt personalizado?

**Acción recomendada obligatoria:** Si se detectan reglas personalizadas en robots.txt → alertar al cliente de que DEBE mantener backup del robots.txt y restaurarlo después de cada actualización de theme.

### Integración en el robots.txt optimizado (Shopify)

Cuando el CMS es Shopify, el robots.txt optimizado (sección 2.8) debe incluir estas reglas específicas:

```
User-agent: *

# =============================================
# FILTROS DE COLECCIONES SHOPIFY
# =============================================
Disallow: /*filter.*=
Disallow: /*sort_by=
Disallow: /*q=

# =============================================
# URLs DE PRODUCTO BAJO COLLECTIONS (duplicadas)
# =============================================
# NOTA: No bloquear /collections/*/products/ en robots.txt
# porque el canonical ya apunta a /products/.
# En su lugar, corregir los enlaces internos en Liquid.

# =============================================
# BÚSQUEDA INTERNA
# =============================================
Disallow: /search
Disallow: /search?*

# =============================================
# SISTEMA Y CHECKOUT
# =============================================
Disallow: /cart
Disallow: /checkout
Disallow: /account
Disallow: /admin

# =============================================
# TRACKING Y MARKETING
# =============================================
Disallow: /*utm_source=
Disallow: /*utm_medium=
Disallow: /*utm_campaign=
Disallow: /*gclid=
Disallow: /*fbclid=

# =============================================
# SITEMAP
# =============================================
Sitemap: https://[DOMINIO]/sitemap.xml
```

### Checklist Shopify (añadir al checklist de verificación final)

Cuando el CMS es Shopify, verificar además:

- [ ] **Trailing slash**: canonical de versión con `/` apunta a versión sin `/`
- [ ] **URLs de producto bajo /collections/**: canonical apunta a `/products/`, enlaces internos usan versión canonical
- [ ] **Filtros de facetas**: bloqueados en robots.txt, canonical a colección principal
- [ ] **Meta robots**: existe mecanismo de control (app o Liquid personalizado)
- [ ] **Schema/datos estructurados**: Product, BreadcrumbList verificados en Rich Results Test
- [ ] **Breadcrumbs**: reflejan jerarquía completa del catálogo
- [ ] **Apps**: inventario de apps activas, impacto en rendimiento evaluado
- [ ] **Robots.txt backup**: cliente alertado sobre reseteo tras update de theme
- [ ] **Hreflang** (si internacional): solo mercados reales activos en Shopify Markets

---

## Segmentos de URL

Al analizar el JSON, intenta detectar segmentos o tipologías de URL a partir de los patrones de ruta. Por ejemplo:
- `/blog/` → contenido informativo
- `/productos/` o `/product/` → ecommerce
- `/tag/` o `/category/` → taxonomías
- `/page/` → paginaciones
- URLs con parámetros (`?`, `&`) → filtros/búsquedas

Esto ayuda a contextualizar los problemas: "el 80% del thin content está en URLs de tipo /tag/, lo que sugiere que las taxonomías no tienen contenido propio".

## Notas técnicas

- El JSON es la fuente de verdad para el análisis. Si necesitas más detalle sobre URLs concretas, consulta `evidencia_auditoria.xlsx`
- Para renderizado JavaScript, referir al usuario a la skill `/seo-render-audit`
- Para WPO, referir a PageSpeed Insights manual
- Los datos de GSC (clics, impresiones, posición) son los que Screaming Frog integra vía API, no un export directo de GSC
- El script `preprocesar_auditoria.py` acepta el argumento `--output` para generar los archivos en otra ruta

## Falsos positivos a ignorar — NO reportar

1. **Canonicals en recursos no-HTML**: URLs que contienen `.pdf`, `.jpg`, `.png`, `.gif`, `.svg`, `.zip`, `.css`, `.js` y no tienen etiqueta canonical → esto es NORMAL. Los recursos no llevan canonical. NO incluir en el análisis de canonicals ni en evidencias.

2. **Sitemap en ruta no estándar**: si `/sitemap.xml` devuelve 404 pero el sitemap real existe en otra ruta (ej: `/1_index_sitemap.xml`) y está correctamente declarado en robots.txt Y dado de alta en Google Search Console → NO reportar como problema. Solo verificar manualmente que está declarado en robots.txt y GSC. Si está declarado en ambos sitios, no hay nada que corregir.

## Generación del Excel de evidencias (modo manual)

**REGLA CRÍTICA:** Si el script `preprocesar_auditoria.py` no funciona (estructura de carpeta diferente, error de ejecución, etc.), genera el Excel manualmente pero **con EXACTAMENTE la misma estructura** que produce el script. NUNCA improvises pestañas o columnas diferentes.

**Nombre del archivo:** `evidencia_auditoria.xlsx`

### Estructura obligatoria

El Excel SIEMPRE debe tener esta estructura, en este orden exacto de pestañas:

#### Pestañas fijas (siempre presentes si hay datos GSC):

| # | Pestaña | Columnas (nombres exactos) | Origen de datos |
|---|---------|---------------------------|-----------------|
| 1 | Resumen_Issues | Área, Sub-área, Tarea, Tipo, Prioridad, Total URLs, % del total, Problema, Solución, Ejemplo | `issues_overview_report.xlsx` reestructurado |
| 2 | GSC_Oportunidades | Dirección, Título, Impresiones, Clics, CTR, Posición, Oportunidad, Clics potenciales, Diagnóstico | `internos_todo.xlsx` filtrado por oportunidad + diagnóstico |

**NOTA:** Situacion_Actual ya NO existe como pestaña separada. Sus datos se fusionaron en GSC_Oportunidades con la columna Diagnóstico.

#### Pestañas de evidencia (aparecen según los problemas detectados):

| Pestaña | Columnas (nombres exactos) | Filtro |
|---------|---------------------------|--------|
| Canonicals | Dirección, Estado, Título, Indexabilidad, Impresiones, Clics | Canonicalizada a otra URL o Sin canonical |
| NoIndex_Con_Impresiones | Dirección, Código de respuesta, Indexabilidad, Estado de indexabilidad, Canonical, Meta robots, Impresiones, Clics, Posición | No indexable AND Impresiones > 0 |
| Errores_4xx | Dirección, Código de respuesta, Respuesta, Impresiones, Clics | status_code in [400-499] |
| Errores_3xx_Cadenas | Tipo, Fuente, Destino, Tamaño (bytes), Texto ALT, Ancla, Código de estado, Estado, Seguir, Destino final, Rel, Tipo de ruta, Ruta del enlace, Posición del enlace, Origen del enlace | Cadenas de redirección |
| Profundidad | Dirección, Profundidad, Indexabilidad, Enlaces internos, Link Score, Impresiones, Clics, Posición, Acción sugerida | depth >= 4 |
| URLs_No_Friendly | Dirección, Tipo de problema, Código de respuesta, Indexabilidad | Mayúsculas, guiones bajos, >115 chars, doble barra |
| Thin_Content | Dirección, Recuento de palabras, Título, Impresiones, Clics | word_count < 200 **Y Content-Type = text/html (solo HTML, no PDFs/imágenes)** |
| Enlaces_3xx_Fuente_Dest | Fuente, Destino, Ancla, Código de estado, Estado, Posición del enlace, Origen del enlace | Enlaces internos que apuntan a 3xx |
| Enlaces_4xx_Fuente_Dest | Fuente, Destino, Ancla, Código de estado, Estado, Posición del enlace, Origen del enlace | Enlaces internos que apuntan a 4xx |
| Interlinking | Dirección, Tipo de problema, Enlaces internos, Link Score, Impresiones, Clics | Huérfanas (0 enlaces), Link Score = 0 |
| H1_H2 | Dirección, Tipo de problema, H1, Longitud H1, Indexabilidad | H1 falta, duplicado*, múltiple, >70 chars |
| Titles | Dirección, Tipo de problema, Título, Longitud del título | Duplicado*, >60 chars, <30 chars, igual que H1 |
| Meta_Descriptions | Dirección, Tipo de problema, Meta description, Longitud meta description | Falta, duplicada* |
| Hreflang | Dirección, Tipo de problema, Indexabilidad | Problemas de hreflang |

\* **Filtro de paginaciones:** Las URLs paginadas (/page/, ?page=, ?p=), parametrizadas (?...) y canonicalizadas a otra URL se EXCLUYEN de TODOS los análisis de H1, H2, Titles y Meta Descriptions (duplicados, mas_60_chars, menos_30_chars, igual_h1, falta, etc.). Estas URLs típicamente heredan el contenido de la página principal y no son problemas reales de contenido.

### Reglas de formato obligatorias

1. **Nombres de columnas**: SIEMPRE en español legible (Dirección, no url; Impresiones, no impressions; Código de respuesta, no status_code; Total URLs, no url en Resumen_Issues)
2. **SIN columna Responsable**: Ya no se incluye esta columna en ninguna pestaña
3. **Resumen_Issues**: reestructurar con Área (Contenido/Técnico-Desarrollo), Sub-área, Tarea extraída del nombre del issue, Problema y Solución despersonalizados, 3 URLs de Ejemplo por issue. Columna "Total URLs" (no "Dirección")
4. **Ejemplos en duplicados**: para issues de title/h1/meta description duplicados, formato de pares: "URL1 y URL2 → mismo título: 'valor'"
5. **Formato visual**: headers con fondo azul claro, negrita, autofilter, freeze panes en fila 1, auto-width
6. **Orden de pestañas**: Resumen_Issues → GSC_Oportunidades → resto de evidencias
7. **Canonicals**: columna "Estado" (no "Problema") con valores: "Canonicalizada a otra URL" o "Sin canonical"
8. **Thin_Content**: SOLO URLs HTML (Content-Type: text/html). Excluir PDFs, imágenes y otros archivos

### Valores de la columna Diagnóstico (GSC_Oportunidades)

| Condición | Diagnóstico |
|-----------|-------------|
| 0 impresiones | Zombie: sin impresiones |
| 0 clics AND impresiones > 100 | Visible pero no clicada: revisar snippet |
| 0 clics | Impresiones bajas sin clics |
| CTR < 1% AND impresiones > 500 | Alto volumen, CTR bajo: optimizar snippet |
| Clics <= 10 | Tráfico marginal: valorar consolidar |
| Resto | Productiva |

### Valores de la columna Oportunidad (GSC_Oportunidades)

| Condición | Oportunidad |
|-----------|-------------|
| Posición 4-10 | Quick win (pos 4-10): optimizar contenido |
| Posición 11-20 | Página 2 (pos 11-20): interlinking + contenido |
| Posición 21-40 | Alcanzable (pos 21-40): reforzar autoridad |
| Posición > 40 | Largo plazo (pos 40+): estrategia completa |

$ARGUMENTS
