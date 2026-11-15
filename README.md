[README.md](https://github.com/user-attachments/files/34885959/README.md)
# Checkpoint 4 – Integraciones avanzadas y blindaje del flujo (Demo Valores)

Pre-entrega del Módulo 4 del curso **AI Automation Avanzado**. Parte del workflow del Módulo 3 (Manager con memoria en Airtable y dos Workers) y le suma tres integraciones reales: una casilla de correo (**Gmail**), un **CRM** (HubSpot) y un canal del equipo (**Slack**), con compuertas de seguridad para evitar bucles, duplicados y payloads mal formados.

> Caso: onboarding de inversores de CEDEARs de **Demo Valores**, un broker ficticio creado con fines de demostración.

**Archivo a importar:** [`checkpoint4_nombre_apellido.json`](./checkpoint4_nombre_apellido.json)

## Credenciales activas en n8n

![Sección Credentials de n8n con las tres conexiones activas](./img/credentials.png)

> Reemplazar por la captura de **Credentials** con Gmail, HubSpot y Slack en verde.

| Rol | Herramienta | Tipo de credencial en n8n | Autenticación |
|---|---|---|---|
| Casilla de correo | Gmail | Gmail OAuth2 API | OAuth2 |
| CRM | HubSpot | HubSpot App Token API | Service Key de HubSpot con scopes acotados |
| Canal del equipo | Slack | Slack API | Token de bot (`xoxb-`) |

## Arquitectura del flujo

```
Gmail Trigger (solo asuntos "Demo Valores")
   │
   ▼
① IF "¿Es auto-reply?" ── Sí ──▶ Ignorar auto-respuesta (corta el bucle)
   │ No
   ▼
④ Set "Limpiar payload del correo"  (deja remitente, asunto, cuerpo recortado y thread)
   │
   ▼
IF "¿Email válido?" ── No ──▶ Descartar (evita el error 400)
   │ Sí
   ▼
Normalizar entrada (email) ─┐
                            ├─▶ Entrada normalizada ─▶ Memoria (Airtable) ─▶ Clasificador ─▶ Worker 1 / Worker 2
Normalizar entrada (chat) ──┘                                                                   │
(Chat Trigger de M3, sigue funcionando)                                                         ▼
                                                                                   Responder al usuario
                                                                                                │
                                                                    IF "¿Es chat?" ── Sí ──▶ (fin: responde en el chat)
                                                                                                │ No (es correo)
                                                                                                ▼
                                                                ② Look up en HubSpot (búsqueda por email)
                                                                      ├─ existe ──▶ Actualizar contacto
                                                                      └─ no existe ─▶ Crear contacto
                                                                                                │
                                                                                                ▼
                                                                ③ Gmail "Create Draft" (borrador, HITL)
                                                                                                │
                                                                                                ▼
                                                                    Set "Limpiar payload para Slack"
                                                                                                │
                                                                                                ▼
                                                                      Slack: aviso de borrador listo
```

Los cuatro nodos que evalúa la rúbrica están numerados (①–④) en el diagrama.

## Compuertas de seguridad

| # | Nodo | Qué previene | Cómo funciona |
|---|---|---|---|
| ① | `¿Es auto-reply?` (IF, justo después del trigger) | Bucle infinito de auto-respuestas | Condiciones visuales combinadas con OR, sin distinguir mayúsculas: el asunto contiene *auto-reply, autoreply, automatic reply, out of office, undeliverable, delivery status notification, respuesta automática, fuera de la oficina*; el remitente contiene *no-reply, noreply, mailer-daemon, postmaster*; o el header `Auto-Submitted` es distinto de `no`. Si alguna se cumple, el flujo termina. |
| ② | `Look up - ¿existe el contacto en el CRM?` + IF | Duplicados de contactos (error 409) | Busca en HubSpot por email **antes** de escribir. Si existe se actualiza; si no, se crea. |
| ③ | `Crear borrador (HITL)` (Gmail, operación *Create*) | Envío sin supervisión | El flujo **nunca envía**: deja la respuesta de la IA en Borradores de Gmail y una persona la revisa y la manda. No hay ningún nodo de envío en el workflow. |
| ④ | `Limpiar payload del correo` (Set) + `¿Email válido?` (IF) | Payloads mal formados (error 400) y cuerpos pesados | Conserva solo remitente, nombre, asunto, cuerpo (recortado a 4.000 caracteres, sin el historial citado) y thread. Verifica que el email tenga formato válido antes de seguir. |

Además, antes de Slack hay un segundo Set (`Limpiar payload para Slack`) que deja solo trace_id, remitente, asunto y los primeros 300 caracteres del borrador, sin objetos binarios ni payloads grandes.

## Mínimo privilegio

- **Gmail:** el workflow usa únicamente el trigger de lectura y la operación *Create Draft*. n8n define los scopes de la credencial de Gmail OAuth2 y no se pueden reducir desde la interfaz; el control está en que el flujo no contiene operaciones de envío ni de borrado. El trigger además está acotado con el filtro `in:inbox subject:"Demo Valores"` y solo correos no leídos, para no procesar el resto de la casilla.
- **HubSpot:** la clave tiene solamente los scopes `crm.objects.contacts.read` y `crm.objects.contacts.write`. Se usa una **Service Key** porque HubSpot retiró la creación de *private apps* (legacy) y n8n la acepta en la credencial *HubSpot App Token*.
- **Slack:** bot con permiso de escritura en el canal de observabilidad.
- El correo del remitente no se guarda en la memoria: el `Session_ID` de los correos es `mail-` + id del hilo. El correo solo aparece en el CRM.

## Pruebas de regresión

| # | Caso | Resultado esperado | Resultado |
|---|---|---|---|
| 1 | Correo normal con "Demo Valores" en el asunto | Crea el contacto en HubSpot, deja un borrador en Gmail y avisa por Slack | ✅ / ❌ |
| 2 | Segundo correo desde la misma cuenta | El Look up encuentra al contacto: se actualiza, no se duplica; el aviso dice "ya existía en el CRM" | ✅ / ❌ |
| 3 | Correo con asunto `Auto-reply: Demo Valores` | Termina en "Ignorar auto-respuesta": sin contacto, sin borrador y sin aviso | ✅ / ❌ |

> Completar con ✅ o ❌ y, si querés, agregar capturas en `./img/`.

## Cómo importarlo

1. En n8n: **Workflows → Import from file** y elegir `checkpoint4_nombre_apellido.json`.
2. Asignar las credenciales propias en los nodos de Gmail, HubSpot y Slack, y las de Airtable y Google Gemini del flujo heredado de M3.
3. En el nodo *Ejecutar Worker 1 - Informativo*, seleccionar el workflow del Worker 1; el Worker 2 también se elige desde la lista.
4. Para probar sin activar el workflow: **Execute workflow** y, en el Gmail Trigger, **Fetch test event**.

El archivo **no contiene secretos**: las credenciales se vinculan en cada instancia de n8n y nunca se exportan sus valores.

## Evolución del proyecto

| Módulo | Aporte |
|---|---|
| M1 | Agente informativo con tool de consulta del manual y log de observabilidad |
| M2 | Arquitectura Manager–Worker con contrato de datos y manejo de fallas |
| M3 | Memoria persistente por sesión y resumen automático en Airtable |
| **M4** | **Integraciones con Gmail, HubSpot y Slack, y compuertas de seguridad** |

## Autor

[Nombre y Apellido] – AI Automation Avanzado
