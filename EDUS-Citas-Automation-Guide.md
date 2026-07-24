---
name: edus-citas-automation-guide
description: "Guía genérica para automatizar la reserva de citas médicas en EDUS (CCSS Costa Rica) con cualquier AI agent — sin datos personales."
category: productivity
---

# EDUS Citas Auto-Booking — Guía Genérica

Automatización de monitoreo y reserva de citas médicas en el sistema EDUS de la CCSS (Costa Rica) usando agentes de IA como Hermes, OpenClaw, o cualquier herramienta con Playwright + cron.

## Arquitectura del Sistema

La CCSS tiene **dos sistemas separados** para citas:

### 1. EDUS Citas Web (citas médicas generales)
- **URL**: `https://edus.ccss.sa.cr/eduscitasweb/`
- **Backend**: JavaServer Faces (JSF) + PrimeFaces 12.0.0 sobre Oracle WebLogic
- **Context path**: `/CitasWebPF/`
- **Login**: Cédula + contraseña + CAPTCHA
- **Endpoint público**: `centroSalud.xhtml` — lista establecimientos sin login

### 2. Sistema de Vacunación (Oracle APEX)
- **URL**: `https://serviciosweb.ccss.sa.cr/pls/APEXPRD/APEX/r/servicios_ccss/sgap361/formulario`
- **Backend**: Oracle APEX 24.x
- **Sin login**: Formulario directo con selección de zona → área → fecha → hora

> **Nota**: Esta guía se enfoca en EDUS Citas Web (citas médicas). El sistema de vacunación usa Playwright para formularios APEX pero es más simple.

---

## Fase 1: Reconocimiento (sin credenciales)

### Listar establecimientos públicos

El endpoint `centroSalud.xhtml` no requiere autenticación. Se puede consultar vía HTTP directo con requests JSF AJAX.

```
POST https://edus.ccss.sa.cr/CitasWebPF/faces/xhtml/centroSalud/centroSalud.xhtml
Content-Type: application/x-www-form-urlencoded
Faces-Request: partial/ajax

javax.faces.partial.ajax=true
javax.faces.ViewState=<viewstate_extraido>
formSIAC:tablaCentroSalud_pagination=true
formSIAC:tablaCentroSalud_first=0
formSIAC:tablaCentroSalud_rows=10
```

- La respuesta es XML `<partial-response>` con CDATA conteniendo HTML de la tabla
- El `ViewState` se extrae de la página inicial y se rota en cada request
- El ViewState viene en GZIP + Base64 (~30KB)
- 98 áreas de salud, ~2000+ servicios reportados

### Filtro por área

Añadir `formSIAC:tablaCentroSalud:globalFilter=NOMBRE_AREA` al POST.

---

## Fase 2: Login con CAPTCHA

### Formulario de login

```
Form ID: formInicioSesion
Campos:
  - tipIdentificacion_input (0=cédula nacional, 6=temporal, 7=extranjero)
  - usuario (9 dígitos para cédula nacional)
  - clave (contraseña)
  - captchaDigitado (texto del CAPTCHA)
```

### CAPTCHA

- **URL de la imagen**: `/CitasWebPF/captcha` (PNG 270×70px)
- Debe descargarse con las cookies de sesión (JSESSIONID + WebLogic cookie)
- No uses screenshot de Playwright — descarga directa HTTP para mejor calidad

### OCR del CAPTCHA

```python
# Estrategia con mejor tasa de acierto (~30-40% por intento)
from PIL import Image, ImageEnhance
import subprocess

img = Image.open("captcha.png").convert("L")        # grayscale
img = ImageEnhance.Contrast(img).enhance(2.0)        # aumentar contraste
img = img.resize((w*3, h*3), Image.LANCZOS)          # upscale 3x

subprocess.run([
    "tesseract", "captcha_processed.png", "-",
    "--psm", "7",
    "-c", "tessedit_char_whitelist=ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"
])
```

- PSM 7 trata la imagen como una línea de texto
- Whitelist alfanumérico (solo letras y números, sin símbolos)
- **Retry loop de 20-30 intentos**: cada intento recarga la página para obtener un nuevo CAPTCHA
- El CAPTCHA no siempre aparece — puede activarse tras varios intentos fallidos

