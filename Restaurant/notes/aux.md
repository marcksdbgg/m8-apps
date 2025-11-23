Voy directo al grano: el flujo está muy completo, pero hay varias cositas de lógica de negocio y de “casos raros” que van a romper o a dejar todo en revisión manual sin que te des cuenta. Te lo separo en:

1. Errores / incoherencias claras en la lógica
2. Dónde **sí o sí** deberías escalar a un admin (y en qué nodo)
3. Casos de error que faltan cubrir

---

## 1. Errores / incoherencias claras

### 1.1. Mensajes a *staff/admin* que en realidad van al **cliente**

**Nodos afectados:**

* `Code - Resumen Staff` → `HTTP - Enviar Resumen Staff`
* `JS - Validación Cruzada` → `Set - Datos Error` → `HTTP - Msg Admin`

**Problema:**

En ambos casos usas:

```json
"to": $('Set - Datos Imagen Pago').item.json.telefono
```

Ese `telefono` viene de `Set - Datos Imagen Pago`:

```json
{
  "name": "telefono",
  "value": "={{ $json.messages[0].from }}"
}
```

Es decir: **el número del cliente**, no del admin/staff.

👉 Resultado:

* El “Resumen Staff” con el comprobante se lo mandas al cliente.
* El “Msg Admin” de error en conciliación también se lo mandas al cliente (además del “Msg Usuario”).

**Lo que esperabas hacer (por tu propio diseño):**

* Tienes almacenado un teléfono de pedidos/admin:

  * `Set - Datos Cliente` define `telefonoPedidos = "+51-987654321"` (no se usa en ninguna parte).
  * `JS - Validación Cruzada` define `admin_phone = '51987654321'` y lo propaga a `Set - Datos Error`.

**Recomendación (cambio concreto):**

* En `HTTP - Enviar Resumen Staff`:

  * Usa `to: $node["Set - Datos Cliente"].json.telefonoPedidos`
  * O bien `to: $json.admin_phone` si prefieres que el admin venga de la validación.

* En `HTTP - Msg Admin`:

  * Cambia el `to` a `to: $json.admin_phone` (lo recibes desde `Set - Datos Error`).
  * Si quieres enviar la **imagen del comprobante** al admin, está en `$('Set - Datos Imagen Pago').first().json.media_id`, pero ojo: el `to` debe ser el admin, no el cliente.

---

### 1.2. Flujo de conciliación de pagos: nunca se aprueba automáticamente

**Nodos clave:**

* `DB - Guardar Comprobante y Finalizar Pedido`
* `DB - Buscar Conciliación`
* `JS - Validación Cruzada`
* `IF - Pago Aprobado`
* `DB - Validar pedido completo y completar`

**Problema 1 – Datos del pedido incompletos en la validación:**

En `JS - Validación Cruzada` haces:

```js
const pedidoDb = $('DB - Guardar Comprobante y Finalizar Pedido').first().json || {};
...
pedido_id: pedidoDb.pedido_finalizado_id || 'N/A',
monto_pedido: parseFloat(pedidoDb.total || 0),
```

Pero el SQL de `DB - Guardar Comprobante y Finalizar Pedido` es:

```sql
RETURNING pedidos.id as pedido_encontrado_id;
```

No devuelve ni `total` ni `pedido_finalizado_id`.
→ Resultado:

* `pedido_id` siempre queda `'N/A'`.
* `monto_pedido` siempre queda `0`.

Por eso en `JS - Validación Cruzada` entras siempre en el bloque:

```js
if (!resultado.pedido_id || resultado.pedido_id === 'N/A') {
  resultado.motivo_tecnico = 'ERROR_SISTEMA_SIN_PEDIDO';
  ...
  return { json: resultado };
}
```

Y **jamás** llegas al bloque donde `aprobado = true`.
Así que:

* `IF - Pago Aprobado` **siempre** va al branch `false`.
* Nunca se ejecuta `DB - Validar pedido completo y completar`.
* Siempre quedan casos “para revisión manual”.

**Recomendación:**

Tienes dos opciones:

1. **Modificar `DB - Guardar Comprobante y Finalizar Pedido`** para que devuelva lo que necesitas:

   ```sql
   ... 
   RETURNING 
     pedidos.id as pedido_finalizado_id,
     pedidos.total;
   ```

   y actualizar el JS:

   ```js
   pedido_id: pedidoDb.pedido_finalizado_id,
   monto_pedido: parseFloat(pedidoDb.total || 0),
   ```

