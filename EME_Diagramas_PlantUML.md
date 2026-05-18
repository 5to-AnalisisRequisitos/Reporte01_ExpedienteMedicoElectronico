# Sistema EME - Diagramas Formales

**Autor:** Edson Joel Carrera Avila

Pega en: https://www.plantuml.com/plantuml/uml/

---

## 1. Diagrama de Casos de Uso

```
@startuml CasosDeUso_EME_v3
!theme plain
title Sistema EME v3.0 — Casos de Uso
left to right direction

actor "Administrador" as admin #LightYellow
actor "Medico" as medico
actor "Recepcionista" as recepcion

rectangle "Sistema EME" {
  package "Autenticacion" {
    usecase "CU01\nBootstrap Admin inicial" as CU01
    usecase "CU02\nIniciar sesion" as CU02
    usecase "CU03\nCerrar sesion" as CU03
    usecase "CU14\nRecuperar contrasena" as CU14
  }
  package "Gestion Usuarios (solo Admin)" {
    usecase "CU04\nGestionar usuarios\n(crear/editar/desactivar)" as CU04
  }
  package "Gestion Pacientes" {
    usecase "CU05\nApertura de expediente" as CU05
    usecase "CU06\nBusqueda inteligente" as CU06
    usecase "CU07\nEditar paciente (con motivo)" as CU07
    usecase "CU08\nEliminacion logica (Soft Delete)" as CU08
    usecase "CU09\nRestaurar paciente" as CU09
  }
  package "Citas" {
    usecase "CU10\nAgendar cita desde paciente" as CU10
    usecase "Cancelar cita" as CC
    usecase "Ver agenda" as VA
  }
  package "Clinico (solo Medico)" {
    usecase "CU11\nRegistrar consulta" as CU11
    usecase "Registrar antecedentes" as CA
    usecase "Ver historial clinico" as VHC
  }
  package "Auditoria" {
    usecase "CU12\nVer mi actividad" as CU12
    usecase "CU13\nAuditoria global (Admin)" as CU13
  }
}

' Administrador
admin --> CU01
admin --> CU02
admin --> CU03
admin --> CU14
admin --> CU04
admin --> CU09
admin --> CU12
admin --> CU13

' Medico
medico --> CU02
medico --> CU03
medico --> CU14
medico --> CU05
medico --> CU06
medico --> CU07
medico --> CU08
medico --> CU10
medico --> CC
medico --> VA
medico --> CU11
medico --> CA
medico --> VHC
medico --> CU12

' Recepcionista
recepcion --> CU02
recepcion --> CU03
recepcion --> CU14
recepcion --> CU05
recepcion --> CU06
recepcion --> CU07
recepcion --> CU10
recepcion --> CC
recepcion --> VA
recepcion --> CU12

note right of CU08 : Solo Medico
note right of CU09 : Solo Admin
note right of CU04 : Solo Admin
note right of CU13 : Solo Admin
note right of CU01 : Solo si no existe Admin
@enduml
```

---

## 2. Diagrama de Secuencia — Nuevo Paciente y Agendamiento

```
@startuml Secuencia_NuevoPaciente_v3
!theme plain
title EME v3.0 — Secuencia: Nuevo Paciente y Agendamiento

actor "Recepcionista/Medico" as usuario
participant "Frontend\n(Pacientes)" as fe
participant "Backend\n(server.js)" as be
database "SQLite\n(eme.db)" as db
participant "Auditoria" as audit

== Busqueda previa ==
usuario -> fe : Escribe "Juan" en buscador
fe -> be : GET /api/pacientes/buscar/Juan
be -> db : SELECT * FROM pacientes\nWHERE activo=1 AND (nombre LIKE OR apellido LIKE\nOR numero_identidad LIKE OR telefono LIKE)
db --> be : [] (sin resultados)
be --> fe : []
fe --> usuario : Sin resultados + boton "+ Nuevo paciente"

== Verificacion previa por CURP ==
usuario -> fe : Click "+ Nuevo paciente"
fe --> usuario : Abre modal con formulario
usuario -> fe : Llena CURP, nombre, fecha_nac, sexo, motivo
fe -> be : GET /api/pacientes/verificar/CURP
be -> db : SELECT * FROM pacientes\nWHERE numero_identidad = ?
db --> be : null (no existe)
be --> fe : {existe: false}

== Creacion de paciente y expediente ==
fe -> be : POST /api/pacientes/apertura-expediente
be -> be : Validar campos
be -> db : INSERT INTO pacientes (...)
db --> be : paciente_id = 1
be -> db : INSERT INTO expedientes\n(EXP-2026-00001, paciente_id, motivo_apertura, ...)
db --> be : expediente_id = 1
be -> audit : CREATE pacientes + CREATE expedientes\n(con datos_despues en JSON)
be --> fe : { paciente_id, expediente_id, numero_expediente }
fe --> usuario : Toast "Paciente creado: EXP-2026-00001"

== Agendamiento desde detalle del paciente ==
usuario -> fe : Click "Agendar cita" en detalle del paciente
fe -> be : GET /api/medicos
be --> fe : { medicos activos }
fe --> usuario : Modal con paciente preseleccionado\n+ fecha, hora, medico, motivo
usuario -> fe : Selecciona fecha 20/05, hora 10:00, medico
fe -> be : POST /api/citas\n{ paciente_id:1, medico_id:2, ... }
be -> db : INSERT INTO citas (...)
be -> audit : CREATE citas
be --> fe : { id, message: "Cita agendada" }
fe --> usuario : Toast "Cita agendada para Juan el 20/05 a las 10:00"
@enduml
```

