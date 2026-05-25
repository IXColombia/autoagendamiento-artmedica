# API de Autoagendamiento — ArtMedica

Documentación técnica de los endpoints involucrados en el flujo completo de autoagendamiento de citas para pacientes externos. Dirigida al proveedor que integre contra la API de **ArtMedica** (`COMPANIA = "AR"`).

---

## Base URL

```
https://reportes-artmedica.qrystalos.com/api/
```

Todos los endpoints se invocan contra esta base. El campo `compania` en los requests de autenticación siempre debe enviarse con el valor `"AR"`.

---

## Autenticación

### Esquema general

La API usa **JWT Bearer Token**. Una vez autenticado el paciente, cada request debe incluir el header:

```
Authorization: Bearer <jwt>
```

El servidor renueva el JWT automáticamente en cada respuesta cuando corresponde. El cliente debe leer el campo `jwt` del body de respuesta y reemplazar el token almacenado.

Si el JWT expira, el servidor responde `401`. El cliente debe limpiar la sesión y redirigir al login.

### Formato de request (endpoints autenticados vía `POST /json`)

La mayoría de los endpoints del flujo de citas se invocan a través de un único endpoint genérico:

```
POST /json
Content-Type: application/json
Authorization: Bearer <jwt>

{
  "MODELO": "<nombre_modelo>",
  "METODO": "<nombre_metodo>",
  "PARAMETROS": { ... }
}
```

### Formato de respuesta estándar

```json
{
  "jwt": "<nuevo_jwt_o_null>",
  "result": {
    "recordset": [{ "OK": "OK" }],
    "recordsets": [
      [{ "OK": "OK" }],
      [ ...datos principales... ],
      [ ...datos secundarios si aplica... ]
    ]
  }
}
```

- `recordset[0].OK === "OK"` indica éxito.
- Si `OK !== "OK"`, `recordsets[1]` puede contener el detalle del error.
- Si el servidor retorna `logout: true`, el cliente debe cerrar sesión.

---

## Flujo completo paso a paso

```
[1] POST /cit/identificar_paciente         → Enviar OTP al correo del paciente
[2] POST /cit/autenticar                   → Validar OTP → obtiene JWT
    (Alternativa) POST /ext/registro/AR    → Autoregistro de paciente nuevo
[3] POST /json  CIT_COL_3 / CITAS_PACIENTE          → Citas vigentes e historial
[4] POST /json  CIT_COL_3 / CITAS_EN_LISTA_ESPERA   → Lista de espera activa
[5] POST /json  CIT_COL_3 / LISTAR_AFILIADOS        → Afiliados del usuario
[6] POST /json  CIT_COL_3 / LISTAR_PLANES           → Planes/convenios del afiliado
     ó  CIT_COL_3 / LISTAR_PLANES_PCITAWEB          → (si CITWEB_FILT_PLANES = SI)
[7] POST /json  CIT_COL_3 / LISTAR_ESPECIALIDADES   → Especialidades disponibles
[8] POST /json  CIT_COL_3 / LISTAR_MEDICOS          → Profesionales por especialidad
[9] POST /json  CIT_COL_3 / LISTAR_SERVICIOS        → Servicios por especialidad y plan
[10] POST /json CIT_COL_3 / LISTAR_SEDES            → Sedes por médico y especialidad
[11] POST /json CIT_COL_3 / LISTAR_CITAS            → Slots disponibles para agendar
[12] POST /json CIT_COL_3 / AGENDAR_CITA            → Confirmar y agendar la cita
[13] POST /json CIT_COL_3 / CANCELAR_CITA           → Cancelar cita vigente
[14] POST /json CIT_COL_3 / AGREGAR_LISTA_ESPERA    → Ingresar a lista de espera
[15] POST /json CIT_COL_3 / ELIMINAR_LISTA_ESPERA   → Salir de lista de espera
```

---

## Endpoints detallados

---

### [1] Identificar paciente (paso 1 del login)

```
POST https://reportes-artmedica.qrystalos.com/api/cit/identificar_paciente
Content-Type: application/json
```

**Sin autenticación.** Recibe tipo y número de documento del paciente. Si el paciente existe en ArtMedica, el servidor envía un código OTP al email registrado.

**Request body:**

