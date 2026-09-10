# UNIVERSIDAD PRIVADA DE TACNA

**Facultad de Ingeniería**<br>
**Escuela Profesional de Ingeniería de Sistemas**

## DINÁMICA DE AULA

**Soluciones Móviles II · SI-988**

**Nombre de la dinámica:** El pedido que se creó dos veces<br>
**Semana:** 04 · **Unidad:** I · **Modalidad:** Individual

**Estudiante:**

- COLQUE PONCE, Sergio · 2022073503

**Docente:** Dr. Oscar Juan Jimenez Flores<br>
**Tacna, Perú · 2026**

# 1. La consigna que resolvimos

Diseñamos el contrato del endpoint principal de **Alerta Ciudadana**, una aplicación orientada a municipalidades que permite a un ciudadano comunicar una situación de riesgo con su tipo, descripción, ubicación y evidencia. La alerta es evaluada por Seguridad Ciudadana para establecer su prioridad y asignar rápidamente al personal correspondiente.

# 2. Lo que resolvimos

## 2.1 Contrato del endpoint

| Elemento | Contenido |
|---|---|
| **El endpoint de escritura** | `POST /alertas` · éxito **`201 Created`** · acompañado de `Location: /alertas/ALT-1048`. Se usa `POST` porque el ciudadano solicita crear una alerta nueva dentro de la colección y el servidor asigna su identificador. `201` confirma que el recurso fue creado y `Location` permite consultarlo. |
| **Idempotencia** | `POST` no es idempotente por naturaleza. Antes del primer intento, la aplicación genera una clave UUID y la envía en `Idempotency-Key`. Si la conexión falla, todos los reintentos conservan la misma clave y el mismo contenido. El servidor guarda la clave asociada al ciudadano y al resultado; si la recibe nuevamente, devuelve el mismo `201`, la misma cabecera `Location` y la misma alerta, sin crear un duplicado. |
| **El `GET` que lo acompaña** | `GET /ciudadanos/me/alertas?cursor=<ultimo_id>&limit=20`. Se usa paginación por cursor porque pueden aparecer alertas nuevas mientras el ciudadano consulta la lista; así se evitan elementos repetidos u omitidos. La respuesta incluye `Cache-Control: private, max-age=15, must-revalidate` y `ETag`: es información personal que no debe almacenarse en cachés compartidas y sus estados cambian rápido, por eso solo se reutiliza durante 15 segundos y luego se valida antes de mostrarla otra vez. |

## 2.2 Tratamiento de errores

| Código | `Failure` de dominio | Mensaje al usuario | ¿Se reintenta? |
|---|---|---|---|
| `401 Unauthorized` | `NoAutorizado` | «Tu sesión venció. Ingresa nuevamente para enviar la alerta.» | No inmediatamente. Se intenta renovar la sesión una vez; después se vuelve a enviar la operación con la misma clave. Si no se puede renovar, se solicita iniciar sesión. |
| `422 Unprocessable Content` | `ErrorValidacion` | «No pudimos identificar el lugar de la alerta. Activa tu ubicación o márcalo en el mapa.» | No. El ciudadano debe proporcionar o corregir la ubicación antes de volver a enviarla. |
| `503 Service Unavailable` | `ErrorServidor` | «Tu alerta aún no fue enviada. Lo intentaremos nuevamente en un momento.» | Sí. Se realizan hasta tres intentos con espera progresiva, conservando siempre la misma clave para no duplicar la alerta. |

# 3. Cómo lo decidimos

Elegimos **alerta ciudadana** en lugar de *reporte* porque la aplicación atiende situaciones de seguridad que requieren evaluación rápida. *Incidencia* queda como término interno para el acontecimiento que analiza la municipalidad. El alcance inicial permite enviar la alerta, revisarla, asignar personal y consultar su estado; se descartó el despacho completamente automático porque requiere reglas operativas y coordinación institucional que todavía no han sido definidas.

Usamos `POST /alertas` y no `PUT` porque el ciudadano no conoce ni elige la URI definitiva del recurso. El servidor crea el identificador y responde `201 Created` con `Location`. Se descartó responder siempre `200 OK` porque no expresa de forma precisa que se creó un recurso nuevo.

La clave de idempotencia se crea antes del primer envío. Generar una clave diferente en cada reintento permitiría que una pérdida de conexión produjera varias alertas iguales y que Seguridad Ciudadana movilizara recursos innecesariamente. Por eso el servidor conserva el primer resultado y lo devuelve cuando reconoce la misma clave.

Escogimos paginación por cursor en lugar de páginas numeradas porque la lista puede cambiar mientras se consulta. Para la caché usamos un tiempo corto de 15 segundos: reduce solicitudes repetidas y consumo de datos, pero evita mostrar durante mucho tiempo un estado desactualizado. `private` impide que la respuesta destinada a un ciudadano se guarde en una caché compartida y `ETag` permite validar cambios sin descargar nuevamente toda la lista.

Los mensajes de error indican qué ocurrió y qué debe hacer la persona sin mostrar códigos ni términos técnicos. Solo se reintentan fallos temporales; los datos incorrectos requieren intervención del ciudadano. Todo reintento del envío conserva la clave original.

# 4. Lo que aprendimos

Una respuesta perdida no significa necesariamente que la alerta no haya sido registrada: el servidor pudo crearla aunque la aplicación no recibiera la confirmación. Comprendimos que la idempotencia permite repetir el envío de manera segura cuando se conserva la misma clave. También aprendimos que los códigos HTTP, la paginación y la caché forman parte del contrato y afectan tanto la experiencia del ciudadano como el uso de datos y batería.

# 5. Fuentes consultadas

- IETF. **RFC 9110: HTTP Semantics**, secciones 9.3.3 (`POST`), 10.2.2 (`Location`) y 15.3.2 (`201 Created`). <https://www.rfc-editor.org/rfc/rfc9110.html>
- IETF. **RFC 9111: HTTP Caching**, secciones 5.2.2.1 (`max-age`), 5.2.2.2 (`must-revalidate`) y 5.2.2.7 (`private`). <https://www.rfc-editor.org/rfc/rfc9111.html>
- IETF HTTPAPI Working Group. **The Idempotency-Key HTTP Header Field**, Internet-Draft `draft-ietf-httpapi-idempotency-key-header-07`. <https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/>