### Submit del login

```python
await page.evaluate(f"""
    document.getElementById('formInicioSesion:usuario').value = '{CEDULA}';
    document.getElementById('formInicioSesion:clave').value = '{CLAVE}';
    document.getElementById('formInicioSesion:captchaDigitado').value = '{CAPTCHA}';
    document.getElementById('formInicioSesion:ejecutarPaso1').click();
""")
```

Login exitoso si el HTML resultante contiene `"Agregar una cita"`.

---

## Fase 3: Reserva de Cita (post-login)

### Estructura de la página principal

- Form: `formSIAC`
- Botón agregar cita: `formSIAC:btnMenuAdd` → PrimeFaces AJAX
- Tabla de citas existentes: `formSIAC:tablaCitas`
- Tabla de grupo familiar: `formSIAC:tablaFamiliares`
- Tabla de exámenes: `formSIAC:listaEstudiosLabcore`

### Flujo de solicitud de cita

1. **Click "Agregar una cita"**:
   ```javascript
   PrimeFaces.ab({s: 'formSIAC:btnMenuAdd', f: 'formSIAC'});
   ```
   Esperar 3-5s para que cargue el formulario.

2. **Seleccionar servicio** (`formSIAC:menuServicios_input`):
   - `1` = MEDICINA
   - Dispara AJAX onchange para cargar especialidades

3. **Seleccionar especialidad** (`formSIAC:menuEspecialidades_input`):
   - `1033` = MEDICINA GENERAL
   - Dispara AJAX onchange para cargar cupos

4. **Leer tabla de cupos** (`formSIAC:cuposDisponibles`):
   ```javascript
   const rows = document.querySelectorAll('#formSIAC\\:cuposDisponibles tbody tr');
   // Columnas: Fecha, Hora, N°Cita, Consultorio, Funcionario, Ver cita
   ```
   - Nota: los `:` en IDs JSF deben escaparse con `\\` en selectores CSS
   - Alternativa: usar `document.getElementById('formSIAC:cuposDisponibles')` (sin escape)

5. **Reservar**: Click en el botón "Ver cita" de la fila → diálogo de confirmación → botón "Confirmar"

### Errores comunes

| Mensaje | Significado |
|---------|-------------|
| "No se encontraron cupos disponibles" | Sin cupos |
| "El paciente posea citas para ese mismo día" | Ya tiene cita ese día |
| "El servicio o la especialidad no estén disponibles para el género" | Restricción de género |
| "La cita haya sido asignada a otro usuario" | Cupo tomado entre consulta y reserva |

---

## Fase 4: Grupo Familiar (citas para familiares)

El sistema permite agendar citas para familiares registrados **sin necesidad de credenciales separadas**.

### Flujo

1. Login con credenciales del **titular** de la cuenta
2. En la tabla `formSIAC:tablaFamiliares`, hacer clic en **"Ver Citas"** en la fila del familiar
3. Se despliega la sección de citas del familiar con su propio botón **"Agregar una cita"**
4. Clic en ese botón → se abre "Solicitar Cita" para el familiar
5. Mismo flujo: servicio → especialidad → cupos → reservar

### Identificar el botón correcto

Hay **dos** botones "Agregar una cita" en la página: uno para el titular y otro para el familiar. Para clickear el correcto, se busca el botón que aparece **después** del nombre del familiar en el DOM:

```python
# ⚠️ CRÍTICO: Las variables Python deben resolverse ANTES de generar el JS.
# NUNCA uses variables de Python como si fueran variables de JS en un f-string.
# Mal:  f"if (MI_VARIABLE && ...)"  → ReferenceError en JS
# Bien: evaluar la condición en Python y generar código JS distinto

fam_nombre = os.environ.get("FAMILIAR_NOMBRE", "")
if fam_nombre:
    # Buscar por nombre del familiar
    search_term = fam_nombre.upper()
    js_code = f"""
        let seen = false;
        const walker = document.createTreeWalker(document.body, NodeFilter.SHOW_ELEMENT);
        let node;
        while (node = walker.nextNode()) {{
            const txt = (node.textContent || '').trim();
            if (txt.toUpperCase().includes('{search_term}') && node.children.length === 0) {{
                seen = true;
                continue;
            }}
            if (seen && txt === 'Agregar una cita'
                && (node.tagName === 'A' || node.tagName === 'BUTTON' || node.getAttribute('onclick'))) {{
                node.click();
                break;
            }}
        }}
    """
else:
    # Buscar por cédula del familiar
    js_code = f"""
        let seen = false;
        const walker = document.createTreeWalker(document.body, NodeFilter.SHOW_ELEMENT);
        let node;
        while (node = walker.nextNode()) {{
            const txt = (node.textContent || '').trim();
            if (txt.includes('{familiar_cedula}') && node.children.length === 0) {{
                seen = true;
                continue;
            }}
            if (seen && txt === 'Agregar una cita'
                && (node.tagName === 'A' || node.tagName === 'BUTTON' || node.getAttribute('onclick'))) {{
                node.click();
                break;
            }}
        }}
    """
await page.evaluate(js_code)
```

> **⚠️ Pitfall importante**: Al generar JavaScript desde Python con f-strings, las variables de entorno como `FAMILIAR_NOMBRE` deben interpolarse con `{FAMILIAR_NOMBRE}`. Si escribís `FAMILIAR_NOMBRE` sin llaves en el f-string, Python lo trata como texto literal y el navegador lanza `ReferenceError: FAMILIAR_NOMBRE is not defined`. La solución es **evaluar condiciones en Python** (`if fam_nombre:`) y generar código JS diferente para cada caso, en lugar de meter lógica condicional con variables de Python sin interpolar en el JS.

---

## Fase 5: Automatización Continua (Watchdog + Cron)

### Script watchdog

Comportamiento ideal para un script de monitoreo recurrente:

- **Sin cupos → salida silenciosa** (exit 0, sin stdout)
- **Con cupos → intenta reservar, imprime resultado** (exit 0 con stdout)
- **Error → imprime error** (exit 1)

Esto permite usarlo como cron job donde solo se reciben notificaciones cuando hay acción.

### Estructura del script

```python
# Variables de entorno recomendadas:
#   EDUS_CEDULA       - cédula del titular
#   EDUS_CLAVE        - contraseña
#   FAMILIAR_CEDULA   - (opcional) cédula del familiar
#   EXCLUIR_FECHAS    - (opcional) "DD/MM/AAAA,DD/MM/AAAA"
#   SERVICIO          - código de servicio (default: 1)
#   ESPECIALIDAD      - código de especialidad (default: 1033)

async def check_and_book():
    # 1. Login con reintentos + OCR de CAPTCHA
    # 2. Si FAMILIAR_CEDULA: switch_to_familiar()
    # 3. Seleccionar servicio + especialidad
    # 4. Parsear cupos (filtrar EXCLUIR_FECHAS)
    # 5. Si hay cupos: intentar reservar el primero
    # 6. Retornar resultado
```

### Cron schedule

Los cupos en EDUS suelen liberarse entre **5am y 8am hora Costa Rica (UTC-6)**. Fuera de ese horario es raro encontrar disponibilidad.

```bash
# Wrapper bash para filtrar por horario + suprimir errores
#!/bin/bash
HOUR=$(TZ='America/Costa_Rica' date +%H)
if [ "$HOUR" -ge 5 ] && [ "$HOUR" -lt 8 ]; then
    export FAMILIAR_CEDULA="CEDULA_FAMILIAR"
    export FAMILIAR_NOMBRE="NOMBRE_FAMILIAR"
    export EXCLUIR_FECHAS="DD/MM/AAAA"  # opcional
    # stderr a log, siempre exit 0 para no disparar alertas de error
    python3 edus_citas_watchdog.py 2>> edus_errors.log || true
fi
```

Cron: `*/5 5-7 * * *` (cada 5 minutos, 5am-7:59am)

---

## Stack Técnico

