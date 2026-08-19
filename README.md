# Chatbot Cafetería — Control de Horarios de Atención

Proyecto construido para **n8n + Telegram + Google Sheets**.

## Archivos incluidos

- `Control_Horarios_Cafeteria_n8n.json`: workflow importable en n8n.
- `Cafeteria_N8N_Google_Sheets.xlsx`: base de datos lista para subir y abrir como Google Sheets.
- `README.md`: instalación, configuración, lógica y pruebas.

- ## FOTOS PRUEBAS DEL Chatbot Cafetería — Control de Horarios de Atención
- <img width="832" height="790" alt="Captura de pantalla 2026-08-18 204521" src="https://github.com/user-attachments/assets/75cedd00-d22b-462c-84ea-bd6f0dcc58b3" />

<img width="860" height="856" alt="Captura de pantalla 2026-08-18 204531" src="https://github.com/user-attachments/assets/4454d426-17f1-4763-96f4-a3567a305d23" />



## Arquitectura

**Telegram → n8n → Google Sheets**

El bot funciona por **botones y mensajes de texto**:

- `INICIO` / `/start`
- `MENU` / botón **📋 Consultar Menú**
- `REALIZAR PEDIDO` / botón **🛒 Realizar Pedido**
- `PEDIDO: 1 Café Americano, 2 Empanadas`
- Botón **✅ Confirmar Pedido**
- Botón **❌ Cancelar**

Google Sheets actúa como:

- configuración central (`CONFIG`);
- catálogo dinámico (`MENU`);
- almacenamiento de pedidos (`PEDIDOS`);
- plan de validación (`PRUEBAS`);
- resumen operativo (`RESUMEN`).

---

# Instalación

## 1. Crear el Google Sheets

1. Sube `Cafeteria_N8N_Google_Sheets.xlsx` a Google Drive.
2. Ábrelo con **Google Sheets**.
3. Verifica que existan las pestañas:
   - `CONFIG`
   - `MENU`
   - `PEDIDOS`
   - `PRUEBAS`
   - `RESUMEN`
4. Copia el ID del documento desde la URL:

```text
https://docs.google.com/spreadsheets/d/ESTE_ES_EL_ID/edit
```

Copia únicamente `ESTE_ES_EL_ID`.

## 2. Importar el workflow en n8n

1. En n8n, crea un workflow nuevo.
2. Usa **Import from File**.
3. Selecciona `Control_Horarios_Cafeteria_n8n.json`.

## 3. Configurar el Spreadsheet ID

Abre el nodo:

```text
CONFIG_GLOBAL
```

Busca:

```javascript
const SPREADSHEET_ID = 'PEGA_AQUI_EL_ID_DE_GOOGLE_SHEETS';
```

Reemplaza únicamente el texto entre comillas por el ID real.

**No necesitas pegar el ID en todos los nodos.** Todos los nodos de Google Sheets toman el mismo ID desde `CONFIG_GLOBAL`.

## 4. Configurar credenciales

### Telegram

Selecciona tu credencial de Telegram en:

- `Telegram Trigger`
- todos los nodos `Telegram - ...`
- `Responder Callback`

### Google Sheets

Selecciona la misma credencial de Google Sheets en:

- `Leer CONFIG`
- `Leer MENU`
- `Google Sheets - Guardar Borrador`
- `Google Sheets - Confirmar Pedido`
- `Google Sheets - Cancelar Pedido`

La cuenta de Google conectada a n8n debe tener acceso de edición al documento.

## 5. Activar

1. Guarda el workflow.
2. Actívalo.
3. En Telegram escribe:

```text
/start
```

---

# Configuración desde Google Sheets

La pestaña `CONFIG` permite cambiar el comportamiento sin modificar la lógica principal.

| CLAVE | Valor inicial |
|---|---|
| `TIMEZONE` | `America/Bogota` |
| `DIAS_ATENCION` | `1,2,3,4,5` |
| `HORA_APERTURA` | `08:00` |
| `HORA_CIERRE` | `17:00` |
| `MONEDA` | `COP` |
| `MENSAJE_CERRADO` | Mensaje obligatorio del examen |

Para `DIAS_ATENCION` se usa el número de día de Luxon:

- 1 = Lunes
- 2 = Martes
- 3 = Miércoles
- 4 = Jueves
- 5 = Viernes
- 6 = Sábado
- 7 = Domingo

---

# Flujo del bot