```json
{
  "tipodoc": "CC",
  "docidafiliado": "1234567890",
  "compania": "AR"
}
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `tipodoc` | string | ✅ | Tipo de documento. Ver tabla de tipos abajo. |
| `docidafiliado` | string | ✅ | Número de documento del paciente. |
| `compania` | string | ✅ | Siempre `"AR"` para ArtMedica. |

**Tipos de documento válidos:**

| Código | Descripción |
|---|---|
| `AS` | Adulto sin identificar |
| `CC` | Cédula de Ciudadanía |
| `CD` | Carnet Diplomático |
| `CE` | Cédula de Extranjería |
| `CN` | Nacido Vivo CN |
| `DE` | Documento extranjero |
| `MS` | Menor sin identificar |
| `NIT` | NIT |
| `PA` | Pasaporte |
| `PE` | Permiso Especial de Permanencia |
| `PT` | Permiso Protección Temporal |
| `RC` | Registro Civil |
| `SC` | Salvo Conducto |
| `SI` | Sin Identificación |
| `TI` | Tarjeta de Identidad |

**Response exitoso:**

```json
{
  "result": {
    "recordset": [
      {
        "OK": "OK",
        "EMAIL": "pac***@correo.com"
      }
    ]
  }
}
```

El campo `EMAIL` es el correo enmascarado donde se envió el OTP. Mostrarlo al usuario para confirmar que es el correo correcto.

---

### [2] Autenticar paciente — validar OTP (paso 2 del login)

```
POST https://reportes-artmedica.qrystalos.com/api/cit/autenticar
Content-Type: application/json
```

**Sin autenticación.** Valida el código OTP de 6 dígitos. Si es correcto, retorna el JWT y los datos del afiliado.

**Request body:**

```json
{
  "codigo": "123456",
  "compania": "AR"
}
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `codigo` | string | ✅ | Código OTP de 6 dígitos (sin espacios). |
| `compania` | string | ✅ | Siempre `"AR"` para ArtMedica. |

**Response exitoso:**

```json
{
  "jwt": "<token_jwt>",
  "result": {
    "recordset": [{ "OK": "OK" }],
    "recordsets": [
      [{ "OK": "OK" }],
      [
        {
          "IDAFILIADO": "AFI001",
          "NOMBREAFI": "Juan Pérez",
          "TIPODOC": "CC",
          "DOCIDAFILIADO": "1234567890",
          "EMAIL": "juan@correo.com",
          "IDPLAN": "PLN01",
          "COMPANIA": "AR"
        }
      ]
    ]
  }
}
```

`recordsets[1][0]` contiene los datos del afiliado autenticado. El `jwt` debe almacenarse y enviarse en todas las solicitudes subsiguientes.

---

### [2b] Autoregistro de paciente (opcional — si habilitado por ArtMedica)

#### Cargar aseguradoras disponibles

```
GET https://reportes-artmedica.qrystalos.com/api/ext/registro/aseguradoras/AR
```

**Sin autenticación.**

**Response:**

```json
{
  "res": "ok",
  "result": {
    "recordset": [
      { "value": "TER001", "label": "EPS Ejemplo" }
    ]
  }
}
```

#### Cargar planes por aseguradora

```
GET https://reportes-artmedica.qrystalos.com/api/ext/registro/planes/AR/:idtercero
```

**Sin autenticación.**

**Response:**

```json
{
  "res": "ok",
  "result": {
    "recordset": [
      { "value": "PLN01", "label": "Plan Básico" }
    ]
  }
}
```

#### Cargar sedes

```
GET https://reportes-artmedica.qrystalos.com/api/ext/registro/sedes/AR
```

**Sin autenticación.**

**Response:**

```json
{
  "res": "ok",
  "result": {
    "recordset": [
      { "value": "SED01", "label": "Sede Principal" }
    ]
  }
}
```

#### Registrar paciente

```
POST https://reportes-artmedica.qrystalos.com/api/ext/registro/AR
Content-Type: application/json
```

**Sin autenticación.**

**Request body:**

