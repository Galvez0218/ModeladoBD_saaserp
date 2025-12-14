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
