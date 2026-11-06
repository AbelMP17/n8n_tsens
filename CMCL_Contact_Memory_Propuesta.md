# CMCL — Propuesta de memoria de contacto en Retell + n8n

Fuente: docs.retellai.com (features/contacts, features/contact-memory, API reference Update/Create/Get Contact by Phone), leídas el 2026-11-06.

## 1. Qué inyecta Retell en el prompt
Al empezar una llamada o SMS, Retell busca el contacto por número de teléfono (el del llamante en entrantes) y pasa como variables dinámicas:
- `{{first_name}}`
- `{{last_name}}`
- `{{do_not_call}}`
- `{{contact_memory}}` (solo si hay memoria guardada)
- una variable por cada campo personalizado, con el nombre del campo.

Si no hay contacto para ese número, las variables no existen. Hoy el prompt usa `{{first_name}}` en la frase de conversación extensa; sin contacto puede quedar sin sustituir.
Con **Use contact memory** activo (`contact_memory_config.enable_read`), la memoria también se añade al contexto automáticamente, sin escribir la variable en el prompt.

## 2. Cómo se guardan first_name y last_name
No se guardan automáticamente a partir de lo que dice el llamante. Retell solo crea el contacto (con el teléfono) al terminar una llamada, y deja nombre y apellidos vacíos. Se rellenan por:
- edición manual o importación CSV/XLSX;
- sincronización con un CRM;
- Post Call Extraction mapeado a `first_name`/`last_name` (modos Overwrite, Fill only if empty o Merge);
- **API**: `PATCH https://api.retellai.com/update-contact/{contact_id}` con `first_name` y `last_name` (también acepta `contact_memory`, máx. 2.000 caracteres, y `custom_fields`).

Endpoints auxiliares:
- `GET https://api.retellai.com/get-contact-by-phone/{phone_e164}` devuelve `contact_id` (404 si no existe).
- `POST https://api.retellai.com/create-contact` (obligatorio `phone_number` en E.164; admite `first_name` y `last_name`). Falla si el número ya existe.

La **memoria automática** (`enable_update`) es un resumen libre que Retell reescribe tras cada llamada (máx. 2.000 caracteres, 0,005 $ por ejecución). No es solo nombre: puede guardar motivo de cita, especialidad o datos de salud.

## 3. Conflicto con las reglas decididas
La regla "se guarda solo nombre y apellidos" no se cumple con la memoria automática por defecto. Opciones:
- **Recomendada:** `enable_read: true` y `enable_update: false` en ambos agentes; los nombres se escriben por API desde n8n. Así es determinista y no se almacena nada clínico.
- Alternativa: dejar `enable_update` y cambiar el Memory prompt del workspace a algo como "Guarda solo nombre y apellidos del paciente verificado; ningún otro dato". Es menos fiable porque lo decide un modelo.

## 4. Flujo n8n propuesto
Nueva tool custom del agente: `save_contact_name`, disparada solo tras una verificación correcta (documento, o nombre más fecha de nacimiento) y solo si quien llama es el paciente.
- Body de Retell: `args` (`nombre`, `apellidos`) y `call` (incluye `from_number`).
- Workflow n8n `/webhook/cmcl/save-contact-name`:
  1. Webhook (POST).
  2. HTTP Request `GET /get-contact-by-phone/{from_number}` con credencial Bearer de Retell guardada en n8n (nunca en el workflow exportado). Activar "never error" para leer el 404.
  3. IF existe `contact_id`:
     - sí → HTTP Request `PATCH /update-contact/{contact_id}` con `{first_name, last_name}`;
     - no → HTTP Request `POST /create-contact` con `{phone_number, first_name, last_name}`.
  4. Respond to Webhook `{ok: true}`.
- Mejor tomar nombre y apellidos de la respuesta de iGaleno (`get_patient_data`) que de lo que dictó el llamante. No guardar teléfonos ni documentos.
- Si el teléfono es oculto o no hay `from_number`, no guardar.
- Nota: el formato exacto del body de las custom functions (`args` y `call`) viene de la documentación de funciones personalizadas, que no estaba entre las dos páginas leídas. Conviene verificarlo en el primer test.

## 5. Cambios de prompt propuestos (añadir al general_prompt)
Nueva sección tras IDENTIFICACIÓN DEL PACIENTE:

```
## MEMORIA DEL LLAMANTE (solo para saludar)
- Si tienes {{first_name}} (y opcionalmente {{last_name}}), tras la apertura y antes de pedir ningún dato pregunta: "¿Hablo con {{first_name}} {{last_name}}?". Si no tienes {{first_name}}, no digas ningún nombre.
- Si confirma, salúdale por su nombre. Si dice que no, o que llama por otra persona, no uses ese nombre ni lo mezcles con la gestión.
- La memoria NUNCA sustituye a la identificación del paciente: sigue siendo obligatorio documento, o nombre más fecha de nacimiento, antes de dar datos o gestionar citas, aunque el nombre coincida.
- Quien llama puede no ser el paciente. No des por hecho que el contacto es el paciente.
- Nunca digas datos de citas, documento ni fecha de nacimiento a partir de la memoria.
- Tras una verificación correcta, si quien llama es el propio paciente, llama a `save_contact_name` con nombre y apellidos tal como figuran en el sistema. Si llama por otra persona, no la llames. No guardes nada más.
```
Y cambiar la frase de conversación extensa para que no dependa del nombre: "Hagamos una cosa, como veo que tiene muchas consultas voy a pasarle con recepción. Gracias."

Añadir la tool `save_contact_name` en `general_tools` de ambos agentes (en el de fuera de horario también: guardar el nombre no requiere humanos).

## 6. Privacidad
Los datos de salud están en categoría especial del RGPD. Con `enable_update: false` y solo nombre y apellidos en el contacto, el riesgo es menor, pero conviene informar en la política de privacidad del cliente. El contacto se asocia al teléfono del llamante, no al paciente, y puede ser un teléfono compartido (familia).