## Consultar menú

Ruta:

```text
Telegram Trigger
→ CONFIG_GLOBAL
→ Leer CONFIG
→ Aplicar CONFIG
→ Router de Acciones
→ Leer MENU
→ Construir Menú
→ Telegram - Menú
```

Esta ruta **no entra al IF de horarios**, por lo tanto el menú puede consultarse 24/7.

## Realizar pedido

Ruta:

```text
Router de Acciones
→ IF Horario - Realizar Pedido ($now)
```

TRUE:

```text
→ Telegram - Instrucciones Pedido
```

FALSE:

```text
→ Telegram - Cafetería Cerrada
```

## Registrar borrador

El usuario escribe:

```text
PEDIDO: 1 Café Americano, 2 Empanadas
```

El pedido se guarda en `PEDIDOS` con:

```text
ESTADO = PENDIENTE
```

Después Telegram muestra:

- ✅ Confirmar Pedido
- ❌ Cancelar

## Confirmar pedido

La confirmación vuelve a validar el horario mediante:

```text
IF Horario - Confirmar Pedido ($now)
```

Esto evita el caso en que un usuario cree un borrador antes del cierre y trate de confirmarlo después de las 5:00 PM.

Si está abierto:

```text
PENDIENTE → CONFIRMADO
```

Si está cerrado:

```text
🌙 Cafetería Cerrada...
```

## Cancelar

Cancelar no crea un pedido nuevo. Actualiza el registro existente:

```text
PENDIENTE → CANCELADO
```

---

# Update: Examen — Control de Horarios de Atención

## Objetivo implementado

Se agregó una restricción de horario para impedir que los usuarios realicen o confirmen pedidos cuando la cafetería está cerrada.

## Horario configurado

```text
Lunes a Viernes
08:00 AM a 05:00 PM
Zona horaria: America/Bogota
```

## Lógica del nodo IF

El workflow usa `$now` en n8n y lo convierte a la zona horaria configurada.

El IF valida simultáneamente:

1. Que el día actual esté incluido en `DIAS_ATENCION`.
2. Que la hora actual sea mayor o igual a `HORA_APERTURA`.
3. Que la hora actual sea menor o igual a `HORA_CIERRE`.

Si las tres condiciones son verdaderas, el flujo continúa.

Si alguna es falsa, el flujo se detiene y se envía:

```text
🌙 Cafetería Cerrada. Nuestro horario es de Lunes a Viernes, 8am a 5pm. ¡Te esperamos mañana!
```

## Excepción solicitada

**Consultar menú:** permitido incluso fuera del horario.

**Realizar pedido:** bloqueado fuera del horario.

**Confirmar pedido:** bloqueado fuera del horario.

## Validación doble

Se implementaron tres IF de protección:

1. `IF Horario - Realizar Pedido ($now)`
2. `IF Horario - Registrar Borrador ($now)`
3. `IF Horario - Confirmar Pedido ($now)`

La validación de confirmación es importante porque el usuario podría haber iniciado un pedido antes de las 5:00 PM y tratar de confirmarlo después.

---

# Casos de prueba

La pestaña `PRUEBAS` del Google Sheets ya contiene casos para demostrar el examen.

Pruebas principales:

1. Martes 10:00 + Realizar Pedido → permitido.
2. Martes 23:00 + Realizar Pedido → bloqueado.
3. Sábado 10:00 + Realizar Pedido → bloqueado.
4. Domingo 12:00 + Consultar Menú → permitido.
5. Viernes 16:59 + Confirmar → permitido.
6. Viernes 17:01 + Confirmar → bloqueado.

---

# Personalización

## Cambiar productos

Edita las filas de `MENU`.

Si un producto no debe mostrarse:

```text
DISPONIBLE = NO
```

## Cambiar horario

Edita:

```text
CONFIG!HORA_APERTURA
CONFIG!HORA_CIERRE
CONFIG!DIAS_ATENCION
```

No necesitas modificar los nodos IF.

## Cambiar mensaje de cierre

Edita:

```text
CONFIG!MENSAJE_CERRADO
```

---

# Nota importante sobre la importación

Las credenciales **no se incluyen** en el JSON por seguridad. Después de importar el workflow debes seleccionar tus credenciales de Telegram y Google Sheets en n8n.

El workflow queda desactivado al importarlo. Actívalo únicamente después de conectar credenciales y pegar el Spreadsheet ID.