```json
{
  "PARAMETROS": {
    "TIPO_DOC": "CC",
    "DOCIDAFILIADO": "1234567890",
    "CIUDADDOC": "11001",
    "ESTADO": "Activo",
    "PAPELLIDO": "Pérez",
    "SAPELLIDO": "Gómez",
    "PNOMBRE": "Juan",
    "SNOMBRE": "Carlos",
    "FNACIMIENTO": "1990-05-15",
    "SEXO": "Masculino",
    "ESTADO_CIVIL": "Soltero",
    "GRUPO_SANG": "O+",
    "GRUPOPOB": "",
    "GRUPOETNICO": "",
    "TIPODISCAPACIDAD": "",
    "IDESCOLARIDAD": "",
    "DIRECCION": "Calle 123 # 45-67",
    "TELEFONORES": "6017001234",
    "CELULAR": "3001234567",
    "CIUDAD": "11001",
    "ZONA": "U",
    "IDBARRIO": "",
    "EMAIL": "juan@correo.com",
    "IDADMINISTRADORA": "TER001",
    "IDPLAN": "PLN01",
    "IDSEDE": "SED01",
    "FECHAAFILIACION": "2024-01-01",
    "PREFIJO_CELULAR": "+57",
    "PREFIJO_TELEFONORES": "+57",
    "IDEMPLEADOR": null,
    "IDPAIS": "57",
    "NIVELSOCIOEC": "",
    "ESTRATO": "3",
    "TIPOUSUARIO": "",
    "PROCEDENCIA": "WEB"
  }
}
```

**Response exitoso:**

```json
{
  "res": "ok",
  "result": {
    "recordset": [
      { "OK": "OK", "MENSAJE": "Paciente registrado exitosamente" }
    ]
  }
}
```

---

### [3] Citas del paciente (vigentes e historial)

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "CITAS_PACIENTE"
}
```

**Sin parámetros adicionales.** El servidor usa el JWT para identificar al paciente.

**Response:**

```json
{
  "result": {
    "recordset": [{ "OK": "OK" }],
    "recordsets": [
      [{ "OK": "OK" }],
      [ ...citas vigentes... ],
      [ ...historial de citas... ]
    ]
  }
}
```

`recordsets[1]` → citas vigentes (por cumplir).  
`recordsets[2]` → historial (atendidas y no cumplidas).

**Estructura de cada cita:**

```json
{
  "CONSECUTIVO": "CIT20240001",
  "FECHA": "2024-06-15T10:30:00.000Z",
  "FDESCRITA": "Sábado 15 de junio de 2024 a las 10:30",
  "CUMPLIDA": "Pendiente",
  "CUMPLIDA_NUM": 0,
  "CIT_ESTADO": "Activo",
  "SER": "[{\"IDSERVICIO\":\"CON001\",\"DESCSERVICIO\":\"Consulta General\"}]",
  "MES": "[{\"IDEMEDICA\":\"MED01\",\"DESCRIPCION\":\"Medicina General\"}]",
  "MED": "[{\"IDMEDICO\":\"DOC001\",\"NOMBRE\":\"Dr. Rodríguez\"}]",
  "SED": "[{\"IDSEDE\":\"SED01\",\"DESCRIPCION\":\"Sede Principal\"}]",
  "AFI": "[{\"IDAFILIADO\":\"AFI001\",\"NOMBREAFI\":\"Juan Pérez\",\"EMAIL\":\"juan@correo.com\"}]"
}
```

> **Nota:** Los campos `SER`, `MES`, `MED`, `SED`, `AFI` vienen como JSON strings. El cliente debe parsearlo con `JSON.parse()`. Si el array tiene elementos, tomar `[0]`.

---

### [4] Lista de espera del paciente

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "CITAS_EN_LISTA_ESPERA"
}
```

**Response:** `recordsets[1]` con las entradas activas en lista de espera.

```json
{
  "CNSNOCIT": "LE20240001",
  "FREQ_DESCRITA": "Lunes 10 de junio de 2024 a las 09:00",
  "ESPECIALIDAD": "Medicina General",
  "PROFESIONAL": "Dr. Rodríguez",
  "SERVICIOS": "[{\"IDSERVICIO\":\"CON001\",\"DESCSERVICIO\":\"Consulta General\"}]"
}
```

> `SERVICIOS` viene como JSON string. Parsear antes de usar.

---

### [5] Listar afiliados del usuario

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "LISTAR_AFILIADOS"
}
```

Retorna los afiliados vinculados al usuario autenticado (el paciente y posibles beneficiarios).

**Response:** `recordsets[1]` con array de afiliados.

```json
{
  "IDAFILIADO": "AFI001",
  "NOMBREAFI": "Juan Pérez",
  "TIPODOC": "CC",
  "DOCIDAFILIADO": "1234567890",
  "IDPLAN": "PLN01"
}
```

---

### [6] Listar planes/convenios del afiliado

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

**Método estándar:**

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "LISTAR_PLANES",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001"
  }
}
```