---

## 3. Diagrama de Actividad — Autenticación con Seguridad

```
@startuml Actividad_Auth_v3
!theme plain
title EME v3.0 — Autenticacion y Gestion de Sesion

start
:Abrir http://localhost:3000;
if (sessionStorage tiene token valido?) then (si)
  :Restaurar sesion;
  :Iniciar watcher inactividad 15 min;
  stop
else (no)
endif
if (Existe Admin en el sistema?) then (no)
  :Pantalla Bootstrap;
  :Crear Administrador inicial;
  :Generar codigo de recuperacion;
  :Mostrar codigo una unica vez;
else (si)
  :Pantalla de Login;
endif
:Ingresar email + contrasena;
:Rate limit: max 5 intentos / 15 min por IP;
if (Limite excedido?) then (si)
  #Pink:HTTP 429: "Demasiados intentos.\nEspera 15 min";
  :Registrar en auditoria;
  stop
endif
:Validar credenciales en BD (bcrypt compare);
if (Credenciales validas?) then (no)
  :Registrar LOGIN_FAILED en auditoria;
  #Pink:Mensaje generico:\n"Credenciales invalidas";
  stop
endif
if (Usuario activo?) then (no)
  :Registrar LOGIN_FAILED en auditoria;
  #Pink:Mismo mensaje generico;
  stop
endif
:Generar JWT con expiracion 24h;
:Guardar en sessionStorage\n(NO localStorage);
:Registrar LOGIN en auditoria;
:Mostrar Dashboard segun rol\n(Admin, Medico o Recepcionista);
:Iniciar watcher inactividad;
repeat
  :Usuario interactua;
  :Resetear timer 15 min;
repeat while (Inactivo 14+ min?) is (no)
:Mostrar modal advertencia\ncon cuenta regresiva 60 segundos;
if (Click "Seguir conectado"?) then (si)
  :Resetear timer;
else (no - tiempo agotado)
endif
:Limpiar sessionStorage;
:Registrar LOGOUT en auditoria;
:Redirigir a Login;
#LightYellow:Mensaje: "Sesion cerrada por inactividad";
stop
@enduml
```

---

## 4. Diagrama de Componentes

```
@startuml Componentes_v3
!theme plain
title EME v3.0 — Componentes

package "Navegador" {
  component [index.html\nSPA] as html
  component [app.js\nLogica cliente] as appjs
  component [styles.css] as css
  component [sessionStorage\nToken JWT temporal] as ss
  component [Watcher inactividad\n15 min -> logout] as w
  appjs --> html
  appjs --> ss
  appjs --> w
}

package "Capa de Seguridad" {
  component [Helmet.js\nCabeceras HTTP] as helmet
  component [CORS restringido\nlocalhost:3000] as cors
  component [Rate Limiter\n5 intentos/15 min] as rl
  component [authMiddleware\nVerifica JWT] as authmw
  component [requireRole()\nRBAC] as rbac
  component [dotenv\nJWT_SECRET] as dotenv
}

package "Backend Express (32 endpoints)" {
  component [Auth /api/auth/*\n(5 endpoints)] as auth
  component [Pacientes /api/pacientes/*\n(8 endpoints)] as pac
  component [Citas /api/citas/*\n(5 endpoints)] as cit
  component [Consultas /api/consultas/*\n(2 endpoints)] as con
  component [Antecedentes /api/antecedentes\n(1 endpoint)] as ant
  component [Admin /api/admin/*\n(6 endpoints)] as adm
  component [Auditoria /api/auditoria/*\n(2 endpoints)] as aud
  component [Utilidades /api/medicos,\n/dashboard/stats, /usuario/info\n(3 endpoints)] as util
  component [registrarAuditoria()\nFuncion transversal] as afn
}

database "eme.db (SQLite)\n7 tablas" {
  component [usuarios] as tu
  component [pacientes] as tp
  component [expedientes] as te
  component [citas] as tc
  component [consultas] as tco
  component [antecedentes_medicos] as ta
  component [auditoria] as taud
}

package "Auto-configuracion" {
  component [setup.js\nGenera .env automatico] as setup
  component [iniciar.sh / iniciar.bat\nScripts multiplataforma] as iniciar
}

appjs --> cors : HTTP
cors --> helmet
helmet --> rl
rl --> auth
authmw --> rbac
dotenv --> auth
auth --> tu
pac --> tp
pac --> te
cit --> tc
con --> tco
ant --> ta
adm --> tu
adm --> tp
aud --> taud
util --> tu
afn --> taud
setup --> dotenv : Genera JWT_SECRET
iniciar --> setup : npm start

auth ..> afn
pac ..> afn
cit ..> afn
con ..> afn
ant ..> afn
adm ..> afn
@enduml
```

