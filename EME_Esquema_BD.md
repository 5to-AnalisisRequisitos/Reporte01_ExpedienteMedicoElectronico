# Sistema EME - Esquema de Base de Datos

**Autor:** Edson Joel Carrera Avila

---

## Resumen

El sistema EME opera con **7 tablas** en SQLite, auto-generadas la primera vez que se ejecuta el servidor. La base de datos se persiste en el archivo `eme.db` en la raíz del proyecto y nunca se expone vía HTTP (las peticiones a `/eme.db` devuelven HTTP 403).

| # | Tabla | Propósito | Registros típicos |
|---|-------|-----------|-------------------|
| 1 | `usuarios` | Cuentas con roles diferenciados | 5–50 |
| 2 | `pacientes` | Datos demográficos con eliminación lógica | 100–10.000 |
| 3 | `expedientes` | Identificador único `EXP-YYYY-NNNNN` | 100–10.000 |
| 4 | `citas` | Agenda de citas con estados | Variable |
| 5 | `consultas` | Historial clínico con signos vitales | Variable |
| 6 | `antecedentes_medicos` | Condiciones, alergias y prescripciones | Variable |
| 7 | `auditoria` | Registro inmutable de todas las acciones | Alto crecimiento |

---

## 1. Tabla `usuarios`

Almacena las cuentas del personal del consultorio con los tres roles soportados.

```sql
CREATE TABLE IF NOT EXISTS usuarios (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE NOT NULL,
  contrasena TEXT NOT NULL,
  codigo_recuperacion TEXT NOT NULL,
  nombre TEXT NOT NULL,
  apellido TEXT NOT NULL,
  rol TEXT NOT NULL,
  especialidad TEXT,
  numero_licencia TEXT UNIQUE,
  telefono TEXT,
  activo INTEGER DEFAULT 1,
  fecha_creacion DATETIME DEFAULT CURRENT_TIMESTAMP,
  ultimo_acceso DATETIME
);
```

### Campos

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | INTEGER PK | Identificador único auto-incremental |
| `email` | TEXT UNIQUE | Email del usuario (único en el sistema) |
| `contrasena` | TEXT | Hash bcrypt (10 rondas) de la contraseña |
| `codigo_recuperacion` | TEXT | Hash bcrypt del código de recuperación |
| `nombre` | TEXT | Nombre del usuario |
| `apellido` | TEXT | Apellido del usuario |
| `rol` | TEXT | Uno de: `admin`, `medico`, `recepcionista` |
| `especialidad` | TEXT | Especialidad médica (solo médicos) |
| `numero_licencia` | TEXT UNIQUE | Número de cédula profesional (obligatorio para médicos) |
| `telefono` | TEXT | Teléfono de contacto |
| `activo` | INTEGER | Bandera: 1 = activo, 0 = desactivado |
| `fecha_creacion` | DATETIME | Timestamp de creación de la cuenta |
| `ultimo_acceso` | DATETIME | Timestamp del último login exitoso |

### Reglas asociadas
- **RN05:** El email es único en toda la BD.
- **RN06:** El `numero_licencia` es único y obligatorio para `rol='medico'`.
- **RN07:** La contraseña se cifra con bcrypt (10 rondas).
- **RN08:** El código de recuperación se cifra con bcrypt y se regenera tras cada cambio de contraseña.
- **RN10:** Solo usuarios con `activo=1` pueden iniciar sesión.
- **RN26:** El primer usuario registrado debe tener `rol='admin'` (Bootstrap).
- **RN31:** No puede desactivarse al último admin activo.

---

## 2. Tabla `pacientes`

Almacena los datos demográficos del paciente con soporte para eliminación lógica.

