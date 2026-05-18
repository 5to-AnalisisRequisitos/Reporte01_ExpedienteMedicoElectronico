# Sistema EME - Especificación de Requisitos de Software

**Autor:** Edson Joel Carrera Avila

---

## Tabla de Contenidos

1. Introducción
2. Descripción General
3. Requisitos Específicos
4. Apéndices

---

## 1. Introducción

### 1.1. Propósito

Definir formalmente los requisitos del **Sistema de Expediente Médico Electrónico (EME)**, aplicación web para gestionar información clínica y administrativa de un consultorio médico pequeño en México.

### 1.2. Ámbito

**EME** — Expediente Médico Electrónico

**Lo que el sistema HARÁ:**
- Gestionar usuarios con tres roles: Administrador, Médico y Recepcionista.
- Registrar y consultar expedientes médicos de pacientes.
- Gestionar citas médicas mediante flujo centrado en el paciente.
- Registrar consultas con signos vitales, diagnóstico y plan de tratamiento.
- Registrar antecedentes médicos y prescripciones de medicamentos.
- Auditar todas las acciones del sistema con trazabilidad completa.
- Implementar medidas de ciberseguridad: rate limiting, sesión temporal en sessionStorage, timeout de inactividad, Helmet, CORS restringido, .env auto-generado.

**Lo que el sistema NO HARÁ:**
- Conectarse a sistemas externos de salud.
- Gestionar facturación ni seguros médicos.
- Generar recetas impresas con validez legal.
- Operar en múltiples sucursales simultáneamente.

### 1.3. Definiciones

| Término | Definición |
|---------|-----------|
| **EME** | Expediente Médico Electrónico |
| **CURP** | Clave Única de Registro de Población (identificador oficial en México) |
| **RBAC** | Role-Based Access Control — control de acceso basado en roles |
| **JWT** | JSON Web Token — mecanismo de autenticación sin estado |
| **Soft delete** | Eliminación lógica: el registro queda con `activo=0`, sin borrado físico |
| **Bootstrap** | Proceso de configuración inicial del sistema (creación del primer admin) |
| **Rate limiting** | Restricción de intentos de autenticación por unidad de tiempo |
| **sessionStorage** | Almacenamiento temporal de navegador que se borra al cerrar la ventana |
| **Hub de pacientes** | Pestaña central desde donde se realizan todas las operaciones sobre pacientes |
| **Timeout de inactividad** | Cierre automático de sesión tras 15 minutos sin actividad del usuario |
| **setup.js** | Script Node.js que auto-genera el archivo `.env` con JWT_SECRET aleatorio en cada instalación |

### 1.4. Referencias

1. IEEE 830-1998 — Recommended Practice for Software Requirements Specifications
2. Caso de uso: Apertura de Expediente Médico (documento base del proyecto)
3. NOM-024-SSA3-2012 — Expediente clínico electrónico (referencia normativa mexicana)
4. OWASP Top 10:2021 — Estándar de mejores prácticas de seguridad web

### 1.5. Visión General

Este documento se organiza en tres secciones: Introducción (contexto), Descripción General (perspectiva del sistema) y Requisitos Específicos (funcionales, no funcionales y restricciones).

---

## 2. Descripción General

### 2.1. Perspectiva del Producto

Sistema web standalone que opera en una intranet local del consultorio:

```
Navegador Web → HTTP localhost:3000 → Node.js + Express → SQLite (eme.db)
                       ↕
              Seguridad: Helmet + CORS + Rate Limit + JWT + RBAC + dotenv
```

### 2.2. Funciones del Producto

| Módulo | Funciones principales |
|--------|-----------------------|
| **Autenticación** | Login, logout, recuperación por código, timeout inactividad, rate limiting |
| **Administración** | Gestión completa de usuarios, auditoría global, restauración de pacientes, estadísticas del sistema |
| **Hub de Pacientes** | Búsqueda, creación, edición, eliminación, agendamiento desde una sola vista |
| **Expedientes** | Apertura automática al crear paciente, archivado en cascada |
| **Citas** | Agendamiento desde detalle del paciente, estados, cancelación |
| **Consultas** | Signos vitales, diagnóstico, tratamiento, antecedentes, prescripciones |
| **Auditoría** | Registro automático de todas las acciones con IP, user-agent, timestamp y datos antes/después |
| **Auto-configuración** | `setup.js` genera el `.env` con JWT_SECRET único; scripts `iniciar.sh`/`.bat` automatizan dependencias y arranque |