2. O bien, hacer un **nuevo SELECT** en `JS - Validación Cruzada` (o un nodo Postgres previo) que lea el pedido actual a partir del `telefono` y/o del `pedido_encontrado_id`.

---

**Problema 2 – Estado `MANUAL` de `pagos_recibidos` nunca se usa**

En `002_pagos_reconciliacion.sql` defines estados:

* `PENDIENTE`, `ASIGNADO`, `MANUAL`

Pero en el workflow:

* Solo cambias a `ASIGNADO` en `DB - Validar pedido completo y completar`.
* En los casos en que hay error o se requiere revisión:

  * `PAGO_NO_ENCONTRADO_EN_BANCO`
  * `MONTO_INSUFICIENTE`
  * `MONTO_EXCEDENTE`
  * `IMAGEN_NO_VALIDA`

  **no** cambias `pagos_recibidos.estado` a `MANUAL`.

**Recomendación:**

En `JS - Validación Cruzada`, cuando detectas que se necesita revisión manual, además de llenar `mensaje_admin`, deberías:

* Añadir el `pago_id` (aunque sea null) y un campo tipo `requiere_flag_manual = true`.
* Después de `Set - Datos Error`, añadir un nodo `Postgres`:

  * `UPDATE pagos_recibidos SET estado = 'MANUAL' WHERE id = $1;`

Así dejas claramente marcados los pagos dudosos.

---

### 1.3. Acción `ESPERAR_PAGO` del LLM no conectada

En el schema que le pasas al `Structured Output Parser` tienes:

```json
"accion": {
  "enum": [
    "NONE",
    "MOSTRAR_CARTA",
    "INFO_PEDIDO",
    "INFO_DELIVERY",
    "ORDEN_DESCONOCIDA",
    "ESPERAR_UBICACION",
    "ESPERAR_PAGO",
    "SOLICITAR_CONFIRMACION"
  ]
}
```

En el nodo `Switch - accion` sí definiste la regla para `ESPERAR_PAGO`, pero en las **conexiones** solo hay 5 salidas:

* 0 → `Set - Mensaje Recurrente`
* 1 → `Set - Solicitar Informacion`
* 2 → `Set - Obtener mensaje`
* 3 → `GH - Obtener Carta`
* 4 → `DB - Actualizar Estado a PENDIENTE_UBICACION`
* (no hay nada conectado para la rama `EsperarPago`)

👉 Si el LLM devuelve `ESPERAR_PAGO`, la ejecución se cortará ahí.

**Recomendación:**

Conectar la salida `EsperarPago` del `Switch - accion` a un nodo tipo:

* `Postgres - UPDATE pedidos SET estado = 'PENDIENTE_PAGO' ...` (si lo necesitas), o
* Simplemente a un `Set - Mensaje Esperar Pago` que diga algo como *"Solo falta que envíes tu comprobante..."*.

---

### 1.4. Acciones del LLM que no tienen implementación

* `INFO_DELIVERY` existe en el enum, pero en `Switch - accion` **no tiene regla**.
* `NONE` tampoco.

Eso significa que:

* Si por error de prompt el modelo devuelve `INFO_DELIVERY` o `NONE`, el flujo termina sin respuesta al usuario.

**Recomendación:**

Añadir un caso por defecto (por ejemplo, en un `If` o con una regla más) que:

* Mande un mensaje genérico tipo “No entendí tu mensaje, ¿quieres ver la carta o hacer un pedido?”.
* Opcionalmente, si ese fallback se dispara varias veces seguidas → escalar a admin (ver sección 2).

---

### 1.5. Botón CANCELAR borra **todos** los pedidos no completos

En `Eliminar Pedido`:

```sql
DELETE FROM pedidos
USING clientes
WHERE pedidos.cliente_id = clientes.id
  AND clientes.telefono = $1
  AND pedidos.estado <> 'COMPLETO';
```

Si por alguna razón en el futuro permites varios pedidos “abiertos” a la vez para el mismo teléfono, el botón `CANCELAR` se va a llevar todo lo que esté vivo.

Si para ti solo puede haber un pedido abierto por cliente, está bien; pero es bueno ser consciente.

---

### 1.6. Manejo de tipos de mensaje incompletos