```sql
CREATE TABLE IF NOT EXISTS pacientes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  numero_identidad TEXT UNIQUE NOT NULL,
  nombre TEXT NOT NULL,
  apellido TEXT NOT NULL,
  fecha_nacimiento DATE NOT NULL,
  sexo TEXT NOT NULL,
  telefono TEXT,
  email TEXT,
  direccion TEXT,
  ciudad TEXT,
  tipo_sangre TEXT,
  alergias TEXT,
  observaciones TEXT,
  activo INTEGER DEFAULT 1,
  fecha_eliminacion DATETIME,
  eliminado_por_id INTEGER,
  motivo_eliminacion TEXT,
  fecha_registro DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (eliminado_por_id) REFERENCES usuarios(id)
);

CREATE INDEX IF NOT EXISTS idx_pacientes_identidad ON pacientes(numero_identidad);
CREATE INDEX IF NOT EXISTS idx_pacientes_nombre ON pacientes(nombre);
CREATE INDEX IF NOT EXISTS idx_pacientes_activo ON pacientes(activo);
```

### Campos

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | INTEGER PK | Identificador único auto-incremental |
| `numero_identidad` | TEXT UNIQUE | CURP del paciente (único en el sistema) |
| `nombre` | TEXT | Nombre del paciente |
| `apellido` | TEXT | Apellido del paciente |
| `fecha_nacimiento` | DATE | Fecha de nacimiento |
| `sexo` | TEXT | M, F u O (otro) |
| `telefono` | TEXT | Teléfono de contacto |
| `email` | TEXT | Email del paciente (opcional) |
| `direccion` | TEXT | Dirección de residencia |
| `ciudad` | TEXT | Ciudad de residencia |
| `tipo_sangre` | TEXT | Grupo y factor RH (ej. O+, A-, AB+) |
| `alergias` | TEXT | Listado de alergias conocidas |
| `observaciones` | TEXT | Notas adicionales del paciente |
| `activo` | INTEGER | Bandera de soft delete: 1 = activo, 0 = eliminado |
| `fecha_eliminacion` | DATETIME | Timestamp del soft delete (null si activo) |
| `eliminado_por_id` | INTEGER FK | ID del médico que eliminó al paciente |
| `motivo_eliminacion` | TEXT | Justificación obligatoria de la eliminación |
| `fecha_registro` | DATETIME | Timestamp de creación del paciente |

### Reglas asociadas
- **RN01:** El CURP es único en toda la BD.
- **RN27:** Eliminación es de tipo lógica (Soft Delete) — `activo=0`.
- **RN32:** Solo el Admin puede restaurar (`activo=1`).
- **RN36:** Al eliminar (Soft Delete), las citas pendientes se cancelan en cascada.

---

## 3. Tabla `expedientes`

Almacena el expediente clínico abierto para cada paciente.

```sql
CREATE TABLE IF NOT EXISTS expedientes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  numero_expediente TEXT UNIQUE NOT NULL,
  paciente_id INTEGER UNIQUE NOT NULL,
  medico_responsable_id INTEGER,
  motivo_apertura TEXT NOT NULL,
  estado TEXT DEFAULT 'activo',
  fecha_apertura DATETIME DEFAULT CURRENT_TIMESTAMP,
  abierto_por_id INTEGER NOT NULL,
  notas_apertura TEXT,
  FOREIGN KEY (paciente_id) REFERENCES pacientes(id),
  FOREIGN KEY (medico_responsable_id) REFERENCES usuarios(id),
  FOREIGN KEY (abierto_por_id) REFERENCES usuarios(id)
);

CREATE INDEX IF NOT EXISTS idx_expedientes_numero ON expedientes(numero_expediente);
CREATE INDEX IF NOT EXISTS idx_expedientes_paciente ON expedientes(paciente_id);
```

### Campos

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | INTEGER PK | Identificador único auto-incremental |
| `numero_expediente` | TEXT UNIQUE | Formato `EXP-YYYY-NNNNN` (auto-generado) |
| `paciente_id` | INTEGER UNIQUE FK | Paciente asociado (1:1) |
| `medico_responsable_id` | INTEGER FK | Médico asignado como responsable |
| `motivo_apertura` | TEXT | Razón de apertura del expediente |
| `estado` | TEXT | `activo` o `archivado` |
| `fecha_apertura` | DATETIME | Timestamp de apertura |
| `abierto_por_id` | INTEGER FK | Usuario que abrió el expediente |
| `notas_apertura` | TEXT | Notas adicionales del médico al abrir |

