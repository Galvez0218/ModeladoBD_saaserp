# 📊 ERP Multi-Tenant - Database Design

> Sistema de base de datos para ERP multi-tenant con arquitectura Laravel

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/tu-usuario/modelado-erp/releases)
[![Database](https://img.shields.io/badge/database-MySQL-orange.svg)](https://www.mysql.com/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

---

## 🎯 Características Principales

- ✅ **Multitenancy Avanzado** - User-Business pivot para múltiples empresas por usuario
- ✅ **Sistema SaaS Completo** - Planes, módulos y suscripciones
- ✅ **Permisos Granulares** - Doble capa: Global (plataforma) + Business (tenant)
- ✅ **Gestión de Sucursales** - Control por ubicación física
- ✅ **Auditoría Completa** - Tracking de cambios y soft deletes
- ✅ **Escalable** - Índices optimizados para alto rendimiento

---

## 🏗️ Arquitectura

### Modelo de Multitenancy

```
┌──────────────────────────────────────────────────────┐
│              NIVEL GLOBAL (Plataforma)                │
│  • global_roles (superadmin, support, auditor)       │
│  • global_permissions                                 │
│  • users (usuarios globales)                         │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│              NIVEL TENANT (Business)                  │
│  ┌────────────────────────────────────────────────┐  │
│  │  BUSINESS (Tenant/Empresa)                     │  │
│  │  • business_plans → plans                      │  │
│  │  • business_modules → modules                  │  │
│  │  • business_roles → business_permissions       │  │
│  │  • branches (sucursales)                       │  │
│  │                                                 │  │
│  │  USUARIOS:                                      │  │
│  │  • user_business (pivote)                      │  │
│  │  • user_business_x_role                        │  │
│  │  • user_branches                               │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

### Jerarquía de Permisos

```
Plan → Módulos → Permisos → Roles → Usuarios
```

**Ejemplo:**
```
Plan Pro
  └─> Módulos: [Almacén, POS, Ventas]
      └─> Permisos: [almacen.view, almacen.create, pos.sell, ...]
          └─> Rol "Vendedor": [pos.sell, almacen.view]
              └─> Usuario "Juan": role='Vendedor' en Business "ABC"
```

---

## 📁 Estructura del Repositorio

```
modelado-erp/
│
├── README.md                          ← Estás aquí
├── CHANGELOG.md                       ← Historial de versiones
├── GUIA_USO.md                        ← Cómo usar estos archivos
│
├── dbml/
│   ├── modelado_erp_completo.dbml    ← Script principal
│   └── versiones/
│       └── v1.0.0_2025-12-14.dbml    ← Versiones anteriores
│
├── docs/
│   ├── DOCUMENTACION_MODELADO.md     ← Documentación completa
│   ├── DECISIONES_ARQUITECTONICAS.md ← Por qué se eligió cada cosa
│   └── CASOS_DE_USO.md               ← Ejemplos prácticos
│
└── exports/
    ├── diagrams/
    │   ├── full_diagram_v1.0.0.png   ← Diagrama completo
    │   └── full_diagram_v1.0.0.pdf
    │
    └── sql/
        └── schema_v1.0.0.sql         ← SQL generado (referencia)
```

---

## 🚀 Inicio Rápido

### 1. Visualizar el Modelado

**Opción A: dbdiagram.io (Recomendado)**
```bash
1. Ir a https://dbdiagram.io/d
2. Click "New Diagram"
3. Copiar contenido de dbml/modelado_erp_completo.dbml
4. Pegar en el editor
5. ¡Listo! El diagrama se genera automáticamente
```

**Opción B: Imagen Exportada**
```bash
Ver exports/diagrams/full_diagram_v1.0.0.png
```

### 2. Leer la Documentación

```bash
1. Leer README.md (este archivo) - Visión general
2. Leer docs/DOCUMENTACION_MODELADO.md - Documentación completa
3. Revisar CHANGELOG.md - Historial de cambios
```

### 3. Usar en tu Proyecto Laravel

⚠️ **NO uses el SQL exportado directamente**. Úsalo como referencia.

**Proceso recomendado:**
1. Revisar el modelado en dbdiagram.io
2. Crear migraciones Laravel manualmente basándote en el diseño
3. Implementar modelos Eloquent con relaciones
4. Implementar middleware TenantIdentificationMiddleware

---

## 📊 Estadísticas del Modelado

| Componente | Cantidad |
|------------|----------|
| **Tablas** | 33 |
| **Relaciones** | 45+ |
| **Índices** | 60+ |
| **TableGroups** | 8 |

### Grupos de Tablas

1. **Authentication** (4 tablas) - Users, details, numbers, tokens
2. **Multitenancy Core** (6 tablas) - Business, user_business, countries
3. **Branches** (2 tablas) - Sucursales y asignaciones
4. **Authorization Business** (3 tablas) - Roles y permisos por tenant
5. **Authorization Global** (4 tablas) - Roles y permisos de plataforma
6. **Plans & Subscriptions** (4 tablas) - Sistema SaaS
7. **Modules** (4 tablas) - Módulos activables
8. **Geography** (2 tablas) - Países

---

## 🎯 Casos de Uso Principales

### Caso 1: Usuario Multi-Empresa

```
Usuario "María" es contadora freelance:
├── Business A (role='admin')
├── Business B (role='superadmin')
└── Business C (role='member')

Usa selector de empresas para cambiar contexto
```

### Caso 2: Gerente Multi-Sucursal

```
Usuario "Carlos" supervisa 3 sucursales:
└── Business X
    ├── Sucursal Norte (is_primary=true)
    ├── Sucursal Sur
    └── Sucursal Este

Puede ver inventarios consolidados y reportes por sucursal
```

### Caso 3: Upgrade de Plan

```
Business "ABC SAC"
├── Plan Básico → [Almacén, Inventario]
│   └── Permisos limitados
│
└── UPGRADE a Plan Pro
    └── Nuevos módulos: [POS, Ventas, Reportes]
        └── Nuevos permisos disponibles para asignar
```

---

## 🔑 Decisiones Arquitectónicas Clave

### 1. `owner_id` vs `role='superadmin'`

**DECISIÓN:** Mantener AMBOS

- `business.owner_id` = Fundador con privilegios únicos (cerrar empresa, transferir ownership)
- `role='superadmin'` = Admin completo pero sin acciones críticas

### 2. Usuarios en Múltiples Sucursales

**DECISIÓN:** ✅ Permitir

- Tabla `user_branches` para asignación múltiple
- Campo `primary_branch_id` en `user_business` para sucursal predeterminada

### 3. Sistema de Permisos

**DECISIÓN:** Doble capa (Global + Business)

- **Global:** Para administradores de PLATAFORMA
- **Business:** Para usuarios de EMPRESAS

### 4. Planes y Módulos

**DECISIÓN:** Sistema modular basado en suscripciones

```
Plan → define módulos disponibles
Business → activa módulos según plan
Módulos → definen permisos base
Roles → asignan permisos a usuarios
```

---

## 🛠️ Tecnologías

- **Base de Datos:** MySQL 8.0+
- **Framework:** Laravel 12 (PHP 8.4)
- **Diagramación:** dbdiagram.io (DBML)
- **Versionado:** Git + Semantic Versioning

---

## 📖 Documentación Completa

- **[DOCUMENTACION_MODELADO.md](docs/DOCUMENTACION_MODELADO.md)** - Documentación exhaustiva
- **[GUIA_USO.md](GUIA_USO.md)** - Cómo usar estos archivos
- **[CHANGELOG.md](CHANGELOG.md)** - Historial de versiones
- **[DBML Online](https://dbdiagram.io/d)** - Visualizar en dbdiagram.io

---

## 🔄 Mantenimiento

### Agregar Nueva Tabla

1. Editar `dbml/modelado_erp_completo.dbml`
2. Actualizar `docs/DOCUMENTACION_MODELADO.md`
3. Actualizar `CHANGELOG.md`
4. Exportar nuevo diagrama
5. Commit y tag de versión

```bash
git add .
git commit -m "Add: Nueva tabla para funcionalidad X"
git tag -a v1.1.0 -m "Added nueva_tabla"
git push origin main --tags
```

### Versionado

- **MAJOR (X.0.0):** Cambios incompatibles
- **MINOR (x.Y.0):** Nueva funcionalidad compatible
- **PATCH (x.y.Z):** Correcciones menores

---

## 📊 Diagramas

### Diagrama ER Completo
![Diagrama Completo](exports/diagrams/full_diagram_v1.0.0.png)

### Diagrama de Multitenancy
```
users ──< user_business >── business
                 │
                 ├── user_business_x_role ── business_roles
                 └── user_branches ── branches
```

---

## 🤝 Contribuir

1. Fork el repositorio
2. Crear rama feature (`git checkout -b feature/nueva-tabla`)
3. Commit cambios (`git commit -m 'Add: Nueva tabla X'`)
4. Push a la rama (`git push origin feature/nueva-tabla`)
5. Crear Pull Request

---

## 📝 Convenciones

### Nomenclatura
- Tablas en plural inglés: `users`, `businesses`
- Snake_case: `user_business`, `business_plans`
- Pivotes: `{tabla1}_x_{tabla2}`

### Campos Estándar
```sql
id bigint [pk, increment]
created_at timestamp
updated_at timestamp
created_by bigint
updated_by bigint
deleted_at timestamp
deleted_by bigint
```

### Índices
- PK automático
- FK siempre indexado
- Unique en combinaciones (user_id, business_id)
- Compuestos en queries frecuentes (business_id, is_active)

---

## 🚀 Roadmap

### Fase 1: Core Multitenancy ✅
- [x] Modelado base completo
- [x] Sistema de permisos
- [x] Planes y módulos
- [x] Documentación

### Fase 2: Módulos Transaccionales ⏳
- [ ] Productos/Servicios
- [ ] Inventario
- [ ] Ventas
- [ ] Compras
- [ ] Clientes/Proveedores

### Fase 3: Módulos Avanzados
- [ ] Contabilidad
- [ ] RRHH
- [ ] Reportes avanzados
- [ ] API externa

### Fase 4: Optimizaciones
- [ ] Caché de permisos
- [ ] Performance tuning
- [ ] Índices adicionales

---

## 📞 Recursos

- **DBML Documentation:** https://dbml.dbdiagram.io/docs/
- **dbdiagram.io:** https://dbdiagram.io
- **Laravel Multi-tenancy:** https://laravel.com/docs/11.x
- **Semantic Versioning:** https://semver.org/

---

## 📜 Licencia

MIT License - Ver [LICENSE](LICENSE) para más detalles

---

## 👤 Autor

**KuroNeko**

- Proyecto: ERP Multi-tenant
- Versión: 1.0.0
- Fecha: 2025-12-14

---

## ⭐ Agradecimientos

- Comunidad Laravel
- dbdiagram.io por la excelente herramienta
- Patrones de multi-tenancy de la comunidad

---

**Versión:** 1.0.0  
**Última Actualización:** 2025-12-14  
**Estado:** ✅ Estable
