# automatizacion-citas-edus-ccss

**Skill para que agentes de IA saquen citas médicas en EDUS (CCSS) a tu nombre.**

Mi [Hermes Agent](https://hermes-agent.nousresearch.com/) aprendió a sacar citas en el EDUS de la CCSS y generó este skill por su cuenta. Lo publico para que cualquier persona con un agente similar (Hermes, OpenClaw, etc.) pueda reutilizarlo sin repetir el proceso de descubrimiento.

El agente que lo creó corría con el modelo **GLM-5.2**.

---

## ⚠️ Antes de empezar — léalo completo

### Alcance
- Solo se pueden sacar citas **para uno mismo y para el grupo familiar registrado en EDUS**. No sirve para terceros ni para uso masivo.
- Es exactamente lo mismo que usted haría a mano desde la app EDUS o el sitio de citas de la CCSS: el agente automatiza los clics, no abre puertas nuevas.
- No es un producto oficial de la CCSS. No hay afiliación ni respaldo institucional.

### Credenciales
Para funcionar, **el agente le va a pedir sus credenciales del EDUS**. Tome esto muy en serio:

- Sus credenciales dan acceso a su **expediente digital único en salud**. Es información médica sensible, suya y de su grupo familiar.
- Use este skill **únicamente en un agente autohospedado**, corriendo en hardware que usted controla. Nunca en una instancia compartida, pública o de un tercero.
- Prefiera guardar las credenciales en el gestor de secretos / variables de entorno de su agente, no en texto plano dentro de un prompt o un archivo del workspace.
- Revise a qué modelo está enviando el contexto. Si usa un proveedor en la nube, **sus credenciales pueden terminar en los logs del proveedor**. Un modelo local elimina ese riesgo.
- No exponga el gateway de su agente a Internet sin autenticación y TLS.

> Si algo de lo anterior no le queda claro, saque la cita a mano. No vale la pena.

---

## Requisitos

- Un agente de IA con soporte para skills y ejecución de herramientas (navegador/HTTP).
- Un modelo con ventana de contexto amplia y buen tool-calling. Probado con **GLM-5.2**.
- Cuenta activa en EDUS con su grupo familiar ya registrado.

---

## Instalación del agente

Puede usar cualquiera de los dos. Ambos son open source, self-hosted y soportan skills.

### Opción A — Hermes Agent (Nous Research)

| Recurso | Enlace |
|---|---|
| Sitio oficial | https://hermes-agent.nousresearch.com/ |
| Repositorio | https://github.com/NousResearch/hermes-agent |
| Documentación | https://hermes-agent.nousresearch.com/docs/ |
| Quickstart | https://hermes-agent.nousresearch.com/docs/getting-started/quickstart |
| Instalación | https://hermes-agent.nousresearch.com/docs/getting-started/installation |

Instalación por terminal (Linux, macOS, WSL2):

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
hermes doctor      # verifica la instalación
hermes setup       # asistente de configuración (proveedor, modelo, tools)
```

También hay instaladores de escritorio para macOS y Windows en el sitio oficial.

> Hermes exige un modelo con al menos **64K tokens de contexto**; los modelos con ventana menor se rechazan al arrancar.

### Opción B — OpenClaw

| Recurso | Enlace |
|---|---|
| Sitio oficial | https://openclaw.ai/ |
| Repositorio | https://github.com/openclaw/openclaw |
| Documentación | https://docs.openclaw.ai/ |
| Directorio de skills (ClawHub) | https://clawhub.ai/ |

Instalación (requiere Node.js 20+):

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

> Recomendación de seguridad de los propios mantenedores: **no exponga el Gateway directamente a Internet.** Use un reverse proxy con TLS y principio de mínimo privilegio.

---

## Instalación del skill

1. Clone este repositorio:

   ```bash
   git clone https://github.com/<usuario>/automatizacion-citas-edus-ccss.git
   ```

2. Copie el skill al directorio de skills de su agente:

   ```bash
   # Hermes Agent
   cp -r automatizacion-citas-edus-ccss ~/.hermes/skills/

   # OpenClaw
   cp -r automatizacion-citas-edus-ccss ~/.openclaw/workspace/skills/
   ```

   > Verifique la ruta exacta en la documentación de su versión; puede variar según cómo instaló el agente.

3. Reinicie el agente y confirme que el skill aparece listado.

4. Pídale al agente algo como:

   ```
   Sacame una cita en el EBAIS para <nombre del familiar>
   ```

   El agente le va a solicitar las credenciales de EDUS en ese momento.

---

## Descargo de responsabilidad

Este proyecto se publica **tal cual, sin garantías**, con fines educativos y de automatización personal. Ni el autor ni los colaboradores se hacen responsables por citas mal agendadas, pérdida de cupos, bloqueos de cuenta, filtración de credenciales o cualquier otro daño derivado del uso de este skill.

La CCSS puede cambiar el EDUS en cualquier momento y romper el skill sin aviso. Úselo bajo su propio riesgo y **siempre verifique la cita en la app oficial de EDUS**.

---

## Contribuciones

Issues y PRs son bienvenidos, sobre todo:
- Ajustes cuando la CCSS cambie el flujo del EDUS.
- Reportes de compatibilidad con otros modelos y agentes.
- Mejoras al manejo de credenciales.

**Nunca incluya credenciales, cédulas, números de asegurado ni capturas con datos personales en issues o PRs.**

---

## Tutoriales en español (YouTube)

Recursos de la comunidad para instalar y configurar los agentes. No son oficiales ni están afiliados a este repositorio.

### Hermes Agent
- [Curso de Hermes Agente — playlist completa en español](https://www.youtube.com/playlist?list=PLWUX-KZsnKXT2gprZvjTymfu3KBIcxSL9)
- [HERMES 2026: Curso COMPLETO en Español (Agente IA)](https://www.youtube.com/watch?v=FQZGoSwdS_0)
- [Hermes Agent + Nous Research como proveedor de IA](https://www.youtube.com/watch?v=8L-V4iLtBBU)

### OpenClaw
- [OpenClaw de NOVATO a PRO — Curso completo en español](https://www.youtube.com/watch?v=JVA09oUTXzM)
- [OpenClaw para Principiantes: Setup Completo y Configuración (Fazt)](https://www.youtube.com/watch?v=f_G3q3yuGFk)
- [Guía completa de OpenClaw: instalación, configuración y funciones](https://www.youtube.com/watch?v=jbvRC7LZcLI)
- [Probé 5 formas de instalar OpenClaw… esta es la mejor](https://www.youtube.com/watch?v=jDX4n9yc6v8)
- [Instala OpenClaw de forma segura — guía paso a paso](https://www.youtube.com/watch?v=od9iUOW5Znc)

---

## Licencia

MIT — ver [LICENSE](LICENSE).
