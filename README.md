# 🏥 Reporte01_ExpedienteMedicoElectronico

Proyecto escolar para diseñar un Expediente Médico Electrónico orientado a consultorios médicos pequeños, conforme al estándar **IEEE 830-1998**.

**Autor:** Edson Joel Carrera Avila

---

## 📌 Planteamiento

Muchos consultorios médicos pequeños y medianos gestionan la información clínica de sus pacientes mediante expedientes en papel u hojas de cálculo, lo que provoca pérdida de información, duplicidad de registros, demoras en la atención, dificultades para dar seguimiento al historial y problemas de confidencialidad.

El sistema **EME** plantea una solución web centralizada que mejora la calidad de la atención clínica, reduce el tiempo administrativo, garantiza la confidencialidad mediante **control de acceso basado en roles (RBAC)** y proporciona **trazabilidad completa** mediante auditoría inmutable.

---

## 🎯 Alcance

**Incluye:** apertura formal de expedientes, gestión de pacientes y citas, registro de consultas con prescripciones, autenticación con tres roles (Administrador, Médico, Recepcionista), auditoría completa y medidas de ciberseguridad.

**No incluye:** seguros médicos, facturación, telemedicina, app móvil nativa, integración con laboratorios externos ni firma digital certificada.

---

## 🎓 Objetivos

### Objetivo general
Desarrollar un sistema web de gestión de expedientes médicos electrónicos que permita al personal de un consultorio pequeño registrar, consultar y administrar la información clínica y administrativa de los pacientes de forma segura, eficiente y trazable, conforme al estándar IEEE 830-1998.

### Objetivos específicos
1. Implementar el módulo de apertura formal de expediente.
2. Desarrollar autenticación con tres roles diferenciados.
3. Implementar la gestión de citas centrada en el paciente.
4. Desarrollar el módulo de consultas médicas exclusivo para médicos.
5. Construir un sistema RBAC que separe datos administrativos de clínicos.
6. Implementar auditoría inmutable de todas las acciones.
7. Incorporar medidas de ciberseguridad (rate limiting, Helmet, sessionStorage, etc.).
8. Generar documentación formal conforme a IEEE 830-1998.

---

## 🛠️ Stack tecnológico previsto

| Capa | Tecnología |
|------|-----------|
| Backend | Node.js + Express.js |
| Frontend | HTML5 + CSS3 + JavaScript Vanilla (SPA) |
| Base de datos | SQLite (sql.js) |
| Autenticación | JWT + bcryptjs |
| Seguridad | Helmet.js + express-rate-limit + dotenv |

---

## 👥 Roles del sistema

- **Administrador** — Gestiona cuentas, supervisa auditoría y restaura pacientes. No accede a datos clínicos.
- **Médico** — Acceso completo a expedientes, consultas, prescripciones y eliminación lógica de pacientes.
- **Recepcionista** — Registro de pacientes y agendamiento de citas. No accede a datos clínicos.

---

## 📚 Documentación del proyecto

| Documento | Descripción |
|-----------|-------------|
| 📋 [**EME_ERS_IEEE830.md**](EME_ERS_IEEE830.md) | Especificación de Requisitos de Software (IEEE 830-1998): reglas de negocio, requisitos funcionales y no funcionales, trazabilidad y matriz de permisos. |
| 🗂️ [**EME_Esquema_BD.md**](EME_Esquema_BD.md) | Esquema de la base de datos: tablas, campos, relaciones y diagrama ER. |
| 🎨 [**EME_Diagramas_PlantUML.md**](EME_Diagramas_PlantUML.md) | Diagramas UML del sistema en formato PlantUML editable. |

---

## 📖 Referencias

- IEEE 830-1998 — Recommended Practice for Software Requirements Specifications
- NOM-024-SSA3-2012 — Expediente Clínico Electrónico (México)
- OWASP Top 10:2021 — Mejores prácticas de seguridad web
