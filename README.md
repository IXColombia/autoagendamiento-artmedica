# API de Gestión de Citas — Integración Agente IA (ArtMedica)

Documentación **externa** para proveedores de agentes conversacionales (IA, WhatsApp, telefonía) que se integran con **ArtMedica** (`COMPANIA = "AR"`) para consultar pacientes/citas y ejecutar asignación, reprogramación y cancelación.

> **Importante:** Las credenciales de acceso, códigos de catálogo (programas, especialidades, EPS) y reglas de negocio específicas serán entregadas por **ArtMedica / equipo de soporte Qrystalos** durante el onboarding. No utilice endpoints, métodos o parámetros que no figuren en este documento sin autorización previa.

---

## Índice

1. [Base URL](#base-url)
2. [Autenticación](#autenticación)
3. [Convenciones de la API](#convenciones-de-la-api)
4. [Flujo recomendado](#flujo-recomendado)
5. [Catálogo de operaciones](#catálogo-de-operaciones)
6. [Parámetros de configuración](#parámetros-de-configuración)
7. [Operaciones detalladas](#operaciones-detalladas)
8. [Referencia rápida](#referencia-rápida)
9. [Diagrama — demanda inducida](#diagrama--demanda-inducida)
10. [Colección Postman](#colección-postman)

---

## Base URL

```
https://reportes-artmedica.qrystalos.com/api/
```

En autenticación, el campo `COMPANIA` siempre debe enviarse como `"AR"`.

---

## Autenticación

### Obtener token (login integrador)

```
POST https://reportes-artmedica.qrystalos.com/api/ususu/ingresar
Content-Type: application/json
```

**Request body** (credenciales entregadas por soporte):

```json
{
  "COMPANIA": "AR",
  "USUARIO": "<usuario_integrador>",
  "CLAVE": "<clave>"
}
```

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `COMPANIA` | string | ✅ | Siempre `"AR"` para ArtMedica |
| `USUARIO` | string | ✅ | Usuario técnico del integrador |
| `CLAVE` | string | ✅ | Contraseña del usuario |

### Respuestas del login

#### Éxito — sesión iniciada (HTTP `200`)

Cuando las credenciales son válidas y no hay pasos de seguridad web pendientes, la API responde:

```json
{
  "res": "ok",
  "jwt": "<token_jwt>",
  "result": {
    "USUARIO": "<usuario_integrador>",
    "NOMBRE": "Integrador IA",
    "COMPANIA": "AR",
    "IDSEDE": "SED01"
  }
}
```

| Campo | Descripción |
|-------|-------------|
| `jwt` | Token Bearer para las operaciones siguientes. **Guardarlo.** |
| `result` | Datos del usuario autenticado. Puede incluir campos adicionales según la instalación. |

#### Credenciales inválidas (HTTP `200`)

```json
{
  "res": "ko",
  "code": "INVALID_CREDENTIALS",
  "message": "Usuario o contraseña incorrectos. Si acaba de cambiar la clave en el login, use la contraseña nueva."
}
```

También puede aparecer `code: "LOGIN_FAILED"` con mensaje genérico.

#### Usuario bloqueado (HTTP `429`)

```json
{
  "res": "ko",
  "code": "USER_BLOCKED",
  "message": "Usuario bloqueado temporalmente hasta las ..."
}
```

#### Compañía no encontrada (HTTP `200`)

```json
{
  "res": "ko",
  "code": "COMPANY_NOT_FOUND",
  "message": "Compañía no encontrada."
}
```

#### Acceso denegado por IP (HTTP `403`)

```json
{
  "res": "ko",
  "code": "WHITELIST_DENIED",
  "message": "..."
}
```

#### Seguridad web pendiente (HTTP `200`) — no aplica a integradores API

Si la respuesta incluye `status: "PENDING_SECURITY"` y `challengeToken` en lugar de `jwt`, el usuario tiene configurados pasos adicionales (MFA, cambio de clave, etc.) propios del portal web.

```json
{
  "res": "ok",
  "status": "PENDING_SECURITY",
  "challengeToken": "...",
  "result": {
    "USUARIO": "...",
    "NOMBRE": "...",
    "EMAIL_MASKED": "..."
  },
  "requirements": { }
}
```

Los usuarios de integración **deben** recibir `jwt` directamente. Si obtiene `PENDING_SECURITY`, contacte a soporte ArtMedica / Qrystalos para ajustar el usuario API.

#### Error interno (HTTP `500`)

```json
{
  "res": "ko",
  "code": "LOGIN_ERROR",
  "message": "..."
}
```

### Resumen login

| HTTP | `res` | Indicador | Acción |
|------|-------|-----------|--------|
| **200** | `"ok"` | Incluye `jwt` | Guardar token; continuar |
| **200** | `"ok"` | `status: "PENDING_SECURITY"` | Escalar a soporte (usuario mal configurado) |
| **200** | `"ko"` | `code` + `message` | Corregir credenciales o `COMPANIA` |
| **403** | `"ko"` | `WHITELIST_DENIED` | Verificar IP permitida con soporte |
| **429** | `"ko"` | `USER_BLOCKED` | Esperar desbloqueo |
| **500** | `"ko"` | Error interno | Reintentar; escalar a soporte |

### Uso del token en operaciones

Incluir en **todas** las llamadas de negocio:

```
Authorization: Bearer <jwt>
```

El campo `jwt` se renueva en cada respuesta exitosa de `POST /api/json`. Reemplazar siempre el token almacenado. Si recibe `logout: true`, cerrar sesión y autenticarse de nuevo.

---

## Convenciones de la API

### Invocar una operación

Todas las operaciones de gestión de citas usan el mismo endpoint:

```
POST https://reportes-artmedica.qrystalos.com/api/json
Content-Type: application/json
Authorization: Bearer <jwt>
```

**Cuerpo estándar:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "VALIDAR_PACIENTE",
  "PARAMETROS": { }
}
```

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `MODELO` | string | ✅ | Siempre `"CIT_COL_4"` para esta integración |
| `METODO` | string | ✅ | Nombre de la operación (ver catálogo) |
| `PARAMETROS` | object | ❌ | Datos específicos de la operación |

### Respuesta exitosa de transporte — HTTP `200`

```json
{
  "res": "ok",
  "jwt": "<jwt_renovado>",
  "result": {
    "recordset": [{ "OK": "OK" }],
    "recordsets": [
      [{ "OK": "OK" }],
      [ "... datos ..." ]
    ]
  }
}
```

| Campo | Descripción |
|-------|-------------|
| `res` | `"ok"` indica que la petición fue procesada. **No garantiza** éxito de negocio. |
| `jwt` | Token renovado. Actualizar el almacenado. |
| `result.recordset` | Primer bloque de datos (estado de la operación). |
| `result.recordsets[n]` | Bloques adicionales en orden (`[0]` suele coincidir con `recordset`). |

### Resultado de negocio — campo `OK`

| Valor en `recordset[0].OK` | Significado | Acción |
|----------------------------|-------------|--------|
| `"OK"` | Operación exitosa | Leer `recordsets[1]` y siguientes |
| `"KO"` | Validación o regla de negocio | Leer `recordsets[1].ERROR` (puede haber varias filas) |

> **Regla:** `HTTP 200` + `res: "ok"` + `recordset[0].OK === "KO"` es un **error de negocio**, no un fallo de red. Corrija datos o informe al paciente; no reintente en bucle.

Algunas operaciones devuelven `OK = "OK"` pero indican condición negativa en **campos** (ej. `GESTIONAR = false`, `CUPO_DISPONIBLE = false`). Revisar siempre los flags del primer bloque.

### Errores HTTP

| HTTP | `res` | Cuándo ocurre | Acción |
|------|-------|---------------|--------|
| **200** | `"ok"` | Petición procesada | Revisar `recordset[0].OK` |
| **200** | `"ko"` | Sesión expirada o login fallido | Reautenticar |
| **400** | `"ko"` | Payload inválido | Corregir `MODELO`, `METODO` o parámetros |
| **401** | — | Token ausente o inválido | Obtener nuevo token |
| **500** | `"ko"` | Error interno | Log + soporte; no asumir si la operación se completó |

### Estructura habitual de `recordsets`

| Índice | Contenido |
|--------|-----------|
| `[0]` / `recordset` | Estado: `{ OK, ...flags }` |
| `[1]` | Datos principales o mensajes `ERROR` |
| `[2+]` | Detalle adicional (paciente, programas, citas, etc.) |

---

## Flujo recomendado

Secuencia sugerida para gestión de citas (demanda inducida NEPS):

```
[0] POST /ususu/ingresar                         → Obtener JWT
[1] VALIDAR_PACIENTE                             → Paciente activo y en programa
[2] CONSULTAR_CITAS_PERIODO                      → ¿Tiene cita en el periodo?
[3] EVALUAR_ELEGIBILIDAD_GESTION                 → Acción sugerida (reuma / MG / apoyo)
[4] CONSULTAR_SERVICIOS_ESPECIALIDAD             → Servicio a agendar
[5] CONSULTAR_DISPONIBILIDAD                     → Ofrecer 2–3 cupos (TOP_N)
[6] VERIFICAR_CUPO                               → Cupo aún libre
[7] VALIDAR_DISPONIBILIDAD                       → Reservar cupo (inmediatamente antes de asignar)
[8] ASIGNAR_CITA                                 → Confirmar cita
    └─ Si falla por concurrencia: LIBERAR_CUPO → volver a [5]
[9] Otras gestiones del paciente:
    ├─ CONFIRMAR_CITAS_PENDIENTES
    ├─ REPROGRAMAR_CITA (+ CONSULTAR_CAUSALES TIPO=CITREPRO)
    └─ CANCELAR_CITA (+ CONSULTAR_CAUSALES TIPO=CITCAN)
```

---

## Catálogo de operaciones

Todas con `MODELO: "CIT_COL_4"`.

| METODO | Tipo | Descripción breve |
|--------|------|-------------------|
| `VALIDAR_PACIENTE` | Consulta | Valida paciente, EPS y programa |
| `ACTUALIZAR_DATOS_PACIENTE` | Ejecución | Actualiza contacto y datos administrativos |
| `CONSULTAR_CITAS` | Consulta | Lista citas con filtros |
| `CONSULTAR_CITAS_PERIODO` | Consulta | ¿Tiene cita en un rango de fechas? |
| `CONSULTAR_ATENCION_ESPECIALIDAD` | Consulta | Historial y cita futura por especialidad |
| `EVALUAR_ELEGIBILIDAD_GESTION` | Consulta | Recomienda acción (no asigna cita) |
| `CONSULTAR_DISPONIBILIDAD` | Consulta | Cupos disponibles |
| `VERIFICAR_CUPO` | Consulta | Verifica disponibilidad antes de asignar |
| `VALIDAR_DISPONIBILIDAD` | Ejecución | Reserva temporal del cupo |
| `LIBERAR_CUPO` | Ejecución | Libera reserva temporal |
| `CONSULTAR_CAUSALES` | Consulta | Causales de cancelación o reprogramación |
| `CONSULTAR_SERVICIOS_ESPECIALIDAD` | Consulta | Servicios por especialidad y plan |
| `CONFIRMAR_CITAS_PENDIENTES` | Consulta | Citas pendientes de confirmación |
| `ASIGNAR_CITA` | Ejecución | Asigna cita al paciente |
| `REPROGRAMAR_CITA` | Ejecución | Reprograma cita existente |
| `CANCELAR_CITA` | Ejecución | Cancela cita vigente |

---

## Parámetros de configuración

ArtMedica entregará al integrador los códigos reales durante la implementación. Envíelos según indique cada operación:

| Parámetro | Uso |
|-----------|-----|
| `IDPESPECIAL` | Programa / modelo del paciente |
| `IDTERCERO` / `IDTERCEROCA` | EPS / población |
| `IDEMEDICA_REUMA` | Especialidad reumatología |
| `IDEMEDICA_MG` | Especialidad medicina general |
| `IDEMEDICAS_APOYO` | Lista separada por comas (Psicología, Trabajo Social, etc.) |

---

## Operaciones detalladas

### 1 — `VALIDAR_PACIENTE`

Validaciones iniciales: paciente existe, pertenece a la EPS indicada y está activo en el programa.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "VALIDAR_PACIENTE",
  "PARAMETROS": {
    "TIPODOC": "CC",
    "DOCIDAFILIADO": "1234567890",
    "IDPESPECIAL": "NEPS01",
    "IDTERCERO": "900298928"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `TIPODOC` + `DOCIDAFILIADO` | ⚠️ | Identificación (alternativa: `IDAFILIADO`) |
| `IDAFILIADO` | ⚠️ | Si ya se conoce, omitir documento |
| `IDPESPECIAL` | ❌ | Programa; valida activo en programa si se envía |
| `IDTERCERO` / `IDTERCEROCA` | ❌ | Filtra población EPS |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):**

| Recordset | Contenido |
|-----------|-----------|
| `[0]` | `{ OK, GESTIONAR, PACIENTE_ACTIVO, PROGRAMA_ACTIVO }` |
| `[1]` | Datos del paciente: `IDAFILIADO`, `NOMBREAFI`, `CELULAR`, `EMAIL`, `IDPLAN`, … |
| `[2]` | Programas activos del paciente |

**Errores de negocio (`OK = "KO"`):**

| `recordsets[1].ERROR` | Acción del agente |
|------------------------|-------------------|
| `Paciente no encontrado` | Verificar tipo y número de documento |
| `Paciente no pertenece a la poblacion de la aseguradora indicada` | No gestionar bajo esa EPS |
| `Paciente inactivo en el sistema. Debe evaluar derechos en la aseguradora` | No gestionar; orientar al paciente |
| `Paciente no activo en el programa/modelo indicado` | Fuera del alcance de gestión |

---

### 2 — `ACTUALIZAR_DATOS_PACIENTE`

Actualiza contacto y datos administrativos. Solo modifica campos enviados con valor.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "ACTUALIZAR_DATOS_PACIENTE",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "CELULAR": "3001234567",
    "TELEFONORES": "6017001234",
    "EMAIL": "paciente@correo.com",
    "IDADMINISTRADORA": "TER001",
    "IDPLAN": "PLN01",
    "TIPOUSUARIO": "C",
    "NIVELSOCIOEC": "3"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `IDAFILIADO` | ✅ | Identificador del paciente |
| `CELULAR`, `TELEFONORES`, `EMAIL` | ❌ | Datos de contacto |
| `IDADMINISTRADORA`, `IDPLAN`, `TIPOUSUARIO`, `NIVELSOCIOEC` | ❌ | Datos administrativos |

**HTTP:** `200`, `res: "ok"`. Sin token → `401`. Payload inválido → `400`.

**Respuesta exitosa (`OK = "OK"`):**

| Recordset | Contenido |
|-----------|-----------|
| `[0]` | `{ OK: "OK" }` |
| `[1]` | Datos actualizados del paciente |

**Errores de negocio (`OK = "KO"`):**

| `recordsets[1].ERROR` | Acción del agente |
|------------------------|-------------------|
| `IDAFILIADO invalido` | Ejecutar `VALIDAR_PACIENTE` antes o corregir ID |

---

### 3 — `CONSULTAR_CITAS`

Listado de citas con filtros opcionales.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "CONSULTAR_CITAS",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "FECHA_INI": "2024-06-01",
    "FECHA_FIN": "2024-06-30",
    "IDEMEDICA": "0012",
    "SOLO_PENDIENTES": true
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `IDAFILIADO` o `CONSECUTIVO` | ⚠️ | Al menos uno obligatorio |
| `FECHA_INI`, `FECHA_FIN` | ❌ | Rango de fechas |
| `IDEMEDICA`, `IDSEDE`, `MODALIDAD`, `CUMPLIDA` | ❌ | Filtros adicionales |
| `SOLO_PENDIENTES` | ❌ | `true` → solo citas pendientes |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):**

| Recordset | Contenido |
|-----------|-----------|
| `[0]` | `{ OK: "OK" }` |
| `[1]` | Lista de citas: `CONSECUTIVO`, `FECHA`, `ESPECIALIDAD`, `NOMBREMED`, `ESTADO_CITA`, `MODALIDAD`, … |

**Errores de negocio (`OK = "KO"`):**

| `recordsets[1].ERROR` | Acción del agente |
|------------------------|-------------------|
| `Indique IDAFILIADO o CONSECUTIVO` | Enviar identificador del paciente o de la cita |

---

### 4 — `CONSULTAR_CITAS_PERIODO`

Evalúa si el paciente tiene cita programada en un rango (ej. mes de gestión).

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "CONSULTAR_CITAS_PERIODO",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "FECHA_INI": "2024-06-01",
    "FECHA_FIN": "2024-06-30",
    "IDEMEDICA": "0012"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `IDAFILIADO` | ✅ | Paciente |
| `FECHA_INI`, `FECHA_FIN` | ✅ | Ventana de evaluación |
| `IDEMEDICA` | ❌ | Filtrar por especialidad |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):**

```json
// recordsets[0][0]
{ "OK": "OK", "TIENE_CITA_EN_PERIODO": true, "CANTIDAD": 2 }
```

| Recordset | Contenido |
|-----------|-----------|
| `[1]` | Detalle de citas en el periodo |

**Errores de negocio (`OK = "KO"`):**

| `recordsets[1].ERROR` | Acción del agente |
|------------------------|-------------------|
| `Parametros requeridos: IDAFILIADO, FECHA_INI, FECHA_FIN` | Completar parámetros obligatorios |

> En **demanda inducida**, si `TIENE_CITA_EN_PERIODO = true`, no continuar con asignación.

---

### 5 — `CONSULTAR_ATENCION_ESPECIALIDAD`

Historial de atención y cita futura por especialidad (ventanas típicas: 6 meses reumatología, 2 meses medicina general).

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "CONSULTAR_ATENCION_ESPECIALIDAD",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "IDEMEDICA": "0012",
    "MESES": 6
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `IDAFILIADO` | ✅ | Paciente |
| `IDEMEDICA` o `ESPECIALIDAD` | ✅ | Especialidad |
| `MESES` | ❌ | Ventana hacia atrás (default `6`) |
| `FECHA_INI`, `FECHA_FIN` | ❌ | Acotar cita futura |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`, bloque `[0]`):**

| Campo | Descripción |
|-------|-------------|
| `ATENDIDA_EN_PERIODO` | Atención cumplida en la ventana |
| `TIENE_CITA_FUTURA` | Cita pendiente futura en la especialidad |
| `ULTIMA_ATENCION` | Fecha última atención |
| `CONSECUTIVO_FUTURA` / `FECHA_FUTURA` | Próxima cita programada |
| `DIAS_DESDE_ULTIMA` | Días desde última atención |

**Errores de negocio (`OK = "KO"`):**

| `recordsets[1].ERROR` | Acción del agente |
|------------------------|-------------------|
| `Parametros requeridos: IDAFILIADO, IDEMEDICA` | Enviar paciente y especialidad |

---

### 6 — `EVALUAR_ELEGIBILIDAD_GESTION`

Motor de decisión: **no asigna cita**; recomienda la acción siguiente.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "EVALUAR_ELEGIBILIDAD_GESTION",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "FECHA_INI": "2024-06-01",
    "FECHA_FIN": "2024-06-30",
    "IDPESPECIAL": "NEPS01",
    "IDEMEDICA_REUMA": "0012",
    "IDEMEDICA_MG": "0001",
    "IDEMEDICAS_APOYO": "0030,0031,0032",
    "TIPO_FLUJO": "DEMANDA_INDUCEIDA"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `IDAFILIADO` | ✅ | Paciente |
| `FECHA_INI`, `FECHA_FIN` | ⚠️ | Requeridos para evaluar cita en periodo |
| `IDPESPECIAL`, `IDEMEDICA_*` | ⚠️ | Códigos entregados por ArtMedica |
| `TIPO_FLUJO` | ❌ | `DEMANDA_INDUCEIDA` (default) o `SOLICITUD_PACIENTE` |
| `MESES_REUMA` / `MESES_MG` | ❌ | Default `6` / `2` |

| `TIPO_FLUJO` | Comportamiento |
|--------------|----------------|
| `DEMANDA_INDUCEIDA` | Si hay cita en periodo → `NO_GESTIONAR` |
| `SOLICITUD_PACIENTE` | Sugiere `OFERTAR_REUMATOLOGO` / `OFERTAR_MEDICINA_GENERAL` |

**HTTP:** `200`, `res: "ok"`.

**Respuesta (`OK = "OK"`; la decisión está en flags):**

| Campo (`recordsets[0]`) | Valores ejemplo |
|------------------------|-----------------|
| `GESTIONAR` | `true` / `false` |
| `MOTIVO_NO_GESTION` | Texto cuando no aplica gestión |
| `ACCION_SUGERIDA` | `ASIGNAR_REUMATOLOGO`, `ASIGNAR_MEDICINA_GENERAL`, `ASIGNAR_PROFESIONAL_APOYO`, `NO_GESTIONAR`, `OFERTAR_*` |
| `IDEMEDICA_OBJETIVO` | Especialidad para consultar disponibilidad |
| `MODALIDAD_OBJETIVO` | `Presencial` o `Virtual` |
| `REUMA_ATENDIDA_MESES` / `MG_ATENDIDA_MESES` | Historial reciente |
| `TIENE_CITA_EN_PERIODO` | Cita en la ventana evaluada |

**Errores:** No devuelve `OK = "KO"`. Tratar `GESTIONAR = false` como resultado válido de negocio.

---

### 7 — `CONSULTAR_DISPONIBILIDAD`

Consulta cupos libres en un rango de fechas para ofrecer opciones al paciente.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "CONSULTAR_DISPONIBILIDAD",
  "PARAMETROS": {
    "FECHA_INI": "2024-06-01",
    "FECHA_FIN": "2024-06-15",
    "TOP_N": 3,
    "IDEMEDICA": "0012",
    "IDSEDE": "SED01",
    "IDMEDICO": "DOC001",
    "MODALIDAD": "Presencial",
    "IDAFILIADO": "AFI001"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `FECHA_INI`, `FECHA_FIN` | ⚠️ | Rango de búsqueda (máximo **62 días**) |
| `TOP_N` | ❌ | Cantidad de cupos a retornar (default `3`) |
| `IDEMEDICA` o `ESPECIALIDAD` | ⚠️ | Especialidad a consultar |
| `IDSEDE`, `IDMEDICO`, `MODALIDAD` | ❌ | Filtros opcionales |
| `IDAFILIADO` | ❌ | Si se envía, puede inferir sede del paciente |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):**

| Recordset | Contenido |
|-----------|-----------|
| `[0]` | `{ OK: "OK" }` |
| `[1]` | Hasta `TOP_N` cupos: `CONSECUTIVO`, `FECHA`, `DISPONIBLE`, `NOMBREMED`, `ESPECIALIDAD`, … |

**Errores de negocio (`OK = "KO"`):**

| `recordsets[1].ERROR` | Acción del agente |
|------------------------|-------------------|
| `Rango maximo 62 dias. Use DIA para consulta detallada por dia.` | Acortar el rango entre `FECHA_INI` y `FECHA_FIN` |

---

### 8 — `VERIFICAR_CUPO`

Comprueba que el cupo sigue libre antes de asignar (otros agentes pueden haberlo tomado).

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "VERIFICAR_CUPO",
  "PARAMETROS": { "CONSECUTIVO": "0100123456" }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `CONSECUTIVO` | ✅ | Identificador del cupo |

**HTTP:** `200`, `res: "ok"`.

**Cupo disponible:**

```json
{ "OK": "OK", "CUPO_DISPONIBLE": true, "EN_USO": false, "USUARIO_EN_USO": null }
```

**Cupo no disponible** (`OK = "OK"`, `CUPO_DISPONIBLE = false`, mensaje en `[1]`):

| `recordsets[1].ERROR` | Acción del agente |
|------------------------|-------------------|
| `Cupo en uso por otro usuario/agente` | Elegir otro cupo o reintentar |
| `Cupo ya asignado a un paciente` | No usar ese consecutivo |

**Errores de negocio (`OK = "KO"`):**

| `recordsets[1].ERROR` | Acción del agente |
|------------------------|-------------------|
| `Consecutivo de cita no existe` | Obtener cupo válido desde `CONSULTAR_DISPONIBILIDAD` |

---

### 9 — `VALIDAR_DISPONIBILIDAD`

Reserva temporalmente el cupo. Invocar **inmediatamente antes** de `ASIGNAR_CITA`.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "VALIDAR_DISPONIBILIDAD",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "CONSECUTIVO": "0100123456",
    "IDMEDICO": "DOC001",
    "ESPECIALIDAD": "0012"
  }
}
```

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):**

| Recordset | Contenido |
|-----------|-----------|
| `[0]` | `{ OK: "OK" }` |
| `[1]` | Centro de costo y área sugeridos |
| `[2]` | Opciones de centro de costo |
| `[3]` | Programas disponibles |
| `[4]` | Validación médico de cabecera |

Use los valores de `[1]`–`[4]` en `ASIGNAR_CITA` (`CCOSTO`, `IDAREA`, `PROGRAMA`, etc.).

**Errores de negocio (`OK = "KO"`, puede haber varias filas en `[1]`):**

| `recordsets[1].ERROR` (ejemplos) | Acción del agente |
|----------------------------------|-------------------|
| `Cita en uso por…` | Otro integrador reservó el cupo; elegir otro |
| `Afiliado ya tiene cita hoy…` | Informar al paciente |
| `El afiliado ya tiene una Cita para el mismo servicio…` | No duplicar servicio |

> Si la reserva falló pero la creó esta misma sesión, puede invocar `LIBERAR_CUPO` antes de reofertar.

---

### 10 — `LIBERAR_CUPO`

Libera la reserva temporal si no se completa la asignación.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "LIBERAR_CUPO",
  "PARAMETROS": {
    "CONSECUTIVO": "0100123456"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `CONSECUTIVO` | ✅ | Cupo reservado |

**HTTP:** `200`, `res: "ok"`.

**Respuesta (`OK = "OK"`):**

```json
{ "OK": "OK", "FILAS": 1 }
```

| `FILAS` | Significado |
|---------|-------------|
| `1` | Reserva liberada |
| `0` | No había reserva activa de esta sesión |

**Errores:** No devuelve `KO`. `FILAS = 0` no es fallo HTTP; continuar con otro cupo.

---

### 11 — `CONSULTAR_CAUSALES`

Catálogo de motivos para cancelación o reprogramación.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "CONSULTAR_CAUSALES",
  "PARAMETROS": { "TIPO": "CITCAN" }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `TIPO` | ❌ | `CITCAN` (cancelación, default) o `CITREPRO` (reprogramación) |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):**

| Recordset | Contenido |
|-----------|-----------|
| `[1]` | `{ IDCAUSAL, DESCCAUSAL }` |

**Errores:** No devuelve `KO`. Lista vacía si no hay causales para el tipo indicado.

---

### 12 — `CONSULTAR_SERVICIOS_ESPECIALIDAD`

Obtiene servicios disponibles para una especialidad y plan (usar `IDSERVICIO` en `ASIGNAR_CITA`).

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "CONSULTAR_SERVICIOS_ESPECIALIDAD",
  "PARAMETROS": {
    "IDEMEDICA": "0012",
    "IDPLAN": "PLN01"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `IDEMEDICA` o `ESPECIALIDAD` | ✅ | Especialidad |
| `IDPLAN` | ❌ | Plan; puede inferirse con `IDAFILIADO` |
| `IDAFILIADO` | ❌ | Alternativa para inferir plan |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):**

| Recordset | Contenido |
|-----------|-----------|
| `[1]` | Hasta 20 servicios: `{ IDSERVICIO, DESCSERVICIO, DURACIONCITA }` |

**Errores:** No devuelve `KO`. Lista vacía si no hay servicios para la combinación indicada.

---

### 13 — `CONFIRMAR_CITAS_PENDIENTES`

Lista citas del paciente pendientes de confirmación.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "CONFIRMAR_CITAS_PENDIENTES",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "FECHA_INI": "2024-06-01",
    "FECHA_FIN": "2024-06-30"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `IDAFILIADO` | ✅ | Paciente |
| `FECHA_INI`, `FECHA_FIN` | ❌ | Acotar ventana |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):**

| Recordset | Contenido |
|-----------|-----------|
| `[0]` | `{ OK, CANTIDAD }` — `CANTIDAD` puede ser `0` |
| `[1]` | Citas pendientes: `CONSECUTIVO`, `FECHA`, `ESPECIALIDAD`, `ESTADO_CITA`, … |

**Errores:** No devuelve `KO`.

---

### 14 — `ASIGNAR_CITA`

Confirma la asignación de la cita al paciente. Ejecutar solo tras `VALIDAR_DISPONIBILIDAD` exitoso.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "ASIGNAR_CITA",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "CONSECUTIVO": "0100123456",
    "ESPECIALIDAD": "0012",
    "SERVICIO": "890201",
    "PROGRAMA": "NEPS01",
    "CCOSTO": "0101",
    "IDAREA": "01",
    "VIRTUAL": false,
    "UNIDNEGO": "UNG01",
    "CLASEORDEN": "Modelo",
    "TIPOSOLICITUD": "Normal",
    "TIPOATENCION": "Ambulatoria",
    "OBSERVACIONES": "Gestión IA ArtMedica"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `IDAFILIADO`, `CONSECUTIVO` | ✅ | Paciente y cupo reservado |
| `ESPECIALIDAD`, `SERVICIO` | ✅ | Especialidad y servicio |
| `PROGRAMA` | ⚠️ | Según reglas del programa del paciente |
| `CCOSTO`, `IDAREA` | ⚠️ | Valores de `VALIDAR_DISPONIBILIDAD` |
| `OBSERVACIONES` | ❌ | Texto libre para trazabilidad |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):**

| Recordset | Contenido |
|-----------|-----------|
| `[0]` | Confirmación; puede incluir datos de copago según configuración |
| `[1+]` | Detalle adicional de la cita asignada |

**Errores de negocio (`OK = "KO"`, múltiples filas posibles):**

| `recordsets[1].ERROR` (ejemplos) | Acción del agente |
|------------------------------------|-------------------|
| `La cita ya fue ocupada.` | `LIBERAR_CUPO` + reofertar |
| `El paciente no se encuentra Activo ni Autorizado.` | No gestionar |
| `Paciente no se encuentra activo en el programa` | Verificar programa |
| `Segun el Plan y el Tercero, el N° de Autorizacion es Obligatorio.` | Solicitar autorización al paciente |
| `El afiliado ya tiene una Cita para el mismo servicio aun vigente…` | No duplicar servicio |

---

### 15 — `REPROGRAMAR_CITA`

Cambia una cita existente por un nuevo cupo.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "REPROGRAMAR_CITA",
  "PARAMETROS": {
    "CONSECUTIVOANT": "0100123456",
    "CONSECUTIVONUE": "0100987654",
    "MOTIVO": "001",
    "OBSERVACION": "Reprogramación vía agente IA",
    "MARCA": 0
  }
}
```

| Campo | Requerido | Descripción |
|-------|-----------|-------------|
| `CONSECUTIVOANT` | ✅ | Cita actual |
| `CONSECUTIVONUE` | ✅* | Nuevo cupo libre (cuando `MARCA=0`) |
| `MOTIVO` | ✅ | `IDCAUSAL` de `CONSULTAR_CAUSALES` (`CITREPRO`) |
| `OBSERVACION` | ❌ | Texto libre |
| `MARCA` | ❌ | `0` intercambio de cupos / `1` reprogramación sin cupo previo |
| `IDMEDICO`, `IDEMEDICA`, `FECHA_CITA`, `HORA_CITA` | ⚠️ | Requeridos si `MARCA=1` |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):** confirmación en `[0]` y detalle en bloques siguientes.

**Errores de negocio (`OK = "KO"`):**

| `recordsets[1].ERROR` (ejemplos) | Acción del agente |
|----------------------------------|-------------------|
| `Cita en uso por…` | Cupo no disponible; elegir otro |
| `No se puede Reprogramar la Cita, por que ya fue Atendida…` | Informar al paciente |
| `Afiliado Inactivo…` | No gestionar |

---

### 16 — `CANCELAR_CITA`

Cancela una cita vigente.

**Request:**

```json
{
  "MODELO": "CIT_COL_4",
  "METODO": "CANCELAR_CITA",
  "PARAMETROS": {
    "CONSECUTIVO": "0100123456",
    "IDCAUSAL": "001",
    "OBSERVACION": "Cancelación solicitada por paciente vía IA",
    "MOTIVO": "001"
  }
}
```

| Parámetro | Requerido | Descripción |
|-----------|-----------|-------------|
| `CONSECUTIVO` | ✅ | Cita a cancelar |
| `IDCAUSAL` / `MOTIVO` | ✅ | De `CONSULTAR_CAUSALES` (`CITCAN`) |
| `OBSERVACION` | ❌ | Texto libre |

**HTTP:** `200`, `res: "ok"`.

**Respuesta exitosa (`OK = "OK"`):** confirmación en `[0]`.

**Errores de negocio (`OK = "KO"`):**

| `recordsets[1].ERROR` (ejemplos) | Acción del agente |
|----------------------------------|-------------------|
| Cita facturada / recaudada | No cancelable; orientar al paciente |
| Cita ya atendida | No cancelable |
| Otros mensajes en `ERROR` | Mostrar mensaje al paciente o escalar a soporte |

---

## Referencia rápida

| METODO | HTTP | `res` | Devuelve `OK=KO` | Notas |
|--------|------|-------|------------------|-------|
| `VALIDAR_PACIENTE` | 200 | ok | Sí | Errores de población/programa |
| `ACTUALIZAR_DATOS_PACIENTE` | 200 | ok | Sí | `IDAFILIADO invalido` |
| `CONSULTAR_CITAS` | 200 | ok | Sí | Falta IDAFILIADO o CONSECUTIVO |
| `CONSULTAR_CITAS_PERIODO` | 200 | ok | Sí | Parámetros requeridos |
| `CONSULTAR_ATENCION_ESPECIALIDAD` | 200 | ok | Sí | Parámetros requeridos |
| `EVALUAR_ELEGIBILIDAD_GESTION` | 200 | ok | No | Decisión en flags `[0]` |
| `CONSULTAR_DISPONIBILIDAD` | 200 | ok | Sí | Rango máx. 62 días; `TOP_N` default 3 |
| `VERIFICAR_CUPO` | 200 | ok | Parcial | `KO` si no existe; aviso en `[1]` si ocupado |
| `VALIDAR_DISPONIBILIDAD` | 200 | ok | Sí | Reserva temporal |
| `LIBERAR_CUPO` | 200 | ok | No | Revisar `FILAS` |
| `CONSULTAR_CAUSALES` | 200 | ok | No | Lista puede ser vacía |
| `CONSULTAR_SERVICIOS_ESPECIALIDAD` | 200 | ok | No | Lista puede ser vacía |
| `CONFIRMAR_CITAS_PENDIENTES` | 200 | ok | No | `CANTIDAD=0` válido |
| `ASIGNAR_CITA` | 200 | ok | Sí | Tras reserva exitosa |
| `REPROGRAMAR_CITA` | 200 | ok | Sí | Requiere causal |
| `CANCELAR_CITA` | 200 | ok | Sí | Requiere causal |

---

## Diagrama — demanda inducida

```
Autenticación → VALIDAR_PACIENTE
       ↓ ¿GESTIONAR?
CONSULTAR_CITAS_PERIODO → ¿cita en periodo? → FIN
       ↓
EVALUAR_ELEGIBILIDAD_GESTION → ACCION_SUGERIDA + IDEMEDICA_OBJETIVO
       ↓
CONSULTAR_DISPONIBILIDAD (TOP_N=3) → presentar opciones al paciente
       ↓
VERIFICAR_CUPO → VALIDAR_DISPONIBILIDAD → ASIGNAR_CITA
       ↓ fallo por concurrencia
LIBERAR_CUPO → reofertar cupos
```

---

## Notas para el integrador

**Token:** Renovar en cada respuesta. No reutilizar tokens expirados.

**Fechas:** Los responses usan ISO 8601 (`2024-06-15T10:30:00.000Z`). Formatear en zona horaria local para el paciente.

**Concurrencia:** Varios agentes pueden competir por el mismo cupo. Siempre ejecutar `VERIFICAR_CUPO` → `VALIDAR_DISPONIBILIDAD` → `ASIGNAR_CITA` en secuencia corta.

**Soporte:** Ante errores `500`, mensajes no documentados o necesidad de nuevos métodos, contactar al equipo **ArtMedica / Qrystalos**. No extrapolar comportamientos no descritos en este documento.

**Versión:** Documento aplicable a integración Agente IA ArtMedica — modelo `CIT_COL_4`.

---

## Colección Postman

La integración es **compatible con Postman**. Se incluyen archivos importables en:

| Archivo | Descripción |
|---------|-------------|
| `runbooks/postman/ArtMedica_AgenteIA_CIT_COL_4.postman_collection.json` | Colección con login, flujo demanda inducida y 16 operaciones |
| `runbooks/postman/ArtMedica_AgenteIA.postman_environment.json` | Variables de entorno (credenciales y códigos de catálogo) |

**Importar en Postman**

1. **Import** → seleccionar ambos archivos JSON.
2. Activar el environment **ArtMedica — Agente IA (sandbox)**.
3. Completar `usuario`, `clave` y códigos entregados por soporte.
4. Ejecutar **Login integrador** (guarda `jwt` automáticamente).
5. Ejecutar el folder **Flujo — Demanda inducida** o operaciones individuales.

La colección renueva `jwt` en cada respuesta que lo incluya (script a nivel de colección).