| Componente | Detalle |
|-----------|---------|
| **Framework** | JSF 2.x + PrimeFaces 12.0.0 (theme: barcelona-ccss) |
| **Servidor** | Oracle WebLogic (cookie: `CitasWebPFCK`) |
| **State** | `javax.faces.ViewState` (GZIP + Base64, ~30KB, rota por request) |
| **AJAX** | POST con `javax.faces.partial.ajax=true` |
| **Encoding** | ISO-8859-1 en respuestas AJAX |
| **Cookies** | `JSESSIONID` + WebLogic cookie (Secure) |
| **Monitoreo** | Dynatrace RUM (`ruxitagentjs`) |
| **ProjectStage** | Development (debug info visible) |

---

## Pitfalls

- **ViewState obligatorio**: Sin ViewState en cada POST, el servidor rechaza la petición
- **Sesión JSF expira rápido**: Cada ciclo de monitoreo debe iniciar sesión fresca
- **CAPTCHA impredecible**: Tasa de acierto ~30-40%, requiere retry loop de 20-30 intentos
- **IDs con `:` en JSF**: Usar `getElementById()` en vez de `querySelector()` para evitar problemas de escape CSS
- **AJAX asíncrono**: Esperar 2-5s entre cada paso (cambio de select → carga de dependientes)
- **Encoding**: Respuestas AJAX usan ISO-8859-1, no UTF-8
- **Cupos agotados**: Una fecha puede aparecer pero sin horarios ("gestionó todas las citas disponibles")
- **Dos botones "Agregar una cita"**: Uno para el titular, otro para el familiar. Identificar por posición en el DOM
- **Horario limitado**: Los cupos nuevos suelen aparecer solo entre 5am-8am. Monitorear 24/7 es innecesario
- **f-strings Python → JS**: Al generar código JavaScript desde Python con f-strings, las variables de entorno deben interpolarse con `{VAR}`. Si dejás `NOMBRE_VAR` sin llaves en el f-string, Python lo pasa como texto literal y el navegador lanza `ReferenceError`. Solución: evaluá condiciones en Python (`if var:`) y generá código JS diferente para cada caso. Ver ejemplo en Fase 4.

---

## Dependencias

```bash
pip install playwright pillow
playwright install chromium
sudo apt install tesseract-ocr tesseract-ocr-spa
```

---

## Configuración en Hermes Agent

```yaml
# En el cron job de Hermes:
#   script: edus_citas_schedule.sh
#   schedule: every 5m
#   no_agent: true
#   deliver: telegram  # o la plataforma que uses
```

El script wrapper (`edus_citas_schedule.sh`) exporta las variables de entorno necesarias y solo ejecuta en el horario correcto.

---

## Adaptación para otros agentes

Esta guía funciona con cualquier agente que pueda ejecutar **Playwright + Python**. Los conceptos clave son:

1. **JSF/PrimeFaces no es REST**: requieres un navegador (Playwright) para manejar ViewState y AJAX
2. **CAPTCHA**: tesseract con preprocesamiento PIL es suficiente (~30-40% por intento)
3. **Familiar sin cuenta propia**: login del titular → navegar a sección familiar
4. **Watchdog silencioso**: solo notificar cuando hay acción (cupos encontrados o errores)
5. **Ventana horaria**: 5am-8am CST es cuando hay probabilidad real de cupos

---

## Referencia Rápida de IDs del DOM

| Elemento | ID |
|----------|-----|
| Form login | `formInicioSesion` |
| Tipo identificación | `formInicioSesion:tipIdentificacion_input` |
| Usuario | `formInicioSesion:usuario` |
| Clave | `formInicioSesion:clave` |
| CAPTCHA input | `formInicioSesion:captchaDigitado` |
| Botón login | `formInicioSesion:ejecutarPaso1` |
| Form principal | `formSIAC` |
| Botón agregar cita (titular) | `formSIAC:btnMenuAdd` |
| Select servicio | `formSIAC:menuServicios_input` |
| Select especialidad | `formSIAC:menuEspecialidades_input` |
| Tabla cupos | `formSIAC:cuposDisponibles` |
| Tabla familiares | `formSIAC:tablaFamiliares` |
| Tabla citas familiar | `formSIAC:tablaCitasFam` |

## Tipos de Identificación

| Valor | Descripción | Dígitos |
|-------|-------------|---------|
| 0 | Cédula de identidad (nacional) | 9 |
| 6 | Identificación temporal/interno | variable |
| 7 | Extranjero con identificación CCSS | 11 |