### 2.3. Características de los Usuarios

#### Administrador
- **Perfil:** Personal directivo o de sistemas del consultorio.
- **Responsabilidades:** Gestión de cuentas, supervisión de seguridad, recuperación de datos.
- **Restricciones:** NO accede a información clínica de pacientes.

#### Médico
- **Perfil:** Profesional de la salud con cédula profesional.
- **Responsabilidades:** Apertura de expedientes, consultas, diagnósticos, prescripciones.
- **Restricciones:** Único actor que puede eliminar pacientes (con motivo obligatorio).

#### Recepcionista
- **Perfil:** Personal administrativo del consultorio.
- **Responsabilidades:** Registro de pacientes, agendamiento de citas, actualización de datos demográficos.
- **Restricciones:** No accede a información clínica; no puede eliminar pacientes.

### 2.4. Restricciones

- **Lenguaje:** Todo el sistema opera en español (México).
- **Plataforma:** Node.js v18+, navegador moderno.
- **BD:** SQLite local (sin servidor de BD externo).
- **Autenticación:** JWT con expiración de 24h, sesión en sessionStorage.
- **Seguridad:** HTTPS recomendado en producción; en desarrollo funciona en HTTP local.

---

## 3. Requisitos Específicos

### 3.1. Interfaces Externas

- **Interfaz de usuario:** Web SPA (Single Page Application) en HTML5/CSS3/JS vanilla.
- **Interfaz de hardware:** Cualquier computadora con navegador moderno.
- **Interfaz de software:** REST API JSON; SQLite via sql.js (in-process).
- **Interfaz de comunicaciones:** HTTP/HTTPS en red local.

### 3.2. Actores del Sistema

| Actor | Descripción | Permisos clave |
|-------|-------------|----------------|
| **Administrador** | Gestiona usuarios y supervisa el sistema | Crear/editar/desactivar usuarios; auditoría global; restaurar pacientes; estadísticas globales |
| **Médico** | Atiende pacientes y registra información clínica | Todo sobre pacientes; eliminar pacientes (soft delete); consultas; prescripciones; antecedentes |
| **Recepcionista** | Gestiona agenda y datos administrativos | Crear pacientes; editar datos básicos (con motivo); agendar/cancelar citas |

### 3.3. Reglas de Negocio

| ID | Regla |
|----|-------|
| **RN01** | Cada paciente debe tener un identificador único (CURP) que no puede repetirse |
| **RN02** | Cada expediente debe tener un número único con formato `EXP-YYYY-NNNNN` |
| **RN03** | No se permite la creación de un nuevo expediente si ya existe uno asociado a la misma CURP |
| **RN04** | Un paciente puede existir sin expediente, pero un expediente siempre debe estar asociado a un paciente |
| **RN05** | El correo electrónico de cada usuario debe ser único en la base de datos |
| **RN06** | El número de licencia profesional debe ser único y es obligatorio para médicos |
| **RN07** | Las contraseñas se cifran con bcrypt (10 rondas) antes de persistirse |
| **RN08** | El código de recuperación se cifra con bcrypt y se regenera tras cada cambio de contraseña |
| **RN09** | El código de recuperación se muestra una única vez; no se puede recuperar después |
| **RN10** | Solo usuarios activos pueden iniciar sesión |
| **RN11** | La sesión JWT expira tras 24 horas desde el inicio |
| **RN12** | El token temporal para restablecer contraseña expira tras 15 minutos |
| **RN13** | Las contraseñas deben tener mínimo 8 caracteres, una mayúscula, una minúscula y un número |
| **RN14** | El sistema registra la fecha y hora de creación de cada expediente automáticamente |
| **RN15** | Solo médicos pueden crear consultas, registrar diagnósticos y prescribir medicamentos |
| **RN16** | Las recepcionistas no acceden a datos clínicos (consultas, antecedentes), aunque sí ven datos demográficos |
| **RN17** | Cuando una consulta se asocia a una cita, esa cita pasa automáticamente a estado `completada` |
| **RN18** | Una cita puede estar en estado: pendiente, completada o cancelada |
| **RN19** | Las prescripciones se registran automáticamente como antecedentes de tipo `prescripcion` |
| **RN20** | Toda acción modificadora queda registrada en la tabla auditoría con IP, user-agent y timestamp |
| **RN21** | Los registros de auditoría son inmutables: no pueden modificarse ni eliminarse |
| **RN22** | Los intentos fallidos de login se registran como `LOGIN_FAILED` en auditoría |
| **RN23** | Los accesos sin autorización se registran como `ACCESS_DENIED` en auditoría |
| **RN24** | Cada usuario ve solo su propia auditoría; el Administrador tiene acceso adicional a la global |
| **RN25** | El sistema opera en idioma español (México) |
| **RN26** | El primer usuario registrado debe asumir obligatoriamente el rol de Administrador (Bootstrap) |
| **RN27** | La eliminación de pacientes es de tipo lógica (Soft Delete) |
| **RN28** | El sistema cierra la sesión automáticamente tras 15 minutos de inactividad, con aviso 60 seg antes |
| **RN29** | El login tiene rate limiting: máximo 5 intentos por IP cada 15 minutos |
| **RN30** | Solo el Administrador puede crear, editar y desactivar cuentas de usuario |
| **RN31** | No puede desactivarse al último Administrador activo ni puede un Admin desactivarse a sí mismo |
| **RN32** | Solo el Administrador puede restaurar pacientes eliminados; su expediente vuelve a `activo` |
| **RN33** | La sesión se almacena en `sessionStorage` y se destruye al cerrar el navegador |
| **RN34** | Toda edición de datos de paciente requiere un motivo obligatorio (mín. 5 caracteres) |
| **RN35** | Las citas se agendan desde el detalle del paciente, con paciente preseleccionado |
| **RN36** | Al eliminar un paciente, sus citas pendientes se cancelan automáticamente en cascada |
| **RN37** | El secreto JWT se configura en `.env` (no en código); se genera aleatoriamente por instalación |
| **RN38** | Los archivos sensibles (`eme.db`, `.env`) devuelven HTTP 403 si se accede por URL |
| **RN39** | Los mensajes de error de login son genéricos (no revelan si el email existe) |