En `Switch - Tipo de mensaje` solo enrutas:

* `text` → flujo principal
* `location` → `DB - Procesar Ubicación`
* `audio` → (no conectado)
* `interactive` → botones
* `image` → flujo de pago

**Problemas:**

* El branch `Audio` no tiene nada conectado → si alguien manda nota de voz, el bot se queda mudo.
* No hay un branch “default” para otros tipos (`document`, `video`, `sticker`…).

**Recomendación:**

* Conectar la salida `Audio` a un `Set - Mensaje No Audio` → `HTTP - Mensaje error`, indicando que no procesas audios.
* Añadir un branch “catch-all” (o un flujo de error) para cualquier tipo no contemplado: mensaje tipo *“Solo puedo procesar texto, botones, ubicación e imágenes de pago.”*

---

### 1.7. Regreso de usuario sin pedido activo

`DB - Obtener Pedido Activo`:

```sql
WHERE c.telefono = $1
  AND p.estado <> 'COMPLETO'
```

Si un cliente antiguo escribe pero **no tiene pedidos abiertos**, pueden pasar dos cosas según cómo se porte n8n:

* Si bien configurado, `JS - Preparar Contexto` lo trata como `order_state = 'NONE'` (tu código ya contempla un try/catch y valor por defecto).
* Pero si `DB - Obtener Pedido Activo` no emite ítems y no se propaga bien, todo el subflujo AI podría no ejecutarse y el usuario se queda sin respuesta.

A nivel de negocio, tiene sentido que:

* Usuario nuevo → `Mensaje Bienvenida`.
* Usuario recurrente sin pedido activo → `Mensaje Recurrente` o algo similar, con botones otra vez.

Revisa en runtime qué hace `alwaysOutputData` en ese nodo y, si no te garantiza siempre un item, añade un `If` a partir de `DB - Obtener Pedido Activo` que detecte “sin pedido” y mande directamente un mensaje recurrente.

---

## 2. ¿Dónde deberías escalar a un admin?

Te marco nodos concretos donde **sí o sí** tendría sentido disparar un flujo de “hablar con administrador” (en WhatsApp o interno):

### 2.1. Conciliación de pago (ya casi lo tienes)

En `JS - Validación Cruzada`, ya etiquetas muy bien los casos:

* `ERROR_SISTEMA_SIN_PEDIDO`
* `IMAGEN_NO_VALIDA`
* `PAGO_NO_ENCONTRADO_EN_BANCO`
* `MONTO_INSUFICIENTE`
* `MONTO_EXCEDENTE`

Y construyes `mensaje_admin` + `admin_phone`.

**Qué falta:**

* Asegurarte de que `HTTP - Msg Admin` realmente usa `admin_phone`.
* Opcional: marcar `pagos_recibidos.estado = 'MANUAL'` (nuevo nodo Postgres después de `Set - Datos Error`).
* Opcional: actualizar también `pedidos.estado = 'PENDIENTE_REVISION_MANUAL'` si quieres diferenciarlo de `PENDIENTE_PAGO`.

👉 Aquí es donde tu mensaje al admin es **obligatorio**, porque afecta plata.

---

### 2.2. Imagen de pago sin pedido pendiente

`If - Pedido Pagado Encontrado`:

* **true** → sigue flujo normal.
* **false** → `Set - Mensaje Imagen Inesperada` → `HTTP - Mensaje error` al cliente.

Aquí puede haber un error grave de estado (ej. el pedido ya fue completado por error, o es de otro día).

**Sugerencia:**

* Después de `Set - Mensaje Imagen Inesperada`, añade en paralelo un `Set / HTTP - Msg Admin Imagen Inesperada` que use el mismo `admin_phone` y media_id.
* Mensaje para admin tipo: “Cliente X envió comprobante pero no hay pedido PENDIENTE_PAGO”.

---

### 2.3. Ubicación inesperada

`DB - Procesar Ubicación` → `If - Pedido Encontrado`:

* **true** → actualiza a `PENDIENTE_PAGO` y manda instrucciones de pago.
* **false** → `Set - Mensaje Ubicacion Inesperada` → mensaje al cliente.

Puede ser simplemente que el cliente se equivocó, pero si esto pasa **después de haber pedido dirección**, es sospechoso de estado roto.

**Sugerencia mínima:**