**Método con filtro por cita web** (si la variable del sistema `CITWEB_FILT_PLANES = SI`):

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "LISTAR_PLANES_PCITAWEB",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001"
  }
}
```

> El cliente debe consultar primero la variable `CITWEB_FILT_PLANES` (ver sección Variables del Sistema) para determinar qué método usar.

**Response:** `recordsets[1]` con los planes disponibles.

```json
{
  "IDPLAN": "PLN01",
  "DESCPLAN": "Plan Básico",
  "RAZONSOCIAL": "EPS Ejemplo",
  "IDTERCERO": "TER001",
  "PARTICULAR": false,
  "CHECKED": true
}
```

| Campo | Descripción |
|---|---|
| `IDPLAN` | ID del plan. Requerido en pasos subsiguientes. |
| `PARTICULAR` | Si `true`, mostrar precio estimado en la UI. |
| `CHECKED` | Si `true`, seleccionar este plan por defecto. |

---

### [7] Listar especialidades

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

**Sin filtro por plan:**

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "LISTAR_ESPECIALIDADES"
}
```

**Con filtro por plan** (si `CITWEB_ESP_PLAN = SI`):

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "LISTAR_ESPECIALIDADES",
  "PARAMETROS": {
    "IDPLAN": "PLN01"
  }
}
```

**Response:** `recordsets[1]` con especialidades habilitadas para autoagenda web.

```json
{
  "IDEMEDICA": "MED01",
  "DESCRIPCION": "Medicina General"
}
```

---

### [8] Listar profesionales por especialidad

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "LISTAR_MEDICOS",
  "PARAMETROS": {
    "IDEMEDICA": "MED01"
  }
}
```

**Response:** `recordsets[1]` con médicos habilitados para agenda web.

```json
{
  "IDMEDICO": "DOC001",
  "NOMBRE": "Dr. Carlos Rodríguez"
}
```

> Si el array retorna exactamente 1 profesional, seleccionarlo automáticamente.

---

### [9] Listar servicios por especialidad y plan

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "LISTAR_SERVICIOS",
  "PARAMETROS": {
    "IDEMEDICA": "MED01",
    "IDPLAN": "PLN01",
    "IDTERCERO": "TER001"
  }
}
```

**Response:** `recordsets[1]` con los servicios disponibles.

```json
{
  "IDSERVICIO": "CON001",
  "DESCSERVICIO": "Consulta de Primera Vez",
  "DURACIONCITA": 30,
  "PRECIO_PART": 80000,
  "DIAS_REAGENDA": 90
}
```

| Campo | Descripción |
|---|---|
| `IDSERVICIO` | ID del servicio. Puede seleccionarse múltiple. |
| `DURACIONCITA` | Duración en minutos. Sumar si se seleccionan múltiples servicios. |
| `PRECIO_PART` | Precio para pacientes particulares. Solo mostrar si `PLN.PARTICULAR = true`. |
| `DIAS_REAGENDA` | Días mínimos que deben pasar desde la última cita atendida para poder reagendar. `0` = sin restricción. |

---

### [10] Listar sedes disponibles

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "LISTAR_SEDES",
  "PARAMETROS": {
    "IDEMEDICA": "MED01",
    "IDMEDICO": "DOC001"
  }
}
```

**Response:** `recordsets[1]` con sedes donde el médico atiende esa especialidad.

```json
{
  "IDSEDE": "SED01",
  "DESCRIPCION": "Sede Principal - Calle 123"
}
```

> Si el array retorna exactamente 1 sede, seleccionarla automáticamente.

---

### [11] Listar slots de cita disponibles

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "LISTAR_CITAS",
  "PARAMETROS": {
    "IDEMEDICA": "MED01",
    "IDMEDICO": "DOC001",
    "IDSEDE": "SED01",
    "FECHA": null,
    "IDPLAN": "PLN01",
    "DURACIONCITA": 30,
    "SERVICIOS": "CON001,CON002"
  }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `IDEMEDICA` | string | Especialidad seleccionada. |
| `IDMEDICO` | string | Profesional seleccionado. |
| `IDSEDE` | string | Sede seleccionada. |
| `FECHA` | string \| null | Fecha en formato `YYYYMMDD`. Si es `null`, el SP retorna los próximos slots disponibles. |
| `IDPLAN` | string | Plan del afiliado. |
| `DURACIONCITA` | int | Suma de minutos de todos los servicios seleccionados. |
| `SERVICIOS` | string | IDs de servicios separados por coma. |