### 3.4. Requisitos Funcionales

| ID | Descripción | Actor(es) | Prioridad |
|----|-------------|-----------|-----------|
| **RF01** | Registrar usuarios con email, datos personales, contraseña robusta y rol; el sistema valida unicidad, hashea credenciales y genera código de recuperación | Admin / Bootstrap | Alta |
| **RF02** | Autenticar usuarios con email + contraseña; aplicar rate limiting; generar JWT 24h; registrar en auditoría | Todos | Alta |
| **RF03** | Cerrar sesión manualmente o por inactividad (15 min con aviso 60 seg) | Todos | Alta |
| **RF04** | Recuperar contraseña usando código único; generar token de reset 15 min; al cambiar password, generar nuevo código | Todos | Alta |
| **RF05** | Verificar la existencia previa de un paciente por CURP antes de abrir un expediente | Médico, Recep. | Alta |
| **RF06** | Crear paciente y expediente en un solo paso con número único `EXP-YYYY-NNNNN` | Médico, Recep. | Alta |
| **RF07** | Listar pacientes activos con paginación y filtro | Médico, Recep. | Alta |
| **RF08** | Búsqueda inteligente por nombre, apellido, CURP o teléfono (mínimo 3 caracteres) | Médico, Recep. | Alta |
| **RF09** | Consultar detalle del paciente filtrado por rol (médico ve clínico; recepcionista no) | Médico, Recep. | Alta |
| **RF10** | Editar datos del paciente con motivo obligatorio (mín. 5 chars) registrado en auditoría | Médico, Recep. | Alta |
| **RF11** | Eliminar paciente (soft delete) con cascada sobre expediente (archivado) y citas pendientes (canceladas) | Médico | Alta |
| **RF12** | Listar y restaurar pacientes eliminados | Admin | Alta |
| **RF13** | Agendar cita desde el detalle del paciente; paciente preseleccionado en modal | Médico, Recep. | Alta |
| **RF14** | Visualizar agenda de citas (todas y las del día) | Médico, Recep. | Alta |
| **RF15** | Cancelar una cita pendiente, cambiando su estado a `cancelada` | Médico, Recep. | Alta |
| **RF16** | Registrar consulta médica con signos vitales, motivo, síntomas, diagnóstico, tratamiento y prescripciones | Médico | Alta |
| **RF17** | Registrar antecedentes médicos (condiciones crónicas o prescripciones) | Médico | Alta |
| **RF18** | Bloquear acceso a endpoints clínicos para roles no autorizados (HTTP 403 + auditoría) | Sistema | Alta |
| **RF19** | Gestionar (CRUD) cuentas de usuario | Admin | Alta |
| **RF20** | Registrar automáticamente en auditoría toda acción modificadora con datos antes/después | Sistema | Alta |
| **RF21** | Consultar la actividad propia (últimas 100 acciones) | Todos | Media |
| **RF22** | Consultar la auditoría global con filtros por acción y email (últimos 200 registros) | Admin | Alta |
| **RF23** | Mostrar dashboard adaptado al rol con estadísticas relevantes | Todos | Media |
| **RF24** | Listar médicos activos disponibles para asignaciones | Todos | Media |
| **RF25** | Auto-configurar el entorno seguro: `setup.js` genera el `.env` con JWT_SECRET aleatorio de 64 caracteres si no existe | Sistema | Alta |