* Igual que antes: de `Set - Mensaje Ubicacion Inesperada` saca un segundo branch hacia un `HTTP - Msg Admin Ubicacion`, con texto del estilo “Ubicación sin pedido PENDIENTE_UBICACION”.

---

### 2.4. Errores repetidos del LLM

`Switch - accion` y `If - Hay Errores`:

* Ahora mismo, si el LLM devuelve siempre `ORDEN_DESCONOCIDA` o llena `errores` una y otra vez, solo hablas con el usuario.

Como patrón de negocio:

* Si en un mismo pedido tienes **N intentos fallidos** (por ejemplo 3) de interpretación de pedido, tiene sentido pasar a humano.

Implementación sencilla:

* Añadir campo `intentos_llm` en `pedidos.metadatos` o en `estado`.
* Incrementarlo cada vez que `If - Hay Errores` sea `true`.
* Si `intentos_llm >= 3` → en lugar de seguir insistiendo, envías mensaje tipo “Te voy a pasar con un asesor” y disparas un `HTTP - Msg Admin Conversación` con transcripción.

---

## 3. Casos de error que faltan cubrir

Te dejo un checklist de gaps que veo:

1. **Timeouts de estados intermedios**

   * Pedidos en `PENDIENTE_DATOS_DELIVERY`, `PENDIENTE_UBICACION`, `PENDIENTE_PAGO` pueden quedarse ahí para siempre.
   * No es de este workflow en sí, pero recomendaría un cron que marque:

     * `PENDIENTE_*` viejo → `CANCELADO_POR_INACTIVIDAD`
     * y opcionalmente notifique al admin.

2. **Manejo de `INFO_DELIVERY` y `NONE`**

   * Como comenté arriba, faltan ramas en `Switch - accion`.
   * `INFO_DELIVERY` debería mandar info de cobertura, costo, zonas, etc.
   * `NONE` debería ir a un fallback amable.

3. **Fallback de tipo de mensaje desconocido**

   * Ningún manejo para `video`, `document`, etc.
   * Recomendado un mensaje genérico de error y quizá logging.

4. **Pedido sin items pero sin errores**

   * En `Validar y armar pedido final`, si el LLM devuelve `pedido_parseado` vacío pero `errores` también vacío, podrías dejar un pedido con `total = 0` y sin detalle.
   * Yo añadiría:

     * si `itemsValidados.length === 0` → forzar un error tipo “No se pudo reconocer ningún producto” y no actualizar `pedido_items`.

5. **Conciliación con OCR dudoso**

   * Ahora mismo, si el OCR devuelve `monto = 0` pero el banco sí tiene un pago, no lo vas a encontrar por el `WHERE monto = $1`.
   * Podrías añadir una segunda búsqueda:

     * Si no encuentras nada:

       * Reintentar `DB - Buscar Conciliación` sólo por `codigo_operacion` o `numero_operacion` (si existen) sin usar `monto`.
       * O marcar ese caso directamente como `MANUAL`.

6. **Confirmación de pedido sin pedido activo**

   * Botón `BTN_CONFIRMAR_PEDIDO` → `DB - Obtener Pedido Activo1`.
   * Si por algún bug no hay pedido (0 filas), el flujo continúa pero `Switch - Tipo Pedido` no encontrará estado válido.
   * Añadir un `If` después de `DB - Obtener Pedido Activo1` que verifique que `id` no está vacío y, si lo está, mande un mensaje de error + opcionalmente notifique al admin.

---

### Resumen corto de lo más importante

* Corrige los **destinatarios** de todos los nodos “Staff/Admin”: ahora mismo todo le llega al cliente.
* Arregla la obtención de `pedido_id` y `total` en `JS - Validación Cruzada` para que la conciliación automática pueda funcionar.
* Conecta la rama `ESPERAR_PAGO` del `Switch - accion`.
* Añade fallbacks para `INFO_DELIVERY`/`NONE`, audio y tipos de mensaje desconocidos.
* Marca explícitamente los pagos “raros” como `MANUAL` y notifica al admin en:

  * Conciliación fallida
  * Imagen de pago sin pedido
  * Ubicación sin pedido
  * Errores LLM repetidos

Si quieres, en un siguiente mensaje puedo proponerte exactamente cómo se verían uno o dos nodos nuevos (por ejemplo, el flujo completo de “escalar a admin” estándar) para que lo copies/pegues en n8n.