---

## 5. Diagrama de Clases

```
@startuml Clases_v3
!theme plain
title EME v3.0 — Diagrama de Clases

class Usuario {
  +id: INTEGER
  +email: STRING UNIQUE
  -contrasena: STRING (bcrypt)
  -codigo_recuperacion: STRING (bcrypt)
  +nombre: STRING
  +apellido: STRING
  +rol: ENUM(admin|medico|recepcionista)
  +especialidad: STRING
  +numero_licencia: STRING UNIQUE
  +telefono: STRING
  +activo: BOOLEAN
  +fecha_creacion: DATETIME
  +ultimo_acceso: DATETIME
  --
  +login(): JWT
  +logout(): void
  +recuperarPassword(codigo): void
  +cambiarRol(): void [admin]
  +desactivar(): void [admin]
}

class Paciente {
  +id: INTEGER
  +numero_identidad: STRING UNIQUE (CURP)
  +nombre: STRING
  +apellido: STRING
  +fecha_nacimiento: DATE
  +sexo: ENUM(M|F|O)
  +telefono: STRING
  +email: STRING
  +direccion: STRING
  +ciudad: STRING
  +tipo_sangre: STRING
  +alergias: TEXT
  +observaciones: TEXT
  +activo: BOOLEAN
  +fecha_eliminacion: DATETIME
  +eliminado_por_id: INTEGER
  +motivo_eliminacion: TEXT
  +fecha_registro: DATETIME
  --
  +crear(): void
  +editar(motivo): void
  +eliminar(motivo): void [medico]
  +restaurar(): void [admin]
  +buscar(termino): List
}

class Expediente {
  +id: INTEGER
  +numero_expediente: STRING UNIQUE (EXP-YYYY-NNNNN)
  +paciente_id: INTEGER UNIQUE
  +medico_responsable_id: INTEGER
  +motivo_apertura: TEXT
  +estado: ENUM(activo|archivado)
  +fecha_apertura: DATETIME
  +abierto_por_id: INTEGER
  +notas_apertura: TEXT
  --
  +abrir(): void
  +archivar(): void
  +reactivar(): void [admin]
}

class Cita {
  +id: INTEGER
  +paciente_id: INTEGER
  +medico_id: INTEGER
  +fecha_cita: DATE
  +hora_cita: TIME
  +estado: ENUM(pendiente|completada|cancelada)
  +motivo_consulta: TEXT
  +notas: TEXT
  +agendada_por_id: INTEGER
  +fecha_creacion: DATETIME
  --
  +agendar(): void
  +cancelar(): void
  +completar(): void
}

class Consulta {
  +id: INTEGER
  +cita_id: INTEGER
  +paciente_id: INTEGER
  +medico_id: INTEGER
  +fecha_consulta: DATETIME
  +peso: REAL
  +altura: REAL
  +presion_arterial: STRING
  +temperatura: REAL
  +frecuencia_cardiaca: INTEGER
  +motivo_consulta: TEXT
  +sintomas: TEXT
  +diagnostico: TEXT
  +plan_tratamiento: TEXT
  +observaciones: TEXT
  +fecha_proximo_control: DATE
  --
  +registrar(): void [medico]
}

class AntecedenteMedico {
  +id: INTEGER
  +paciente_id: INTEGER
  +consulta_id: INTEGER
  +tipo: ENUM(condicion|prescripcion|alergia)
  +condicion: TEXT
  +medicamento_nombre: STRING
  +dosis: STRING
  +cantidad: INTEGER
  +duracion_dias: INTEGER
  +estado: ENUM(activo|inactivo)
  +notas: TEXT
  +fecha_registro: DATETIME
  --
  +registrar(): void [medico]
}

class RegistroAuditoria {
  +id: INTEGER
  +usuario_id: INTEGER
  +usuario_email: STRING
  +usuario_rol: STRING
  +accion: STRING
  +entidad: STRING
  +entidad_id: INTEGER
  +descripcion: TEXT
  +datos_antes: JSON
  +datos_despues: JSON
  +ip_origen: STRING
  +user_agent: STRING
  +fecha_accion: DATETIME
  +exitoso: BOOLEAN
  +mensaje_error: TEXT
}

Usuario "1" -- "0..*" Expediente : abre
Usuario "1" -- "0..*" Cita : agenda/atiende
Usuario "1" -- "0..*" Consulta : registra
Usuario "1" -- "0..*" RegistroAuditoria : genera
Usuario "1" -- "0..*" Paciente : elimina
Paciente "1" -- "1" Expediente : tiene
Paciente "1" -- "0..*" Cita : tiene
Paciente "1" -- "0..*" Consulta : recibe
Paciente "1" -- "0..*" AntecedenteMedico : tiene
Cita "1" -- "0..*" Consulta : origina
Consulta "1" -- "0..*" AntecedenteMedico : prescribe
@enduml
```