### 3.5. Requisitos No Funcionales

| ID | Categoría | Descripción |
|----|-----------|-------------|
| **RNF01** | Rendimiento | Búsqueda de pacientes en ≤ 500ms con hasta 1000 registros; dashboard en ≤ 1 segundo |
| **RNF02** | Seguridad | Contraseñas con bcrypt (10 rondas); JWT 24h; tokens de reset 15 min; prepared statements |
| **RNF03** | Seguridad | RBAC: middleware `requireRole()` retorna HTTP 403 ante roles no autorizados, con auditoría `ACCESS_DENIED` |
| **RNF04** | Seguridad | Rate limiting: 5 intentos por IP / 15 minutos en el endpoint de login (HTTP 429) |
| **RNF05** | Seguridad | Cabeceras HTTP seguras vía Helmet.js (X-Frame-Options, X-Content-Type-Options, Referrer-Policy) |
| **RNF06** | Seguridad | JWT_SECRET en `.env` no versionado; auto-generación aleatoria en cada instalación mediante `setup.js` |
| **RNF07** | Seguridad | Sesión en sessionStorage (no persistente al cerrar el navegador) |
| **RNF08** | Seguridad | Cierre automático de sesión tras 15 min de inactividad con aviso de 60 segundos |
| **RNF09** | Seguridad | Bloqueo HTTP 403 al intentar acceder por URL a `eme.db` y `.env` |
| **RNF10** | Usabilidad | Compatible con Chrome 120+, Firefox 120+, Edge 120+ y Safari 17+ |
| **RNF11** | Portabilidad | SQLite embebido vía sql.js (sin servidor de BD externo) |
| **RNF12** | Portabilidad | Auto-configuración multiplataforma: scripts `iniciar.sh` (Linux/macOS) e `iniciar.bat` (Windows) |
| **RNF13** | Mantenibilidad | Código modular en `server.js`; frontend desacoplado vía REST |
| **RNF14** | Trazabilidad | Auditoría inmutable con usuario, rol, acción, entidad, IP, user-agent, datos JSON antes/después |
| **RNF15** | Privacidad | Datos clínicos restringidos al rol Médico; recepcionista y admin no acceden vía ningún endpoint |
| **RNF16** | Normatividad | Documentación conforme a IEEE 830-1998; diagramas UML en PlantUML |
| **RNF17** | Localización | Interfaz, mensajes y errores en español (México) |
| **RNF18** | Integridad | Eliminación lógica preserva los datos clínicos; sin borrado físico |

---

## 4. Apéndices

### 4.1. Tabla de Trazabilidad

| Caso de Uso | Reglas de Negocio | Requisitos Funcionales |
|-------------|------------------|------------------------|
| CU01: Bootstrap del Admin | RN5, RN6, RN7, RN8, RN9, RN13, RN20, RN26 | RF01, RF20, RF25 |
| CU02: Iniciar sesión | RN10, RN11, RN20, RN22, RN29, RN39 | RF02, RF20 |
| CU03: Cerrar sesión | RN20, RN28, RN33 | RF03, RF20 |
| CU04: Gestionar usuarios | RN5, RN6, RN7, RN13, RN20, RN30, RN31 | RF01, RF19, RF20 |
| CU05: Apertura de expediente | RN1, RN2, RN3, RN4, RN14, RN20 | RF05, RF06, RF20 |
| CU06: Búsqueda de pacientes | — | RF07, RF08 |
| CU07: Editar paciente | RN20, RN34 | RF10, RF20 |
| CU08: Eliminación lógica | RN20, RN27, RN36 | RF11, RF18, RF20 |
| CU09: Restaurar paciente | RN20, RN32 | RF12, RF20 |
| CU10: Agendar cita | RN18, RN20, RN35 | RF13, RF20 |
| CU11: Registrar consulta | RN15, RN16, RN17, RN19, RN20 | RF16, RF17, RF18, RF20 |
| CU12: Mi actividad | RN24 | RF21 |
| CU13: Auditoría global | RN20, RN21, RN24 | RF22 |
| CU14: Recuperar contraseña | RN8, RN9, RN12, RN13, RN20 | RF04, RF20 |

