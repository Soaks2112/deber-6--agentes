# Equipo de agentes para revisar `usuarios.py`

## Parte 1 — Equipo y flujo

| Agente | Misión | Herramientas autorizadas | Entrada | Salida | Límites |
|---|---|---|---|---|---|
| Orquestador | Repartir la revisión de `usuarios.py` y consolidar resultados. | Ninguna propia; solo delega y consolida. | Ruta `usuarios.py`. | Tabla con hallazgos y casos de prueba. | No modifica archivos ni inventa hallazgos. |
| Especialista 1 — Seguridad | Buscar secretos escritos en código y entradas de usuario sin validar. | Leer y buscar. | Ruta `usuarios.py`. | Lista con línea, problema y severidad. | Solo revisa ese archivo; no escribe ni divulga secretos completos. |
| Especialista 2 — Pruebas | Proponer tres casos de prueba faltantes. | Leer. | Ruta `usuarios.py`. | Lista con nombre del caso y lo que verifica. | Solo propone pruebas; no las implementa ni modifica archivos. |

### Flujo paralelo

```text
                       ┌───────────────────────────────────────┐
                       │ Orquestador: recibe `usuarios.py`      │
                       └───────────────┬───────────────────────┘
                                       │
                 ┌─────────────────────┴─────────────────────┐
                 ▼                                           ▼
┌───────────────────────────────────────┐   ┌───────────────────────────────────────┐
│ Especialista 1: Seguridad              │   │ Especialista 2: Pruebas                │
│ secretos y entradas no validadas       │   │ tres casos faltantes                    │
└──────────────────┬────────────────────┘   └──────────────────┬────────────────────┘
                   └─────────────────────┬─────────────────────┘
                                         ▼
                       ┌───────────────────────────────────────┐
                       │ Orquestador: consolida en una tabla    │
                       └───────────────┬───────────────────────┘
                                       ▼
                 ╔═══════════════════════════════════════════════╗
                 ║ CHECKPOINT HUMANO: valida evidencia y alcance ║
                 ╚═══════════════════════┬═══════════════════════╝
                                         ▼
                       ┌───────────────────────────────────────┐
                       │ Aprobación para ejecutar cambios o PR  │
                       └───────────────────────────────────────┘
```

### Consolidado esperado

| Fuente | Resultado |
|---|---|
| Seguridad | Línea 10: clave API escrita en código (alta); línea 11: contraseña administrativa escrita en código (alta); línea 21: concatenación de `nombre` en SQL sin parametrizar (alta); línea 42: comparación directa con contraseña hardcodeada (alta). Los valores se mantienen redactados. |
| Pruebas | 1. `buscar_usuario` debe usar consulta parametrizada aun si `nombre` contiene comillas o texto de inyección. 2. `crear_usuario` debe guardar correctamente nombre, edad y correo con valores válidos. 3. `calcular_descuento` debe devolver el precio esperado para 0 %, 100 % y un porcentaje intermedio. |

### Checkpoint humano

Antes de aprobar, la persona comprueba que cada línea citada existe, que la evidencia no expone secretos y que cada caso de prueba se relaciona con una función real. Rechaza el resultado si un hallazgo no está en el archivo, si la severidad no se justifica, si un caso no tiene sentido o si un especialista salió de su alcance.

El orquestador detecta un fallo si la salida está vacía sin explicación, cita líneas inexistentes, inventa código o ignora el límite de un archivo y de no escritura. En ese caso valida las líneas, pide una corrección o reintenta una vez con instrucciones más concretas; si persiste, avisa a la persona y no presenta el resultado como aprobado.

## A2A

*Simplificación didáctica; no es JSON conforme a una versión específica de A2A.*

```json
{
  "name": "especialista-pruebas-remoto",
  "description": "Propone casos de prueba faltantes para un archivo Python.",
  "skills": [
    {
      "id": "proponer-pruebas-python",
      "description": "Analiza un archivo y devuelve tres casos de prueba sugeridos."
    }
  ],
  "interface": {
    "endpoint": "https://agente.ejemplo/a2a",
    "protocol": "A2A ilustrativo"
  },
  "capabilities": { "streaming": true },
  "security": {
    "authentication": "OAuth 2.0 con token de audiencia",
    "transport": "HTTPS",
    "authorization": "solo solicitudes con la skill autorizada"
  }
}
```

El cliente podría obtener esta Agent Card desde una URL conocida como `https://agente.ejemplo/.well-known/agent-card.json` o desde un registro de agentes. Antes de confiar, valida el origen y certificado, que ofrece la skill requerida, una interfaz compatible y la autenticación exigida. Descubrirla no demuestra que sea confiable.

```text
obtener Agent Card
→ verificar skill, interfaz y seguridad
→ enviar mensaje con la tarea
→ respuesta inmediata o creación de Task
→ consultar o recibir estado
→ obtener resultado (Artifact)
```

Camino simplificado de una Task: `submitted → working → completed`. Si llega a `input-required`, el cliente muestra qué dato falta y solo reanuda tras obtenerlo de una persona autorizada. Si llega a `failed`, registra el motivo, no usa un resultado parcial como válido y puede reintentar según una política o escalar a la persona.

Frente al subagente local, A2A implica una interfaz de red, descubrimiento de capacidades, interoperabilidad y una frontera de confianza entre organizaciones. Sus permisos y datos deben negociarse y validarse; el subagente local hereda el entorno del harness. MCP conecta al orquestador con herramientas o recursos —por ejemplo, acceso a archivos—, mientras que A2A delega un trabajo autónomo a otro agente remoto. La elección depende de la arquitectura y de la interfaz de comunicación, no de que la tarea lea archivos.
