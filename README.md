# 🏥 Reporte01_ExpedienteMedicoElectronico

Sistema web de gestión de expedientes médicos para consultorios pequeños, conforme al estándar **IEEE 830-1998**.

**Autor:** Edson Joel Carrera Avila

---

## 📌 Planteamiento

Muchos consultorios pequeños gestionan expedientes en papel u hojas de cálculo, lo que genera pérdida de información, duplicidad de registros y problemas de confidencialidad. **EME** resuelve esto con una solución web centralizada con RBAC y auditoría inmutable.

---

## 🎯 Alcance

**Incluye:** apertura formal de expedientes, gestión de pacientes y citas, registro de consultas con prescripciones, autenticación con tres roles (Administrador, Médico, Recepcionista), auditoría completa y medidas de ciberseguridad.

**No incluye:** seguros médicos, facturación, telemedicina, app móvil nativa, integración con laboratorios ni firma digital certificada.

---

## 🛠️ Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Backend | Node.js + Express.js |
| Frontend | HTML5 + CSS3 + JavaScript Vanilla (SPA) |
| Base de datos | SQLite (sql.js) |
| Autenticación | JWT + bcryptjs |
| Seguridad | Helmet.js + express-rate-limit + dotenv |

---

## 👥 Roles del sistema

| Rol | Responsabilidad |
|-----|----------------|
| **Administrador** | Gestiona cuentas y auditoría global. No accede a datos clínicos. |
| **Médico** | Acceso completo a expedientes, consultas, prescripciones y eliminación lógica de pacientes. |
| **Recepcionista** | Registro de pacientes y agendamiento de citas. No accede a datos clínicos. |

---

## 🛠️ Estructura del proyecto

```
eme-prototipo/
├── server.js        # Backend Node.js + Express (32 endpoints REST)
├── setup.js         # Auto-generador del .env
├── package.json
├── iniciar.sh / iniciar.bat
├── .env             # Auto-generado, no versionado en Git
├── eme.db           # Base de datos SQLite (auto-generada)
└── public/
    ├── index.html   # Frontend SPA
    ├── styles.css
    └── app.js       # Lógica del cliente (sessionStorage, modales, timer)
```

---

## 🔐 Seguridad

- Contraseñas cifradas con `bcrypt` (10 rondas).
- JWT con expiración de 24 h almacenado en `sessionStorage` (no persistente).
- Rate limiting: 5 intentos de login por IP cada 15 min → HTTP 429.
- Cierre automático de sesión tras 15 min de inactividad con aviso previo de 60 seg.
- Cabeceras HTTP seguras vía `Helmet.js`.
- `eme.db` y `.env` devuelven HTTP 403 si se accede por URL.
- Mensajes de error de login genéricos (no revelan si el email existe).
- `JWT_SECRET` generado aleatoriamente en cada instalación vía `setup.js`.

---

## 🌐 Endpoints REST (32)

### Autenticación
| Método | Endpoint | Roles |
|--------|----------|-------|
| GET | `/api/auth/setup-status` | Público |
| POST | `/api/auth/registro` | Bootstrap / Admin |
| POST | `/api/auth/login` | Público |
| POST | `/api/auth/verificar-codigo` | Público |
| POST | `/api/auth/reset-password` | Público |

### Pacientes
| Método | Endpoint | Roles |
|--------|----------|-------|
| GET | `/api/pacientes` | Médico, Recep. |
| GET | `/api/pacientes/buscar/:termino` | Médico, Recep. |
| GET | `/api/pacientes/verificar/:identidad` | Médico, Recep. |
| GET | `/api/pacientes/:id` | Médico, Recep. |
| POST | `/api/pacientes` | Médico, Recep. |
| POST | `/api/pacientes/apertura-expediente` | Médico, Recep. |
| PUT | `/api/pacientes/:id` | Médico, Recep. |
| DELETE | `/api/pacientes/:id` | Solo Médico |

### Citas
| Método | Endpoint | Roles |
|--------|----------|-------|
| GET | `/api/citas` | Médico, Recep. |
| GET | `/api/citas/hoy` | Médico, Recep. |
| POST | `/api/citas` | Médico, Recep. |
| PUT | `/api/citas/:id` | Médico, Recep. |
| DELETE | `/api/citas/:id` | Médico, Recep. |

### Consultas y Antecedentes
| Método | Endpoint | Roles |
|--------|----------|-------|
| POST | `/api/consultas` | Solo Médico |
| GET | `/api/consultas/:id` | Solo Médico |
| POST | `/api/antecedentes` | Solo Médico |

### Auditoría
| Método | Endpoint | Roles |
|--------|----------|-------|
| GET | `/api/auditoria/mias` | Todos |
| GET | `/api/auditoria/todas` | Solo Admin |

### Administración
| Método | Endpoint | Roles |
|--------|----------|-------|
| GET | `/api/admin/usuarios` | Solo Admin |
| PUT | `/api/admin/usuarios/:id` | Solo Admin |
| PUT | `/api/admin/usuarios/:id/estado` | Solo Admin |
| GET | `/api/admin/pacientes-eliminados` | Solo Admin |
| POST | `/api/admin/pacientes/:id/restaurar` | Solo Admin |
| GET | `/api/admin/stats` | Solo Admin |

### Utilidades
| Método | Endpoint | Roles |
|--------|----------|-------|
| GET | `/api/medicos` | Autenticado |
| GET | `/api/dashboard/stats` | Autenticado |
| GET | `/api/usuario/info` | Autenticado |

---

## 📚 Documentación del proyecto

| Documento | Descripción |
|-----------|-------------|
| 📋 [EME_ERS_IEEE830.md](EME_ERS_IEEE830.md) | Especificación de Requisitos (IEEE 830-1998): reglas de negocio, RF, RNF, trazabilidad y matriz de permisos. |
| 🗂️ [EME_Esquema_BD.md](EME_Esquema_BD.md) | Esquema de la base de datos: tablas, campos, relaciones y diagrama ER. |
| 🎨 [EME_Diagramas_PlantUML.md](EME_Diagramas_PlantUML.md) | 8 diagramas UML del sistema en formato PlantUML editable. |

---

## 📖 Referencias

- IEEE 830-1998 — Recommended Practice for Software Requirements Specifications
- NOM-024-SSA3-2012 — Expediente Clínico Electrónico (México)
- OWASP Top 10:2021 — https://owasp.org/Top10/