---

## 6. Diagrama Entidad-Relación

```
@startuml ER_v3
!theme plain
title EME v3.0 — Entidad-Relacion (7 tablas)

entity "usuarios" {
  *id : INTEGER <<PK>>
  --
  email : TEXT UNIQUE
  contrasena : TEXT (bcrypt)
  codigo_recuperacion : TEXT (bcrypt)
  nombre : TEXT
  apellido : TEXT
  rol : TEXT (admin|medico|recepcionista)
  especialidad : TEXT
  numero_licencia : TEXT UNIQUE
  telefono : TEXT
  activo : INTEGER (default 1)
  fecha_creacion : DATETIME
  ultimo_acceso : DATETIME
}

entity "pacientes" {
  *id : INTEGER <<PK>>
  --
  numero_identidad : TEXT UNIQUE (CURP)
  nombre : TEXT
  apellido : TEXT
  fecha_nacimiento : DATE
  sexo : TEXT
  telefono : TEXT
  email : TEXT
  direccion : TEXT
  ciudad : TEXT
  tipo_sangre : TEXT
  alergias : TEXT
  observaciones : TEXT
  activo : INTEGER (default 1)
  fecha_eliminacion : DATETIME
  eliminado_por_id : INTEGER <<FK>>
  motivo_eliminacion : TEXT
  fecha_registro : DATETIME
}

entity "expedientes" {
  *id : INTEGER <<PK>>
  --
  numero_expediente : TEXT UNIQUE
  paciente_id : INTEGER UNIQUE <<FK>>
  medico_responsable_id : INTEGER <<FK>>
  motivo_apertura : TEXT
  estado : TEXT (activo|archivado)
  fecha_apertura : DATETIME
  abierto_por_id : INTEGER <<FK>>
  notas_apertura : TEXT
}

entity "citas" {
  *id : INTEGER <<PK>>
  --
  paciente_id : INTEGER <<FK>>
  medico_id : INTEGER <<FK>>
  fecha_cita : DATE
  hora_cita : TIME
  estado : TEXT (pendiente|completada|cancelada)
  motivo_consulta : TEXT
  notas : TEXT
  agendada_por_id : INTEGER <<FK>>
  fecha_creacion : DATETIME
}

entity "consultas" {
  *id : INTEGER <<PK>>
  --
  cita_id : INTEGER <<FK>>
  paciente_id : INTEGER <<FK>>
  medico_id : INTEGER <<FK>>
  fecha_consulta : DATETIME
  peso : REAL
  altura : REAL
  presion_arterial : TEXT
  temperatura : REAL
  frecuencia_cardiaca : INTEGER
  motivo_consulta : TEXT
  sintomas : TEXT
  diagnostico : TEXT
  plan_tratamiento : TEXT
  observaciones : TEXT
  fecha_proximo_control : DATE
}

entity "antecedentes_medicos" {
  *id : INTEGER <<PK>>
  --
  paciente_id : INTEGER <<FK>>
  consulta_id : INTEGER <<FK>>
  tipo : TEXT (condicion|prescripcion|alergia)
  condicion : TEXT
  medicamento_nombre : TEXT
  dosis : TEXT
  cantidad : INTEGER
  duracion_dias : INTEGER
  estado : TEXT (activo|inactivo)
  notas : TEXT
  fecha_registro : DATETIME
}

entity "auditoria" {
  *id : INTEGER <<PK>>
  --
  usuario_id : INTEGER <<FK>>
  usuario_email : TEXT
  usuario_rol : TEXT
  accion : TEXT
  entidad : TEXT
  entidad_id : INTEGER
  descripcion : TEXT
  datos_antes : TEXT (JSON)
  datos_despues : TEXT (JSON)
  ip_origen : TEXT
  user_agent : TEXT
  fecha_accion : DATETIME
  exitoso : INTEGER (default 1)
  mensaje_error : TEXT
}

usuarios ||--o{ expedientes : "abre / responsable"
usuarios ||--o{ citas : "atiende / agenda"
usuarios ||--o{ consultas : "registra"
usuarios ||--o{ auditoria : "genera"
usuarios ||--o{ pacientes : "elimina"
pacientes ||--|| expedientes : "tiene"
pacientes ||--o{ citas : "tiene"
pacientes ||--o{ consultas : "recibe"
pacientes ||--o{ antecedentes_medicos : "tiene"
citas ||--o{ consultas : "origina"
consultas ||--o{ antecedentes_medicos : "prescribe"
@enduml
```