### Reglas asociadas
- **RN02:** Cada expediente tiene número único `EXP-YYYY-NNNNN`.
- **RN03:** No se puede crear un nuevo expediente si la CURP ya existe.
- **RN04:** Un paciente puede existir sin expediente, pero no al revés.
- **RN14:** La fecha de apertura es automática e inalterable.
- **Cascada de eliminación lógica:** cuando el paciente se elimina, el expediente pasa a `estado='archivado'`.

---

## 4. Tabla `citas`

Almacena la agenda de citas médicas.

```sql
CREATE TABLE IF NOT EXISTS citas (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  paciente_id INTEGER NOT NULL,
  medico_id INTEGER NOT NULL,
  fecha_cita DATE NOT NULL,
  hora_cita TIME NOT NULL,
  estado TEXT DEFAULT 'pendiente',
  motivo_consulta TEXT,
  notas TEXT,
  agendada_por_id INTEGER,
  fecha_creacion DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (paciente_id) REFERENCES pacientes(id),
  FOREIGN KEY (medico_id) REFERENCES usuarios(id),
  FOREIGN KEY (agendada_por_id) REFERENCES usuarios(id)
);

CREATE INDEX IF NOT EXISTS idx_citas_fecha ON citas(fecha_cita);
CREATE INDEX IF NOT EXISTS idx_citas_paciente ON citas(paciente_id);
CREATE INDEX IF NOT EXISTS idx_citas_medico ON citas(medico_id);
CREATE INDEX IF NOT EXISTS idx_citas_estado ON citas(estado);
```

### Campos

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | INTEGER PK | Identificador único auto-incremental |
| `paciente_id` | INTEGER FK | Paciente que tendrá la cita |
| `medico_id` | INTEGER FK | Médico asignado para atender |
| `fecha_cita` | DATE | Fecha de la cita |
| `hora_cita` | TIME | Hora de la cita |
| `estado` | TEXT | `pendiente`, `completada` o `cancelada` |
| `motivo_consulta` | TEXT | Motivo declarado |
| `notas` | TEXT | Notas adicionales |
| `agendada_por_id` | INTEGER FK | Usuario que agendó la cita |
| `fecha_creacion` | DATETIME | Timestamp del agendamiento |

### Reglas asociadas
- **RN17:** Cuando se registra una consulta asociada, la cita pasa a `completada`.
- **RN18:** Estados válidos: `pendiente`, `completada`, `cancelada`.
- **RN35:** Las citas se agendan desde el detalle del paciente.
- **RN36:** Al eliminar el paciente, sus citas pendientes pasan a `cancelada`.

---

## 5. Tabla `consultas`

Almacena el registro de cada consulta médica con signos vitales y diagnóstico.

```sql
CREATE TABLE IF NOT EXISTS consultas (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  cita_id INTEGER,
  paciente_id INTEGER NOT NULL,
  medico_id INTEGER NOT NULL,
  fecha_consulta DATETIME DEFAULT CURRENT_TIMESTAMP,
  peso REAL,
  altura REAL,
  presion_arterial TEXT,
  temperatura REAL,
  frecuencia_cardiaca INTEGER,
  motivo_consulta TEXT NOT NULL,
  sintomas TEXT NOT NULL,
  diagnostico TEXT NOT NULL,
  plan_tratamiento TEXT,
  observaciones TEXT,
  fecha_proximo_control DATE,
  FOREIGN KEY (cita_id) REFERENCES citas(id),
  FOREIGN KEY (paciente_id) REFERENCES pacientes(id),
  FOREIGN KEY (medico_id) REFERENCES usuarios(id)
);

CREATE INDEX IF NOT EXISTS idx_consultas_paciente ON consultas(paciente_id);
CREATE INDEX IF NOT EXISTS idx_consultas_medico ON consultas(medico_id);
CREATE INDEX IF NOT EXISTS idx_consultas_fecha ON consultas(fecha_consulta);
```