**Response:** `recordsets[1]` con los slots disponibles, ordenados por fecha/hora.

```json
{
  "CONSECUTIVO": "AG20240015",
  "FECHA": "2024-06-15T10:30:00.000Z",
  "FDESCRITA": "Sábado 15 de junio de 2024 a las 10:30"
}
```

> El primer elemento del array (`recordsets[1][0]`) es la cita más cercana disponible. Ofrecerla como acceso directo en la UI. Si el array está vacío, no hay slots programados → derivar a lista de espera.

**Comportamiento del cliente:**
- Invocar este endpoint apenas se seleccionen los servicios (con debounce de ~1 segundo).
- Re-invocar si cambia la fecha, el médico o los servicios.

---

### [12] Agendar cita

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "AGENDAR_CITA",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "CONSECUTIVO": "AG20240015",
    "IDSERVICIOS": "CON001,CON002",
    "IDEMEDICO": "DOC001",
    "IDSEDE": "SED01",
    "IDEMEDICA": "MED01"
  }
}
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `IDAFILIADO` | string | ✅ | ID del afiliado para quien se agenda. |
| `CONSECUTIVO` | string | ✅ | Consecutivo del slot elegido en `LISTAR_CITAS`. |
| `IDSERVICIOS` | string | ✅ | IDs de servicios separados por coma. |
| `IDEMEDICO` | string | ✅ | ID del médico seleccionado. |
| `IDSEDE` | string | ✅ | ID de la sede seleccionada. |
| `IDEMEDICA` | string | ✅ | ID de la especialidad seleccionada. |

**Response exitoso:**

```json
{
  "result": {
    "recordset": [
      {
        "OK": "OK",
        "CONSECUTIVO": "CIT20240001",
        "EMAIL": "juan@correo.com"
      }
    ]
  }
}
```

**Response de error:**

```json
{
  "result": {
    "recordset": [{ "OK": "ERROR", "ERROR": "El paciente ya tiene una cita para este servicio el mismo día.", "MENSAJE": "..." }],
    "recordsets": [
      [{ "OK": "ERROR" }],
      [{ "ERROR": "Detalle del error" }]
    ]
  }
}
```

> Tras un agendamiento exitoso, el `CONSECUTIVO` retornado en `recordset[0]` es el número de la cita confirmada. Usarlo para generar el comprobante (PDF).

#### Validaciones de negocio que el cliente DEBE aplicar antes de llamar este endpoint

Estas reglas están implementadas en el frontend y deben replicarse en cualquier integración para evitar rechazos del servidor:

**Regla 1 — Una cita por servicio por día** (aplica si `CITWEB_UNA_CITA_DIA = SI`):  
Si el paciente ya tiene una cita (vigente o en historial) para el mismo `IDSERVICIO` en la misma fecha que el slot seleccionado, mostrar error y no invocar `AGENDAR_CITA`.

**Regla 2 — Días mínimos de reagenda** (aplica si `CITWEB_UNA_CITA_DIA = SI`):  
Para cada servicio seleccionado, verificar `DIAS_REAGENDA` (obtenido en `LISTAR_SERVICIOS`). Si la última cita **atendida** (`CUMPLIDA_NUM = 1`) del paciente para ese servicio es menor a `DIAS_REAGENDA` días antes de la fecha del slot, mostrar error.

> Ambas reglas usan los datos ya obtenidos en `CITAS_PACIENTE` sin requerir llamadas adicionales al servidor.

---

### [13] Cancelar cita

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "CANCELAR_CITA",
  "PARAMETROS": {
    "CONSECUTIVO": "CIT20240001"
  }
}
```

**Response exitoso:** `recordset[0].OK === "OK"`.

> Tras cancelar, recargar `CITAS_PACIENTE` para refrescar la lista.

---

### [14] Agregar a lista de espera

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "AGREGAR_LISTA_ESPERA",
  "PARAMETROS": {
    "IDAFILIADO": "AFI001",
    "IDEMEDICA": "MED01",
    "IDMEDICO": "DOC001",
    "SERVICIOS": "CON001,CON002",
    "FECHAREQ": "20240615 09:00",
    "OBSERVACION": "Comentario adicional del paciente"
  }
}
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `IDAFILIADO` | string | ✅ | ID del afiliado. |
| `IDEMEDICA` | string | ✅ | ID de la especialidad. |
| `IDMEDICO` | string | ❌ | ID del médico preferido (puede ser null). |
| `SERVICIOS` | string | ✅ | IDs de servicios separados por coma. |
| `FECHAREQ` | string | ✅ | Fecha y hora ideal en formato `YYYYMMDD HH:mm`. |
| `OBSERVACION` | string | ❌ | Comentarios adicionales del paciente. |

**Response exitoso:** `recordset[0].OK === "OK"`.

#### Validaciones previas (igual que `AGENDAR_CITA`)

Aplicar las mismas reglas de `CITWEB_UNA_CITA_DIA` usando `FECHAREQ` como fecha de referencia antes de invocar este endpoint.

---

### [15] Salir de lista de espera

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "CIT_COL_3",
  "METODO": "ELIMINAR_LISTA_ESPERA",
  "PARAMETROS": {
    "CNSNOCIT": "LE20240001"
  }
}
```