### 4.2. Matriz de Permisos

| Acción | Admin | Médico | Recepcionista |
|--------|:-----:|:------:|:-------------:|
| Login / Logout | ✅ | ✅ | ✅ |
| Recuperar contraseña | ✅ | ✅ | ✅ |
| Ver "Mi actividad" | ✅ | ✅ | ✅ |
| Crear usuarios | ✅ | ❌ | ❌ |
| Editar usuarios | ✅ | ❌ | ❌ |
| Activar/desactivar usuarios | ✅ | ❌ | ❌ |
| Ver auditoría global | ✅ | ❌ | ❌ |
| Ver pacientes eliminados | ✅ | ❌ | ❌ |
| Restaurar pacientes | ✅ | ❌ | ❌ |
| Ver estadísticas globales | ✅ | ❌ | ❌ |
| Buscar pacientes | ❌ | ✅ | ✅ |
| Crear pacientes | ❌ | ✅ | ✅ |
| Editar pacientes (con motivo) | ❌ | ✅ | ✅ |
| Eliminar pacientes (soft delete) | ❌ | ✅ | ❌ |
| Verificar existencia por CURP | ❌ | ✅ | ✅ |
| Agendar/cancelar citas | ❌ | ✅ | ✅ |
| Ver agenda del día | ❌ | ✅ | ✅ |
| Ver datos clínicos | ❌ | ✅ | ❌ |
| Registrar consulta | ❌ | ✅ | ❌ |
| Registrar antecedentes | ❌ | ✅ | ❌ |
| Listar médicos | ✅ | ✅ | ✅ |

### 4.3. Endpoints REST (32 totales)

#### Autenticación (5)
- `GET /api/auth/setup-status` (público)
- `POST /api/auth/registro` (bootstrap o admin)
- `POST /api/auth/login` (público, rate-limited)
- `POST /api/auth/verificar-codigo` (público)
- `POST /api/auth/reset-password` (público)

#### Pacientes (8)
- `GET /api/pacientes` (médico, recep)
- `GET /api/pacientes/buscar/:termino` (médico, recep)
- `GET /api/pacientes/verificar/:identidad` (médico, recep)
- `GET /api/pacientes/:id` (médico, recep)
- `POST /api/pacientes` (médico, recep)
- `POST /api/pacientes/apertura-expediente` (médico, recep)
- `PUT /api/pacientes/:id` (médico, recep)
- `DELETE /api/pacientes/:id` (solo médico)

#### Citas (5)
- `GET /api/citas` (médico, recep)
- `GET /api/citas/hoy` (médico, recep)
- `POST /api/citas` (médico, recep)
- `PUT /api/citas/:id` (médico, recep)
- `DELETE /api/citas/:id` (médico, recep)

#### Consultas y Antecedentes (3)
- `POST /api/consultas` (solo médico)
- `GET /api/consultas/:id` (solo médico)
- `POST /api/antecedentes` (solo médico)

#### Auditoría (2)
- `GET /api/auditoria/mias` (autenticado)
- `GET /api/auditoria/todas` (solo admin)

#### Administración (6)
- `GET /api/admin/usuarios` (solo admin)
- `PUT /api/admin/usuarios/:id` (solo admin)
- `PUT /api/admin/usuarios/:id/estado` (solo admin)
- `GET /api/admin/pacientes-eliminados` (solo admin)
- `POST /api/admin/pacientes/:id/restaurar` (solo admin)
- `GET /api/admin/stats` (solo admin)

#### Utilidades (3)
- `GET /api/medicos` (autenticado)
- `GET /api/dashboard/stats` (autenticado)
- `GET /api/usuario/info` (autenticado)