### Campos

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | INTEGER PK | Identificador único auto-incremental |
| `cita_id` | INTEGER FK | Cita asociada (opcional) |
| `paciente_id` | INTEGER FK | Paciente atendido |
| `medico_id` | INTEGER FK | Médico que atendió |
| `fecha_consulta` | DATETIME | Timestamp de la consulta |
| `peso` | REAL | Peso en kg |
| `altura` | REAL | Altura en cm |
| `presion_arterial` | TEXT | Presión arterial (ej. "120/80") |
| `temperatura` | REAL | Temperatura corporal en °C |
| `frecuencia_cardiaca` | INTEGER | Frecuencia cardiaca en bpm |
| `motivo_consulta` | TEXT | Motivo de la consulta (obligatorio) |
| `sintomas` | TEXT | Síntomas reportados (obligatorio) |
| `diagnostico` | TEXT | Diagnóstico del médico (obligatorio) |
| `plan_tratamiento` | TEXT | Plan de tratamiento |
| `observaciones` | TEXT | Notas adicionales |
| `fecha_proximo_control` | DATE | Próxima cita sugerida |

### Reglas asociadas
- **RN15:** Solo médicos pueden registrar consultas.
- **RN16:** Las recepcionistas no pueden acceder a este recurso (HTTP 403).
- **RN17:** Si `cita_id` no es null, esa cita pasa a `completada`.

---

## 6. Tabla `antecedentes_medicos`

Almacena las condiciones crónicas, alergias y prescripciones asociadas al paciente.

```sql
CREATE TABLE IF NOT EXISTS antecedentes_medicos (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  paciente_id INTEGER NOT NULL,
  consulta_id INTEGER,
  tipo TEXT NOT NULL,
  condicion TEXT,
  medicamento_nombre TEXT,
  dosis TEXT,
  cantidad INTEGER,
  duracion_dias INTEGER,
  estado TEXT DEFAULT 'activo',
  notas TEXT,
  fecha_registro DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (paciente_id) REFERENCES pacientes(id),
  FOREIGN KEY (consulta_id) REFERENCES consultas(id)
);

CREATE INDEX IF NOT EXISTS idx_antecedentes_paciente ON antecedentes_medicos(paciente_id);
CREATE INDEX IF NOT EXISTS idx_antecedentes_tipo ON antecedentes_medicos(tipo);
```

### Campos

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | INTEGER PK | Identificador único auto-incremental |
| `paciente_id` | INTEGER FK | Paciente al que pertenece |
| `consulta_id` | INTEGER FK | Consulta de origen (si aplica) |
| `tipo` | TEXT | `condicion`, `prescripcion` o `alergia` |
| `condicion` | TEXT | Descripción de la condición (si tipo=`condicion` o `alergia`) |
| `medicamento_nombre` | TEXT | Nombre del medicamento (si tipo=`prescripcion`) |
| `dosis` | TEXT | Dosis prescrita (ej. "500 mg cada 8h") |
| `cantidad` | INTEGER | Cantidad de unidades |
| `duracion_dias` | INTEGER | Duración del tratamiento en días |
| `estado` | TEXT | `activo` o `inactivo` |
| `notas` | TEXT | Notas adicionales |
| `fecha_registro` | DATETIME | Timestamp del registro |

### Reglas asociadas
- **RN15:** Solo médicos pueden registrar antecedentes.
- **RN19:** Las prescripciones se registran automáticamente como antecedentes de tipo `prescripcion` cuando se completa una consulta con medicamentos.

---

## 7. Tabla `auditoria`

Registro inmutable de todas las acciones del sistema.

```sql
CREATE TABLE IF NOT EXISTS auditoria (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  usuario_id INTEGER,
  usuario_email TEXT NOT NULL,
  usuario_rol TEXT NOT NULL,
  accion TEXT NOT NULL,
  entidad TEXT NOT NULL,
  entidad_id INTEGER,
  descripcion TEXT,
  datos_antes TEXT,
  datos_despues TEXT,
  ip_origen TEXT,
  user_agent TEXT,
  fecha_accion DATETIME DEFAULT CURRENT_TIMESTAMP,
  exitoso INTEGER DEFAULT 1,
  mensaje_error TEXT,
  FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

CREATE INDEX IF NOT EXISTS idx_auditoria_usuario ON auditoria(usuario_id);
CREATE INDEX IF NOT EXISTS idx_auditoria_fecha ON auditoria(fecha_accion);
CREATE INDEX IF NOT EXISTS idx_auditoria_accion ON auditoria(accion);
CREATE INDEX IF NOT EXISTS idx_auditoria_entidad ON auditoria(entidad);
```

