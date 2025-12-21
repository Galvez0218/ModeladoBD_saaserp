# Changelog

Todos los cambios notables en el diseño de la base de datos serán documentados en este archivo.

El formato está basado en [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
y este proyecto adhiere a [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Considerando
- Tabla para logs de auditoría centralizados
- Sistema de notificaciones
- Tabla para configuraciones por business

---

## [1.2.0] - 2025-12-20

### 🔗 Plans-Modules Relationship

#### Added - SaaS Plans Module
- Tabla `plan_x_modules` - Relación muchos a muchos entre planes y módulos
  - Campos: plan_id, module_id
  - Define explícitamente qué módulos incluye cada plan
  - Índice único compuesto (plan_id, module_id)
  - Índices para búsquedas por plan y por módulo

#### Changed - TableGroups
- Actualizado `saas_plans` group para incluir `plan_x_modules`
  - Nuevo orden: plans → plans_detail → plan_x_modules → business_plans → business_plan_history

#### Decisiones Arquitectónicas
- ✅ Separación explícita de qué módulos incluye cada plan
- ✅ Facilita upgrades/downgrades de planes
- ✅ Permite crear planes personalizados con combinaciones únicas de módulos
- ✅ Mejora la trazabilidad: Plan → plan_x_modules → Modules → business_modules

#### Flujo de Datos
```
1. Plan "Pro" define módulos via plan_x_modules
2. Business contrata Plan "Pro" → business_plans
3. Se copian módulos del plan → business_modules
4. Módulos activos definen permisos disponibles
```

#### Ejemplos de Uso
```
Plan Free → plan_x_modules: [Inventario básico]
Plan Basic → plan_x_modules: [Inventario, Almacén]
Plan Pro → plan_x_modules: [Inventario, Almacén, POS, Ventas, Reportes]
Plan Enterprise → plan_x_modules: [Todos los módulos + API]
```

---

## [1.1.0] - 2025-12-14

### 🎯 Business Entities Separation

#### Added - CRM Module
- Tabla `customers` - Clientes sin login por defecto
  - Campos: business_id (tenant), customer_code, name, document_type_id, document_number
  - Email y phone opcionales (pueden no tener)
  - credit_limit y current_debt para gestión comercial
  - user_id nullable para dar acceso opcional al portal
  - Índices optimizados para búsquedas por business y documento

#### Added - Human Resources Module
- Tabla `employees` - Empleados de planilla sin login por defecto
  - Campos: business_id (tenant), branch_id, employee_code, names, apellidos
  - document_type_id y document_number para identificación
  - hire_date, termination_date, position, salary
  - user_id nullable para dar acceso opcional al ERP
  - Índices optimizados para búsquedas por business, branch y documento

#### Added - Catalog Module
- Tabla `document_types` - Catálogo de tipos de documento
  - Campos: name, code (código SUNAT), max_length, min_length
  - country_id para especificidad por país
  - Ejemplos: DNI (8 chars), RUC (11 chars), Carnet Extranjería (12 chars)
  - Soporte para validaciones de longitud automáticas

#### Changed - Business Details
- Campo `tax_id` renombrado a `document_number` en business_details
- Agregado `document_type_id` FK a document_types
- Mejora en normalización de datos de documentos

#### Decisiones Arquitectónicas
- ✅ Separación clara: users (login) vs customers/employees (entidades negocio)
- ✅ customers y employees NO tienen login por defecto
- ✅ user_id nullable en customers/employees para promoción opcional
- ✅ Catálogo centralizado de document_types para validaciones
- ✅ Evita confusión entre "usuario del sistema" y "persona registrada"
- ✅ Email opcional en customers/employees (pueden no tener)

#### Índices Agregados
- `idx_customers_business`, `idx_customers_business_active`
- `idx_customers_document`, `idx_customers_business_document`
- `idx_employees_business`, `idx_employees_business_active`
- `idx_employees_branch`, `idx_employees_document`
- `idx_document_types_code`, `idx_document_types_country`

#### TableGroups Agregados
- `crm` (customers)
- `human_resources` (employees)  
- `catalogs` reorganizado (countries, document_types, business_x_countries)

---

## [1.0.0] - 2025-12-14

### 🎉 Initial Release

#### Added - Authentication & Users
- Tabla `users` - Usuarios globales del sistema
- Tabla `user_details` - Información personal de usuarios
- Tabla `user_detail_numbers` - Números de contacto múltiples
- Tabla `refresh_tokens` - Sistema de refresh tokens para JWT

#### Added - Multitenancy Core
- Tabla `business` - Empresas/tenants con owner_id
- Tabla `business_details` - Detalles legales (RUC, dirección, etc)
- Tabla `user_business` - **Pivote principal** para multitenancy
- Tabla `user_business_x_role` - Roles de usuarios en empresas
- Tabla `countries` - Catálogo de países
- Tabla `business_x_countries` - Países donde opera cada empresa

#### Added - Branches (Sucursales)
- Tabla `branches` - Sucursales/sedes de empresas
- Tabla `user_branches` - Asignación de usuarios a múltiples sucursales
- Campo `primary_branch_id` en `user_business` para sucursal predeterminada
- Campo `is_primary` en `user_branches` para marcar sucursal principal

#### Added - Authorization Business Level
- Tabla `business_roles` - Roles personalizados por empresa
- Tabla `business_permissions` - Permisos disponibles por empresa
- Tabla `business_role_x_permissions` - Relación roles-permisos

#### Added - Authorization Global Level
- Tabla `global_roles` - Roles a nivel plataforma
- Tabla `global_permissions` - Permisos globales de plataforma
- Tabla `user_x_global_roles` - Asignación de roles globales
- Tabla `user_global_role_x_permissions` - Permisos de roles globales

#### Added - Plans & Subscriptions
- Tabla `plans` - Planes disponibles (Free, Basic, Pro, Enterprise)
- Tabla `plans_detail` - Características de cada plan
- Tabla `business_plans` - Plan activo por empresa
- Tabla `business_plan_history` - Historial de cambios de plan

#### Added - Modules System
- Tabla `modules` - Módulos del ERP (Almacén, Ventas, etc)
- Tabla `modules_menu` - Estructura de menús por módulo
- Tabla `business_modules` - Módulos activos por empresa
- Tabla `module_permissions` - Relación módulos-permisos

#### Decisiones Arquitectónicas
- ✅ Adoptado modelo multitenancy: Shared Database, Shared Schema
- ✅ `user_business` como pivote principal (muchos a muchos)
- ✅ `owner_id` en `business` para privilegios únicos del fundador
- ✅ Doble sistema de permisos: global (plataforma) + business (tenant)
- ✅ Usuarios pueden pertenecer a múltiples empresas con roles diferentes
- ✅ Usuarios pueden trabajar en múltiples sucursales
- ✅ Sistema modular basado en planes SaaS
- ❌ Descartado `gruposRazonesSociales` por complejidad innecesaria

#### Índices Optimizados
- Índices únicos en combinaciones (user_id, business_id)
- Índices compuestos para queries frecuentes (business_id, is_active)
- Índices en todas las foreign keys
- Índices en campos de búsqueda y filtrado

#### Campos de Auditoría
- Implementado en todas las tablas transaccionales:
  - created_at, updated_at
  - created_by, updated_by, deleted_by
  - deleted_at (soft deletes)

---

## Template para Próximas Versiones

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Added
- Nueva tabla `nombre_tabla` - Descripción
- Nuevo campo `nombre_campo` en tabla `tabla_x`

### Changed
- Modificado índice en tabla `tabla_y`
- Renombrado campo `old_name` a `new_name`

### Deprecated
- Campo `campo_deprecado` será removido en v2.0.0

### Removed
- Tabla `tabla_antigua`
- Campo `campo_obsoleto`

### Fixed
- Corregida relación entre `tabla_a` y `tabla_b`

### Security
- Agregado índice para prevenir N+1 queries
```

---

## Versionado

- **MAJOR** (X.0.0): Cambios incompatibles en estructura
- **MINOR** (x.Y.0): Nueva funcionalidad compatible
- **PATCH** (x.y.Z): Correcciones y mejoras menores

---

## Notas

- Para ver cambios entre versiones: `git diff v1.0.0 v1.1.0 -- dbml/`
- Para rollback: `git checkout v1.0.0 -- dbml/modelado_erp_completo.dbml`
