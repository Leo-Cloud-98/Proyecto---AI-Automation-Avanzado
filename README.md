[README.md](https://github.com/user-attachments/files/34885959/README.md)
# Checkpoint 4 – Integraciones avanzadas y blindaje del flujo (Demo Valores)

Pre-entrega del Módulo 4 del curso **AI Automation Avanzado**. Se consideró como base  el workflow del Módulo 3 (Manager con memoria en Airtable y dos Workers) y le suma tres integraciones reales: un correo electronico (**Gmail**), un **CRM** (HubSpot) y un canal del equipo (**Slack**), con compuertas de seguridad necesarias para evitar bucles, duplicados y payloads mal formados.

> Caso: onboarding de inversores de CEDEARs de **Demo Valores**, un broker ficticio creado con fines de demostración.



## Credenciales activas en n8n



| Rol | Herramienta | Tipo de credencial en n8n | Autenticación |
|---|---|---|---|
| Correo electrónico | Gmail | Gmail OAuth2 API | OAuth2 |
| CRM | HubSpot | HubSpot Service Key | Service Key de HubSpot con scopes acotados |
| Canal del equipo | Slack | Slack API | Token de bot |

## Arquitectura del flujo

```
Gmail Trigger (solo asuntos "Demo Valores")
   │
   ▼
  IF "¿Es auto-reply?" ── Sí ──▶ Ignorar auto-respuesta (corta el bucle)
   │ No
   ▼
  Set "Limpiar payload del correo"  (deja remitente, asunto, cuerpo recortado y thread)
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
                                                                    Look up en HubSpot (búsqueda por email)
                                                                      ├─ existe ──▶ Actualizar contacto
                                                                      └─ no existe ─▶ Crear contacto
                                                                                                │
                                                                                                ▼
                                                                    Gmail "Create Draft" (borrador, HITL)
                                                                                                │
                                                                                                ▼
                                                                    Set "Limpiar payload para Slack"
                                                                                                │
                                                                                                ▼
                                                                      Slack: aviso de borrador listo
```

## Compuertas de seguridad

| # | Nodo | Qué previene | Cómo funciona |
|---|---|---|---|
| ① | `¿Es auto-reply?` (IF, justo después del trigger) | Bucle infinito de auto-respuestas | Condiciones visuales combinadas con OR, sin distinguir mayúsculas: el asunto contiene *auto-reply, autoreply, automatic reply, out of office, undeliverable, delivery status notification, respuesta automática, fuera de la oficina*; el remitente contiene *no-reply, noreply, mailer-daemon, postmaster*; o el header `Auto-Submitted` es distinto de `no`. Si alguna se cumple, el flujo termina. |
| ② | `Look up - ¿existe el contacto en el CRM?` + IF | Duplicados de contactos (error 409) | Busca en HubSpot por email **antes** de escribir. Si existe se actualiza; si no, se crea. |
| ③ | `Crear borrador (HITL)` (Gmail, operación *Create*) | Envío sin supervisión | El flujo **nunca envía**: deja la respuesta de la IA en Borradores de Gmail y una persona la revisa y la manda. No hay ningún nodo de envío en el workflow. |
| ④ | `Limpiar payload del correo` (Set) + `¿Email válido?` (IF) | Payloads mal formados (error 400) y cuerpos pesados | Conserva solo remitente, nombre, asunto, cuerpo (recortado a 4.000 caracteres, sin el historial citado) y thread. Verifica que el email tenga formato válido antes de seguir. |

Además, antes de Slack hay un segundo Set (`Limpiar payload para Slack`) que deja solo trace_id, remitente, asunto y los primeros 300 caracteres del borrador, sin objetos binarios ni payloads grandes.

## Mínimo privilegio

- **Gmail:** el workflow usa únicamente el trigger de lectura y la operación *Create Draft*. n8n define los scopes de la credencial de Gmail OAuth2 y no se pueden reducir desde la interfaz; el control está en que el flujo no contiene operaciones de envío ni de borrado. El trigger además está acotado con el filtro `in:inbox subject:"Demo Valores"` y solo correos no leídos, para no procesar el resto de los mails.
- **HubSpot:** la clave tiene solamente los scopes `crm.objects.contacts.read` y `crm.objects.contacts.write`.
- **Slack:** bot con permiso de escritura en el canal de observabilidad.
- El correo del remitente no se guarda en la memoria: el `Session_ID` de los correos es `mail-` + id del hilo. El correo solo aparece en el CRM.