### Campos

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | INTEGER PK | Identificador único auto-incremental |
| `usuario_id` | INTEGER FK | ID del usuario que realizó la acción (null si no logueado) |
| `usuario_email` | TEXT | Email del usuario (snapshot al momento de la acción) |
| `usuario_rol` | TEXT | Rol del usuario al momento de la acción |
| `accion` | TEXT | Tipo de acción: `LOGIN`, `LOGOUT`, `LOGIN_FAILED`, `ACCESS_DENIED`, `CREATE`, `UPDATE`, `DELETE`, `RESTORE`, etc. |
| `entidad` | TEXT | Tabla afectada (ej. `pacientes`, `usuarios`, `citas`) |
| `entidad_id` | INTEGER | ID del registro afectado |
| `descripcion` | TEXT | Descripción legible de la acción |
| `datos_antes` | TEXT (JSON) | Snapshot del registro antes del cambio (para UPDATE/DELETE) |
| `datos_despues` | TEXT (JSON) | Snapshot del registro después del cambio (para CREATE/UPDATE) |
| `ip_origen` | TEXT | Dirección IP del cliente |
| `user_agent` | TEXT | User-Agent del navegador |
| `fecha_accion` | DATETIME | Timestamp de la acción |
| `exitoso` | INTEGER | 1 si la acción tuvo éxito, 0 si falló |
| `mensaje_error` | TEXT | Detalle del error si `exitoso=0` |

### Tipos de acción registrados

| Acción | Cuándo se registra |
|--------|-------------------|
| `LOGIN` | Inicio de sesión exitoso |
| `LOGIN_FAILED` | Intento de login con credenciales inválidas |
| `LOGOUT` | Cierre de sesión (manual o automático) |
| `BOOTSTRAP_ADMIN` | Creación del primer Administrador del sistema |
| `ACCESS_DENIED` | Acceso a recurso sin rol autorizado |
| `CREATE` | Creación de cualquier entidad |
| `UPDATE` | Actualización de cualquier entidad |
| `DELETE` | Eliminación (lógica) de cualquier entidad |
| `RESTORE` | Restauración de un paciente eliminado |
| `PASSWORD_RESET` | Cambio de contraseña |
| `USER_ACTIVATED` | Activación de cuenta |
| `USER_DEACTIVATED` | Desactivación de cuenta |

### Reglas asociadas
- **RN20:** Cada acción modificadora se registra con todos los campos.
- **RN21:** Los registros son inmutables (no se modifican ni eliminan).
- **RN22:** Los intentos fallidos de login generan `LOGIN_FAILED`.
- **RN23:** Los accesos sin autorización generan `ACCESS_DENIED`.
- **RN24:** Cada usuario solo ve su propia auditoría; el admin ve la global.

---

## Diagrama Entidad-Relación

![Diagrama del proyecto](img/Entidad_Relacion.png)

---

## Inicialización de la base de datos

La base de datos se crea automáticamente cuando se ejecuta `npm start`. El servidor:

1. Ejecuta `setup.js` para generar el `.env` con un `JWT_SECRET` aleatorio (si no existe).
2. Carga sql.js y verifica si existe `eme.db`.
3. Si no existe, crea las 7 tablas y los índices.
4. Si existe, carga el archivo y mantiene los datos previos.

### Bootstrap del primer Administrador

El sistema detecta automáticamente si no hay administradores activos:

```js
GET /api/auth/setup-status
// → { tieneUsuarios: false, tieneAdmin: false, requiereBootstrap: true }
```

Mientras `requiereBootstrap=true`, el endpoint `POST /api/auth/registro` acepta la creación del primer administrador sin autenticación previa. Una vez creado, este endpoint requiere autenticación como admin.