| Campo | Descripción |
|---|---|
| `CNSNOCIT` | Consecutivo de la entrada en lista de espera obtenido en `CITAS_EN_LISTA_ESPERA`. |

**Response exitoso:** `recordset[0].OK === "OK"`.

---

## Variables del sistema

Antes de iniciar el flujo de agendamiento, el cliente debe consultar estas 4 variables. Controlan comportamientos opcionales de la UI y las reglas de negocio.

```
POST https://reportes-artmedica.qrystalos.com/api/json
Authorization: Bearer <jwt>
```

```json
{
  "MODELO": "USVGS_COL",
  "METODO": "VALORVARIABLE",
  "PARAMETROS": {
    "IDVARIABLE": "CITWEB_FILT_PLANES"
  }
}
```

**Response:** `recordsets[1][0].DATO` contiene el valor de la variable.

| Variable | Valores | Efecto |
|---|---|---|
| `CITWEB_FILT_PLANES` | `SI` / vacío | Si `SI`, usar `LISTAR_PLANES_PCITAWEB` en lugar de `LISTAR_PLANES`. |
| `CITWEB_UNA_CITA_DIA` | `SI` / vacío | Si `SI`, aplicar validaciones de reagenda antes de agendar o entrar a lista de espera. |
| `CITWEB_PLAN_AFI` | `SI` / vacío | Si `SI`, preseleccionar el plan del afiliado (`IDPLAN` del usuario autenticado) y no permitir cambiarlo. |
| `CITWEB_ESP_PLAN` | `SI` / vacío | Si `SI`, filtrar especialidades por plan seleccionado al invocar `LISTAR_ESPECIALIDADES`. |

> Consultar las 4 variables en paralelo al cargar el módulo de citas. El cliente debe cachear los valores durante la sesión para no repetir la consulta.

---

## Manejo de errores HTTP

| Código | Significado | Acción del cliente |
|---|---|---|
| `401` | JWT expirado o inválido | Limpiar sesión y redirigir al login. |
| `404` | URL incorrecta o servidor caído | Mostrar error de conexión. |
| `500` | Error interno del servidor | Parsear `error.originalError.message` para mostrar el detalle. |

---

## Diagrama de flujo de agendamiento

```
Inicio
  │
  ├─[POST /cit/identificar_paciente]────────── Ingresar tipo+doc → OTP al email
  │
  ├─[POST /cit/autenticar]──────────────────── Validar OTP → JWT
  │    └─ (Alternativa: /ext/registro/AR ───── Autoregistro)
  │
  ├─ Cargar variables del sistema (x4 en paralelo)
  │
  ├─[CIT_COL_3 / CITAS_PACIENTE]────────────── Citas vigentes + historial
  ├─[CIT_COL_3 / CITAS_EN_LISTA_ESPERA]──────── Lista de espera activa
  │
  └─ Botón "Agendar Nueva Cita"
       │
       ├─[CIT_COL_3 / LISTAR_AFILIADOS]──────── ¿Para quién? (afiliado)
       ├─[CIT_COL_3 / LISTAR_PLANES]──────────── ¿Qué convenio? (plan)
       ├─[CIT_COL_3 / LISTAR_ESPECIALIDADES]──── ¿Qué especialidad?
       ├─[CIT_COL_3 / LISTAR_MEDICOS]──────────── ¿Qué profesional?
       ├─[CIT_COL_3 / LISTAR_SERVICIOS]─────────── ¿Qué servicio(s)?
       ├─[CIT_COL_3 / LISTAR_SEDES]──────────────── ¿Dónde?
       ├─[CIT_COL_3 / LISTAR_CITAS]──────────────── ¿Cuándo? (slots disponibles)
       │    │
       │    ├─ Hay slots disponibles
       │    │    └─[CIT_COL_3 / AGENDAR_CITA] ────── Confirmar cita ✅
       │    │
       │    └─ No hay slots
       │         └─[CIT_COL_3 / AGREGAR_LISTA_ESPERA] ── Lista de espera ⏳
       │
       └─ Desde lista vigente:
            ├─[CIT_COL_3 / CANCELAR_CITA]──────── Cancelar cita ❌
            └─[CIT_COL_3 / ELIMINAR_LISTA_ESPERA] ─ Salir de espera 🚪
```

