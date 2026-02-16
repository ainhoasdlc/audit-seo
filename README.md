# Skills de auditoría SEO para Claude Code

Tres skills (slash commands) para Claude Code que automatizan auditorías SEO técnicas completas.

## Skills incluidos

| Skill | Comando | Descripción |
|-------|---------|-------------|
| **SEO Técnico** | `/seo-tecnico` | Auditoría técnica desde crawl de Screaming Frog. Analiza indexabilidad, canonicals, hreflang, paginación, errores HTTP y contenido |
| **Render Audit** | `/seo-render-audit` | Diagnóstico de renderizado JavaScript. Compara HTML estático vs renderizado para detectar contenido dependiente de JS |
| **Auditoría completa** | `/auditoria-seo-completa` | Workflow unificado que ejecuta técnica + renderizado en paralelo y genera informe consolidado |

## Instalación

1. Copia la carpeta `.claude/commands/` en la raíz de tu proyecto:

```bash
# Desde la raíz de tu proyecto
mkdir -p .claude/commands
cp ruta/a/este/repo/.claude/commands/*.md .claude/commands/
```

2. Configura la ruta de Python en tu `CLAUDE.md`:

```markdown
## Entorno Python

PYTHON="C:\ruta\a\tu\python.exe"   # Windows
PYTHON="python3"                     # Linux/Mac
```

> Todos los skills usan `$PYTHON` como referencia. Sin esta configuración, los comandos de Python no funcionarán.

## Dependencias

```bash
pip install pandas openpyxl requests playwright
playwright install chromium
```

## Uso

### Auditoría técnica
```
/seo-tecnico https://ejemplo.com
```
Requiere: export de crawl de Screaming Frog en `.xlsx`

### Auditoría de renderizado
```
/seo-render-audit https://ejemplo.com
```
Compara HTML estático vs JavaScript para detectar problemas de renderizado.

### Auditoría completa (recomendado)
```
/auditoria-seo-completa https://ejemplo.com
```
Ejecuta ambas auditorías en paralelo y genera un informe unificado.

## Cómo subir skills nuevos a este repo

Si creas o modificas skills y quieres subirlos a GitHub:

### Primera vez (clonar el repo)

```bash
git clone https://github.com/ainhoasdlc/audit-seo.git
cd audit-seo
```

### Añadir o actualizar skills

1. Copia los archivos `.md` de tus skills a `.claude/commands/`:

```bash
cp /ruta/a/tu/proyecto/.claude/commands/mi-skill.md .claude/commands/
```

2. **Parametriza las rutas de Python** antes de subir. Reemplaza cualquier ruta hardcodeada por `$PYTHON`:

```
# MAL (hardcodeado, no funcionará en otros equipos)
"C:\Users\tu_usuario\AppData\Local\Programs\Python\Python310\python.exe" -c "import pandas..."

# BIEN (parametrizado, portable)
$PYTHON -c "import pandas..."
```

3. Verifica que no queden rutas personales:

```bash
grep -r "Users/" .claude/commands/
# No debería devolver resultados
```

4. Commit y push:

```bash
git add .claude/commands/mi-skill.md
git commit -m "Add mi-skill: descripción breve"
git push
```

### Subir desde un proyecto existente de Claude Code

Si tu proyecto ya tiene un `.git` con archivos pesados (Excel, crawls, etc.), el push puede fallar por timeout. En ese caso, crea un directorio de export limpio:

```bash
# Crear directorio limpio
mkdir -p github-export/.claude/commands

# Copiar solo los skills
cp .claude/commands/mi-skill.md github-export/.claude/commands/

# Inicializar repo limpio y push
cd github-export
git init
git remote add origin https://github.com/ainhoasdlc/audit-seo.git
git pull origin main
git add .
git commit -m "Add mi-skill"
git push origin main
```

## Reglas SEO integradas

Los skills aplican automáticamente estas reglas:

- **Nunca bloquear paginaciones** en robots.txt ni marcarlas con noindex
- **La demanda manda**: al encontrar URLs duplicadas, la URL con más tráfico en GSC siempre gana
- **Investigación de causa raíz**: nunca reportar solo síntomas, siempre llegar al porqué
- **Pruebas de validación**: probar URLs inventadas, formatos alternativos, valores extremos