---

## 7. Diagrama de Estados — Cita

```
@startuml Estados_Cita_v3
!theme plain
title EME v3.0 — Estados de Cita Medica

[*] --> Pendiente : Medico o recepcionista\nagenda desde el detalle\ndel paciente\n(paciente preseleccionado)

Pendiente --> Completada : Medico registra\nconsulta medica\nasociada a la cita
Pendiente --> Cancelada : Medico/recepcionista\ncancela manualmente
Pendiente --> Cancelada : Soft-delete del paciente\n(cascada automatica)

Cancelada --> [*] : Estado final
Completada --> [*] : Genera registro\nen consultas + antecedentes

note right of Pendiente
  Cita creada desde modal
  "Agendar cita" con paciente
  preseleccionado.
  Fecha minima = hoy.
end note

note right of Completada
  Genera consulta con:
  - Signos vitales
  - Diagnostico
  - Plan tratamiento
  - Prescripciones (antecedentes)
end note

note right of Cancelada
  Cascade automatico: al eliminar
  un paciente (Soft Delete), todas
  sus citas en estado pendiente
  se cancelan automaticamente.
end note
@enduml
```

---

## 8. Diagrama de Despliegue

```
@startuml Despliegue_v3
!theme plain
title EME v3.0 — Despliegue

node "Computadora del Consultorio" {

  node "Navegador Web" {
    component "Frontend SPA\n(HTML + CSS + JS)" as fe
    component "sessionStorage\nToken JWT temporal\n(se destruye al cerrar)" as ss
    component "Watcher inactividad\n15 min -> logout\nAviso 60 seg" as w
  }

  node "Node.js + Express\n(server.js)" {
    component "Helmet.js\nCabeceras HTTP" as hel
    component "CORS\nlocalhost:3000" as cors
    component "Rate Limiter\n5 intentos / 15 min" as rl
    component "authMiddleware\nJWT + RBAC" as mw
    component "API REST\n32 endpoints" as api
    component "registrarAuditoria()\nTransversal" as af
  }

  node "SQLite (sql.js)" {
    database "eme.db\n7 tablas" as db
  }

  node "Auto-configuracion" {
    component "setup.js\nGenera .env si no existe" as setup
    component "iniciar.sh / iniciar.bat\nScripts multiplataforma" as ini
  }

  file ".env\nJWT_SECRET\nPORT\nINACTIVITY_TIMEOUT_MINUTES\nMAX_LOGIN_ATTEMPTS\n(no en Git)" as env
}

fe --> cors : HTTP localhost:3000
fe --> ss
fe --> w
cors --> hel
hel --> rl
rl --> mw
mw --> api
api --> db
api ..> af
af --> db
env --> mw : JWT_SECRET
setup --> env : Genera
ini --> setup : npm start

note bottom of db
  Pacientes eliminados
  permanecen con activo=0.
  Acceso directo por URL
  bloqueado (HTTP 403).
end note

note right of env
  Generado automaticamente
  por setup.js con un
  JWT_SECRET aleatorio de
  64 caracteres unico por
  instalacion.
end note

note left of ss
  No persiste entre sesiones
  del navegador. Se destruye
  al cerrar pestana/ventana.
end note
@enduml
```
