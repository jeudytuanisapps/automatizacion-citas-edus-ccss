> Sáquele la cita a su agente, no a la fila.

---

# automatizacion-citas-edus-ccss

**Un skill para que su agente de IA le saque las citas del EDUS.**

Si alguna vez intentó sacar cita por el EDUS, ya sabe el chiste: no hay citas. Nunca. Hay que pagarle a la señora del barrio que va a hacer fila.

No más, mi [Hermes Agent](https://hermes-agent.nousresearch.com/) aprendió a manejar el EDUS por su cuenta, y escribió esta guía documentando todo lo que descubrió. La publico para que usted no tenga que repetir el proceso.

Corría con el modelo **GLM-5.2**, por si le sirve el dato.

---

## Ojo: esto es un skill, no un programa

Este repo tiene **un solo archivo**: [`EDUS-Citas-Automation-Guide.md`](EDUS-Citas-Automation-Guide.md).

No hay nada que compilar, ni dependencias, ni configuración previa. Es conocimiento en Markdown — cómo está armado el EDUS por dentro, el flujo de reserva, los IDs, los errores típicos y todos los tropiezos que el agente encontró en el camino.

Se lo pasa a su agente y **él resuelve el cómo** con las herramientas que tenga. Uno con navegador lo hará de una forma, otro escribiendo código lo hará de otra. La guía no lo amarra a ningún método.

Por eso corre igual en Hermes, OpenClaw, Claude Code, Codex, o lo que esté usando.

Trae frontmatter estándar, así que casi todos los agentes lo detectan solos:

```yaml
name: edus-citas-automation-guide
description: Guía genérica para automatizar la reserva de citas médicas en EDUS (CCSS Costa Rica) con cualquier AI agent — sin datos personales.
category: productivity
```

Y no, **no tiene datos personales**. Ninguna cédula, ningún nombre, ninguna clave.

---

## El alcance, para que no haya sorpresas

- Solo saca citas **para usted y para su grupo familiar ya registrado en EDUS**. No es para terceros ni para uso masivo.
- Es exactamente lo mismo que haría a mano desde la app o el sitio de la CCSS. El agente hace los clics, nada más.
- Esto no es oficial de la CCSS ni tiene nada que ver con ellos.

---

## Hablemos de sus credenciales 🔐

Su agente va a necesitar su cédula y su clave del EDUS. Antes de dárselas, lea esto:

- Esas credenciales abren su **expediente digital único en salud** — el suyo y el de su grupo familiar. No es una cuenta de Netflix.
- Corra esto **solo en un agente suyo, en hardware suyo**. Nunca en una instancia compartida, pública o de un tercero.
- Páselas por **variables de entorno**, no escritas en el chat ni en un archivo del workspace. La guía ya asume ese patrón: `EDUS_CEDULA`, `EDUS_CLAVE`, `FAMILIAR_CEDULA`.
- Fíjese a qué modelo le está mandando el contexto. Con un proveedor en la nube, **sus credenciales pueden quedar en los logs de ellos**. Con un modelo local, ese problema no existe.
- No exponga el gateway de su agente a Internet sin autenticación y TLS.

> Si algo de esto no le queda claro, mejor saque la cita a mano. En serio.

---

## ¿Qué agente uso?

El que quiera, mientras pueda navegar y ejecutar. Los más comunes:

| Agente | Dónde | Instalación |
|---|---|---|
| **Hermes Agent** | [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com/) · [GitHub](https://github.com/NousResearch/hermes-agent) | `curl -fsSL https://hermes-agent.nousresearch.com/install.sh \| bash` |
| **OpenClaw** | [openclaw.ai](https://openclaw.ai/) · [GitHub](https://github.com/openclaw/openclaw) | `npm install -g openclaw@latest` → `openclaw onboard --install-daemon` |
| **Claude Code** | [claude.com/product/claude-code](https://claude.com/product/claude-code) | Ver docs oficiales |
| **Codex** | [openai.com/codex](https://openai.com/codex/) | Ver docs oficiales |

Docs de instalación con más detalle:
- Hermes — [Quickstart](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart) · [Instalación](https://hermes-agent.nousresearch.com/docs/getting-started/installation)
- OpenClaw — [docs.openclaw.ai](https://docs.openclaw.ai/) · [ClawHub](https://clawhub.ai/)

Dos notas rápidas:

> Hermes pide un modelo con mínimo **64K de contexto**. GLM-5.2, Claude, GPT, Gemini, DeepSeek y Qwen pasan sobrados.
>
> OpenClaw recomienda **no exponer el Gateway directo a Internet** — reverse proxy con TLS y mínimo privilegio.

---

## Instalarlo

### La forma rápida (sin instalar nada)

Pásele el link y ya:

```
Lea https://github.com/jeudytuanisapps/automatizacion-citas-edus-ccss/blob/main/EDUS-Citas-Automation-Guide.md
y siga esa guía para sacarme una cita en el EDUS.
```

O baja el `.md` y lo adjunta al chat. Funciona igual de bien.

### La forma permanente (como skill)

Si lo va a usar seguido, déjelo instalado:

```bash
git clone https://github.com/jeudytuanisapps/automatizacion-citas-edus-ccss.git
cd automatizacion-citas-edus-ccss

# Hermes Agent
mkdir -p ~/.hermes/skills/edus-citas
cp EDUS-Citas-Automation-Guide.md ~/.hermes/skills/edus-citas/SKILL.md

# OpenClaw
mkdir -p ~/.openclaw/workspace/skills/edus-citas
cp EDUS-Citas-Automation-Guide.md ~/.openclaw/workspace/skills/edus-citas/SKILL.md

# Claude Code
mkdir -p ~/.claude/skills/edus-citas
cp EDUS-Citas-Automation-Guide.md ~/.claude/skills/edus-citas/SKILL.md
```

Casi todos los agentes esperan el archivo como `SKILL.md` dentro de una carpeta — por eso lo renombramos. Revise la ruta exacta en las docs de su versión, que puede cambiar.

Reinicie el agente y confirme que aparece en la lista.

---

## Cómo se lo pide

### La primera vez

Antes de darle credenciales, deje que lea y le cuente el plan:

```
Lea el skill de citas EDUS. Cuénteme qué va a hacer, qué herramientas
necesita y qué información me va a pedir. No ejecute nada todavía.
```

Así revisa antes de exponer nada. Si el plan no le gusta, lo corrige ahí mismo.

### Sacar su cita

```
Use el skill de citas EDUS y sáqueme una cita de medicina general.
Mis credenciales están en EDUS_CEDULA y EDUS_CLAVE.
```

### Sacarle cita a un familiar

Su grupo familiar no necesita credenciales propias — se entra con las suyas:

```
Use el skill de citas EDUS. Entre con mis credenciales (EDUS_CEDULA,
EDUS_CLAVE) y sáquele una cita de medicina general a mi familiar
con cédula FAMILIAR_CEDULA.
```

### El watchdog (lo bueno de verdad)

Aquí es donde esto se pone interesante. La guía documenta que los cupos **se liberan entre 5am y 8am hora Costa Rica**. Fuera de esa ventana casi no hay nada, así que monitorear 24/7 es desperdicio.

Le dice que vigile y reserve solo:

```
Arme un cron con el skill de citas EDUS que revise cupos de medicina
general entre 5am y 8am hora Costa Rica, reserve el primero que
encuentre, y me avise por Telegram cuando lo haga.
```

En Hermes eso va como cron job con `no_agent: true` y entrega por Telegram. Cada 5 minutos en esa ventana es más que suficiente.

Y listo — se despierta con la cita ya sacada.

### Si prefiere decidir usted

También puede dejarlo en modo chismoso, que solo avise:

```
Use el skill de citas EDUS. Revise si hay cupos de medicina general
y dígame cuáles hay, pero no reserve nada.
```

### Tips

- **Sea específico con servicio y especialidad.** "Medicina general" es lo común, pero el EDUS tiene un montón más.
- **Pídale que confirme.** Fecha, hora y consultorio.
- **Verifique en la app oficial.** Siempre. El cupo se puede haber tomado entre que lo vio y lo reservó.
- **Si falla, pídale el error textual.** La guía tiene tabla de errores del EDUS y qué significa cada uno.

---

## Qué trae la guía por dentro

| Sección | De qué va |
|---|---|
| Arquitectura | Los dos sistemas de citas de la CCSS y en qué difieren |
| Fase 1 | Reconocimiento sin credenciales |
| Fase 2 | El login |
| Fase 3 | La reserva: servicio → especialidad → cupos → confirmar |
| Fase 4 | Citas para el grupo familiar |
| Fase 5 | Monitoreo recurrente y la ventana horaria que sí sirve |
| Pitfalls | Todo con lo que el agente se topó, para que usted no |
| Referencia | IDs del DOM y tipos de identificación |

---

## El descargo de siempre

Esto se publica **tal cual, sin garantías**. Si le queda mal la cita, pierde un cupo, le bloquean la cuenta o se le filtran las credenciales, corre por su cuenta.

La CCSS puede cambiar el EDUS cuando le dé la gana y romper todo sin avisar. Úselo bajo su propio riesgo y **verifique siempre en la app oficial**.

---

## ¿Quiere aportar?

Issues y PRs bienvenidos, sobre todo:
- Arreglos cuando la CCSS cambie el flujo.
- Reportes de qué agentes y modelos funcionan.
- Mejoras al manejo de credenciales.

**Nunca suba credenciales, cédulas, números de asegurado ni capturas con datos personales.** Ni en issues ni en PRs.

---

## Tutoriales en español

Material de la comunidad para montar los agentes. No son oficiales ni tienen relación con este repo.

**Hermes Agent**
- [Curso de Hermes Agente — playlist completa](https://www.youtube.com/playlist?list=PLWUX-KZsnKXT2gprZvjTymfu3KBIcxSL9)
- [HERMES 2026: Curso COMPLETO en Español](https://www.youtube.com/watch?v=FQZGoSwdS_0)
- [Hermes Agent + Nous Research como proveedor](https://www.youtube.com/watch?v=8L-V4iLtBBU)

**OpenClaw**
- [OpenClaw de NOVATO a PRO — curso completo](https://www.youtube.com/watch?v=JVA09oUTXzM)
- [OpenClaw para Principiantes: Setup Completo (Fazt)](https://www.youtube.com/watch?v=f_G3q3yuGFk)
- [Guía completa de OpenClaw](https://www.youtube.com/watch?v=jbvRC7LZcLI)
- [Probé 5 formas de instalar OpenClaw… esta es la mejor](https://www.youtube.com/watch?v=jDX4n9yc6v8)
- [Instala OpenClaw de forma segura](https://www.youtube.com/watch?v=od9iUOW5Znc)

---

## Licencia

MIT

---

<sub>Hecho en Costa Rica 🇨🇷 por alguien que se cansó de madrugar para nada.</sub>