---

## Notas de implementación

**Patrón de request unificado:** Casi todos los endpoints del flujo de citas usan `POST /json` con el mismo esquema `{ MODELO, METODO, PARAMETROS }`. El enrutamiento real lo hace el servidor según `MODELO` + `METODO`. No hay URLs individuales por operación.

**Compañía fija:** En todos los endpoints de autenticación y autoregistro, el campo `compania` siempre debe enviarse como `"AR"`. No es necesario consultarlo ni derivarlo dinámicamente.

**Selección automática:** Si `LISTAR_MEDICOS` retorna un solo elemento, seleccionarlo automáticamente sin interacción del usuario. Mismo comportamiento para `LISTAR_SEDES`.

**Slots en tiempo real:** `LISTAR_CITAS` debe invocarse con `FECHA: null` para que el SP retorne los próximos slots disponibles sin requerir que el usuario seleccione una fecha. El primer resultado es la cita más próxima.

**Re-fetch de servicios:** Cuando el usuario cambia de especialidad, limpiar `SER`, `MED`, `SED`, `CONSECUTIVO` y `FECHA` del formulario antes de cargar los nuevos profesionales y servicios.

**Comprobante PDF:** Tras un agendamiento exitoso, el servidor retorna el `CONSECUTIVO` de la cita. Usarlo para generar el PDF del comprobante a través del endpoint de impresión (endpoint separado, consultar con el equipo Qrystalos).

**Zona horaria:** Las fechas en los responses vienen en formato ISO 8601 (`2024-06-15T10:30:00.000Z`). Para mostrar en UI, parsear localmente. El campo `FDESCRITA` ya viene formateado en español para mostrar directamente.

**Refresco post-acción:** Después de agendar, cancelar, agregar o salir de lista de espera, siempre recargar `CITAS_PACIENTE` y `CITAS_EN_LISTA_ESPERA` para mantener el estado sincronizado.

---

## Tablas involucradas en el flujo

Referencia del modelo de datos Qrystalos que respalda este API. Útil para diagnóstico, validaciones y entender la semántica de cada campo.

---

### AFI — Afiliados / Pacientes

Maestro de pacientes. Es la tabla central del sistema: referenciada por más de 43 tablas.

| Campo | Descripción |
|---|---|
| `IDAFILIADO` | PK. Identificador único del afiliado. |
| `NOMBREAFI` | Nombre completo del paciente. |
| `TIPO_DOC` | Tipo de documento de identidad (CC, TI, CE, PA, etc.). |
| `DOCIDAFILIADO` | Número de documento. Usado en el login del paciente. |
| `EMAIL` | Correo electrónico. Destino del OTP de autenticación. |
| `CELULAR` | Celular. Usado para OTP vía SMS / WhatsApp (procesado en servidor). |
| `IDPLAN` | FK → PLN. Plan de salud principal del afiliado. |
| `ESTADO` | Estado del afiliado (activo / inactivo). |
| `VIA_COMUNICA` | Canal de comunicación preferido (email, SMS, WhatsApp). |

---

### CIT — Citas médicas

Tabla transaccional de citas. Cada fila es una cita agendada, vigente o en historial.

| Campo | Descripción |
|---|---|
| `CONSECUTIVO` | PK. Identificador único de la cita. |
| `IDAFILIADO` | FK → AFI. Paciente de la cita. |
| `IDMEDICO` | FK → MED. Profesional de salud asignado. |
| `IDSEDE` | FK → SED. Sede donde se atiende. |
| `IDPLAN` | FK → PLN. Convenio / plan con el que se agenda. |
| `IDEMEDICA` | FK → MES. Especialidad de la cita. |
| `IDSERVICIO` | FK → SER. Servicio prestado. |
| `FECHA` | Fecha y hora de la cita. |
| `ESTADO` | Estado de la cita (activa, cancelada, etc.). |
| `CUMPLIDA` | Indicador de si la cita fue atendida. |
| `CUMPLIDA_NUM` | `1` = cita atendida. Usado para calcular `DIAS_REAGENDA`. |
| `WEB` | Bit. `1` si la cita fue agendada desde autoagenda web. |
| `DURACION` | Duración de la cita en minutos. |

---

### CITCI — Cancelaciones de citas

Archivo histórico de citas canceladas. Las cancelaciones nunca se eliminan de esta tabla.

| Campo | Descripción |
|---|---|
| `CONSECUTIVOCAN` | PK. ID de la cancelación. |
| `CONSECUTIVO` | FK → CIT. Consecutivo original de la cita cancelada. |
| `IDAFILIADO` | FK → AFI. Paciente. |
| `FECHA_ANU` | Fecha y hora de la cancelación. |
| `IDCAUSAL` | Código de causal de cancelación. |
| `RAZON` | Motivo textual de la cancelación. |
| `VALORMULTA` | Valor de multa si aplica por cancelación tardía. |

---

### MED — Profesionales de salud

Maestro de médicos, enfermeros y demás profesionales que prestan servicios en la clínica.

| Campo | Descripción |
|---|---|
| `IDMEDICO` | PK. Identificador único del profesional. |
| `NOMBRE` | Nombre completo del profesional. |
| `IDEMEDICA` | FK → MES. Especialidad principal del profesional. |
| `ESTADO` | Estado del profesional (activo / inactivo). |
| `AGENDAWEB` | Bit. `1` = el profesional aparece en autoagenda web. Solo estos se retornan en `LISTAR_MEDICOS`. |
| `TOPEPACIENTES` | Número máximo de pacientes por jornada. |
| `MINUTOSCAMBIO` | Minutos de cambio / preparación entre citas. |

---

### MES — Especialidades médicas

Catálogo de especialidades médicas disponibles. Tabla maestra sin dependencias de salida.

| Campo | Descripción |
|---|---|
| `IDEMEDICA` | PK. Identificador único de la especialidad. |
| `DESCRIPCION` | Nombre de la especialidad (ej: Medicina General, Pediatría). |
| `ESTADO` | Estado (activa / inactiva). |
| `NDIASPVEZ` | Número de días antes de repetir una cita de primera vez. |
| `NUMCIT` | Número máximo de citas simultáneas en esa especialidad. |

---

### PLN — Planes / Convenios

Planes de salud o convenios vigentes. Define el esquema de cobertura del afiliado.

| Campo | Descripción |
|---|---|
| `IDPLAN` | PK. Identificador del plan. |
| `DESCPLAN` | Nombre descriptivo del plan. |
| `IDTERCERO` | FK → TER. Aseguradora / EPS / prepagada a la que pertenece. |
| `RAZONSOCIAL` | Razón social del tercero (se obtiene vía JOIN). |
| `ESTADO` | Estado del plan. |
| `PARTICULAR` | Bool. Si `true`, el plan es particular: mostrar precio estimado al agendar. |

---

### SED — Sedes

Sedes físicas o clínicas donde se prestan los servicios.

| Campo | Descripción |
|---|---|
| `IDSEDE` | PK. Identificador único de la sede. |
| `DESCRIPCION` | Nombre descriptivo de la sede (ej: Sede Principal - Calle 123). |
| `DIRECCION` | Dirección física. |
| `CIUDAD` | Ciudad. |
| `TELEFONOS` | Teléfonos de contacto de la sede. |

---

### SER — Servicios

Catálogo de servicios clínicos que la institución vende / presta. Cada cita tiene al menos un servicio asociado.

| Campo | Descripción |
|---|---|
| `IDSERVICIO` | PK. Identificador único del servicio. |
| `DESCSERVICIO` | Nombre del servicio (ej: Consulta de Primera Vez, ECG). |
| `IDEMEDICA` | FK → MES. Especialidad a la que pertenece el servicio. |
| `DURACIONCITA` | Duración estándar en minutos. Si se seleccionan múltiples servicios, sumar las duraciones. |
| `DIAS_REAGENDA` | Días mínimos que deben pasar desde la última cita **atendida** para poder reagendar este servicio. `0` = sin restricción. |
| `ESTADO` | Estado del servicio (activo / inactivo). |
