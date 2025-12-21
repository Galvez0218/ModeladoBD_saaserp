# 📊 DOCUMENTACIÓN COMPLETA - MODELADO BASE DE DATOS ERP

**Proyecto:** Sistema ERP Multi-tenant
**Versión:** 1.2.0
**Fecha:** 2025-12-20
**Arquitectura:** Laravel Multi-tenant con user_business pivot
**Base de Datos:** MySQL  

---

## 📋 ÍNDICE

1. [Visión General](#visión-general)
2. [Arquitectura de Multitenancy](#arquitectura-de-multitenancy)
3. [Decisiones Arquitectónicas Clave](#decisiones-arquitectónicas-clave)
4. [Estructura por Módulos](#estructura-por-módulos)
5. [Flujos de Datos Principales](#flujos-de-datos-principales)
6. [Jerarquía de Permisos](#jerarquía-de-permisos)
7. [Sistema de Planes y Módulos](#sistema-de-planes-y-módulos)
8. [Casos de Uso](#casos-de-uso)
9. [Convenciones y Estándares](#convenciones-y-estándares)
10. [Próximos Pasos](#próximos-pasos)

---

## 🎯 VISIÓN GENERAL

### Propósito del Sistema

Sistema ERP SaaS multi-tenant que permite:
- **Múltiples empresas (tenants)** en una sola base de datos
- **Usuarios multi-empresa**: Un usuario puede pertenecer a varias empresas con diferentes roles
- **Sistema de suscripciones**: Planes con módulos activables
- **Permisos granulares**: A nivel global (plataforma) y por empresa (tenant)
- **Gestión de sucursales**: Control de inventarios, ventas y costos por ubicación

### Arquitectura General

```
┌─────────────────────────────────────────────────────────────────┐
│                    NIVEL PLATAFORMA (Global)                     │
│  - global_roles (superadmin, support, auditor)                  │
│  - global_permissions                                            │
│  - users (usuarios globales del sistema)                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    NIVEL TENANT (Business)                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  BUSINESS (Empresa/Tenant)                                │  │
│  │  - business_plans (plan contratado)                       │  │
│  │  - business_modules (módulos activos según plan)          │  │
│  │  - business_roles (roles personalizados)                  │  │
│  │  - business_permissions (permisos según módulos)          │  │
│  │  - branches (sucursales/sedes)                            │  │
│  │                                                            │  │
│  │  USUARIOS EN ESTA EMPRESA:                                │  │
│  │  - user_business (pivote: user ↔ business)                │  │
│  │  - user_business_x_role (roles del user en esta empresa)  │  │
│  │  - user_branches (sucursales asignadas)                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏢 ARQUITECTURA DE MULTITENANCY

### Modelo Adoptado: **Shared Database, Shared Schema**

**Decisión:** Base de datos única con `business_id` como discriminador de tenant.

### Tabla Pivote Principal: `user_business`

```sql
user_business
├── user_id         → Usuario global
├── business_id     → Empresa/tenant
├── primary_branch_id → Sucursal predeterminada (puede trabajar en varias)
├── is_active       → Estado de la membresía
├── joined_at       → Fecha de ingreso
└── last_access_at  → Último acceso (para tracking)
```

**Características:**
- ✅ Un usuario puede pertenecer a **múltiples empresas**
- ✅ Cada usuario tiene **roles diferentes** en cada empresa
- ✅ Control de **sucursales asignadas** por empresa
- ✅ Auditoría completa de membresías

### Identificación del Tenant

**Middleware:** `TenantIdentificationMiddleware`

**Prioridad de extracción:**
1. Header HTTP: `X-Business-ID`
2. Query parameter: `?business_id=`
3. Route parameter: `/api/{business_id}/...`
4. Request body: `business_id`
5. Fallback: `primary_business_id` del usuario

**Validación:**
```php
// Pseudocódigo
1. Extraer business_id del request
2. Verificar: user->hasAccessToBusiness(business_id)
   → Consulta user_business con is_active=true
3. Verificar: business->flag == true (empresa activa)
4. Setear: TenantContext::setCurrentBusiness($business)
5. Actualizar: user_business.last_access_at
6. Agregar al request:
   - current_business_id
   - current_business_name
   - current_user_role
   - is_business_owner
```

---

## 🔑 DECISIONES ARQUITECTÓNICAS CLAVE

### 1. `owner_id` vs `role='superadmin'`

**DECISIÓN:** Mantener AMBOS

#### `business.owner_id` (FK a users.id)
- **Es el FUNDADOR/CREADOR** de la empresa
- Solo puede haber **UNO**
- Privilegios ÚNICOS:
  - ✅ Cerrar/eliminar empresa
  - ✅ Transferir ownership a otro usuario
  - ✅ Cancelar suscripción
  - ✅ Cambiar plan
  - ✅ Responsable legal y de facturación

#### `role='superadmin'` en user_business
- Son **administradores completos** de la empresa
- Puede haber **VARIOS**
- Privilegios:
  - ✅ Gestionar usuarios
  - ✅ Crear/editar/eliminar roles
  - ✅ Ver todos los módulos
  - ✅ Acceso completo a datos
  - ❌ NO pueden acciones críticas (cerrar empresa, transferir ownership)

**Ejemplo:**
```
Business "ABC SAC"
├── owner_id = User 1 (fundador - 60% acciones)
│
└── user_business:
    ├── User 1 → role='superadmin' (el owner también tiene rol)
    ├── User 2 → role='superadmin' (socio - 30% acciones)
    ├── User 3 → role='superadmin' (socio - 10% acciones)
    └── User 4 → role='admin' (empleado)

Solo User 1 puede transferir o cerrar la empresa
```

### 2. Usuarios en Múltiples Sucursales

**DECISIÓN:** ✅ Permitir asignación a MÚLTIPLES sucursales

**Razones:**
- Gerentes regionales supervisan varias sucursales
- Auditores necesitan ver todas las sucursales
- Vendedores móviles trabajan en diferentes ubicaciones
- Contadores consolidan información de todas las sedes

**Implementación:**
```
user_business
└── primary_branch_id (sucursal predeterminada)

user_branches (tabla pivote)
├── user_id
├── branch_id
└── is_primary (marca su sucursal principal)
```

### 3. Eliminación de `gruposRazonesSociales`

**DECISIÓN:** ❌ No implementar (al menos en Fase 1)

**Razones:**
- Agrega complejidad innecesaria
- 95% de empresas tienen UNA razón social
- Puede agregarse después si hay demanda real

**Alternativa para casos especiales:**
- Crear múltiples `business` (Business Perú, Business Chile)
- Agrupar visualmente en frontend
- Compartir usuarios via `user_business`

### 4. Doble Sistema de Permisos

**DECISIÓN:** ✅ Permisos Globales + Permisos por Business

#### Permisos Globales (Platform-level)
```
global_roles → global_permissions
├── platform_superadmin
├── platform_support
└── platform_auditor
```

**Uso:** Administradores de la PLATAFORMA SaaS

#### Permisos por Business (Tenant-level)
```
business_roles → business_permissions
├── Dependen de módulos activos
├── Personalizables por empresa
└── Asignados via user_business_x_role
```

**Uso:** Usuarios normales de empresas

---

## 📦 ESTRUCTURA POR MÓDULOS

### MÓDULO 1: Authentication & Users

**Tablas:**
- `users` - Usuarios globales
- `user_details` - Información personal
- `user_detail_numbers` - Números de contacto
- `refresh_tokens` - Tokens JWT

**Propósito:** Gestión de identidad y autenticación

**Características:**
- Email único global
- Status: active/inactive
- Soporte para múltiples números de contacto
- Refresh tokens para JWT

---

### MÓDULO 2: Multitenancy Core

**Tablas:**
- `business` - Empresas/tenants
- `business_details` - Detalles legales (RUC, dirección, etc)
- `user_business` - **PIVOTE PRINCIPAL**
- `user_business_x_role` - Roles del usuario en la empresa
- `business_x_countries` - Países donde opera

**Propósito:** Núcleo del sistema multi-tenant

**Flujo:**
```
1. Usuario se registra → crea user
2. Usuario crea empresa → crea business
3. Se vinculan → user_business (con role='owner')
4. Se actualiza → business.owner_id = user.id
```

---

### MÓDULO 3: Branches (Sucursales)

**Tablas:**
- `branches` - Sucursales/sedes de una empresa
- `user_branches` - Asignación de usuarios a sucursales

**Propósito:** Control granular por ubicación física

**Uso:**
- Inventarios por sucursal
- Ventas por sucursal
- Costos por sucursal
- Reportes por ubicación

**Características:**
- Un usuario puede trabajar en múltiples sucursales
- `primary_branch_id` en `user_business` define sucursal predeterminada
- `is_primary` en `user_branches` marca sucursal principal

---

### MÓDULO 4: Authorization - Business Level

**Tablas:**
- `business_roles` - Roles personalizados por empresa
- `business_permissions` - Permisos disponibles
- `business_role_x_permissions` - Relación roles-permisos

**Propósito:** Sistema de permisos a nivel tenant

**Jerarquía:**
```
Plan → Módulos → Permisos → Roles → Usuarios
```

**Ejemplo:**
```
Business "XYZ" con Plan Pro tiene módulos: [Almacén, Ventas]

business_permissions:
├── almacen.view
├── almacen.create
├── almacen.edit
├── almacen.delete
├── ventas.view
├── ventas.create
└── ...

business_roles:
├── Almacenero → [almacen.view, almacen.create]
├── Jefe Almacén → [almacen.*]
└── Gerente → [almacen.*, ventas.*]
```

---

### MÓDULO 5: Authorization - Global Level

**Tablas:**
- `global_roles` - Roles de plataforma
- `global_permissions` - Permisos de plataforma
- `user_x_global_roles` - Asignación de roles globales
- `user_global_role_x_permissions` - Permisos de roles globales

**Propósito:** Gestión de la plataforma SaaS

**Casos de uso:**
- Soporte técnico accede a cualquier empresa
- Auditor revisa todas las empresas
- Superadmin gestiona planes y módulos

---

### MÓDULO 6: Plans & Subscriptions (SaaS)

**Tablas:**
- `plans` - Planes disponibles (Free, Basic, Pro, Enterprise)
- `plans_detail` - Características de cada plan
- `plan_x_modules` - **NUEVA v1.2.0** Módulos incluidos en cada plan
- `business_plans` - Plan contratado por empresa
- `business_plan_history` - Historial de cambios

**Propósito:** Sistema de suscripciones con módulos activables

**Flujo:**
```
1. Super Admin crea Plan Pro
2. Asigna módulos al plan → plan_x_modules (plan_id, module_id)
3. Business contrata Plan Pro
4. Se crea business_plans (business_id, plan_id, active=true)
5. Se registra en business_plan_history (tipo='initial')
6. Se copian módulos del plan → business_modules (desde plan_x_modules)
7. Se crean permisos disponibles → business_permissions
```

**Tabla plan_x_modules (Nueva v1.2.0):**
```
Define explícitamente qué módulos incluye cada plan:

Plan Free → plan_x_modules:
  ├── module_id=1 (Inventario básico)

Plan Basic → plan_x_modules:
  ├── module_id=1 (Inventario)
  └── module_id=2 (Almacén)

Plan Pro → plan_x_modules:
  ├── module_id=1 (Inventario)
  ├── module_id=2 (Almacén)
  ├── module_id=3 (POS)
  ├── module_id=4 (Ventas)
  └── module_id=5 (Reportes)

Plan Enterprise → plan_x_modules:
  └── [Todos los módulos + API]
```

**Características de plan:**
```
plans_detail:
- feature: "Usuarios Máx." → value: "10"
- feature: "Almacenamiento" → value: "10GB"
- feature: "Sucursales Máx." → value: "5"
```

---

### MÓDULO 7: Modules System

**Tablas:**
- `modules` - Módulos del ERP (Almacén, Ventas, RRHH, etc)
- `modules_menu` - Estructura de menús
- `business_modules` - Módulos activos por empresa
- `module_permissions` - Relación módulos-permisos

**Propósito:** Sistema modular activable según plan

**Jerarquía:**
```
Plan define módulos disponibles
  ↓
Business activa módulos
  ↓
Módulos definen permisos base
  ↓
Business asigna permisos a roles
  ↓
Usuarios heredan permisos de roles
```

**Ejemplo:**
```
Plan Básico → [Almacén, Inventario]
Plan Pro → [Almacén, Inventario, POS, Ventas, Reportes]
Plan Enterprise → [TODOS + API]

Business con Plan Pro:
└── business_modules:
    ├── Almacén (is_active=true)
    ├── Inventario (is_active=true)
    ├── POS (is_active=true)
    └── Ventas (is_active=true)

Solo permisos de estos módulos están disponibles
```

---

### MÓDULO 8: Geography

**Tablas:**
- `countries` - Catálogo de países
- `business_x_countries` - Países donde opera la empresa

**Propósito:** Gestión de operaciones multinacionales

**Uso:**
- Sucursales por país
- Regulaciones fiscales por país
- Reportes por país

---

## 🔄 FLUJOS DE DATOS PRINCIPALES

### FLUJO 1: Registro de Nueva Empresa

```
1. Usuario se registra
   CREATE users (email, username, password)
   CREATE user_details (names, apellidos, ...)

2. Usuario crea empresa
   CREATE business (name, owner_id=user.id)
   CREATE business_details (tax_id, address, ...)

3. Asignación automática
   CREATE user_business (user_id, business_id, is_active=true)
   CREATE user_business_x_role (user_business_id, business_role_id='owner')

4. Plan por defecto
   CREATE business_plans (business_id, plan_id='Free', active=true)
   CREATE business_plan_history (tipo='initial')

5. Activación de módulos según plan
   CREATE business_modules para cada módulo del plan Free
```

---

### FLUJO 2: Invitar Usuario a Empresa

```
1. Gerente (role='superadmin') invita a juan@email.com

2. Sistema verifica:
   - ¿Existe user con ese email?
     → SÍ: Usar user existente
     → NO: Crear nuevo user

3. Crear relación
   CREATE user_business (user_id, business_id, is_active=true)
   CREATE user_business_x_role (role='member')

4. Asignar a sucursal (opcional)
   UPDATE user_business SET primary_branch_id = X
   CREATE user_branches (user_id, branch_id, is_primary=true)

5. Notificación
   Email a juan@email.com con invitación
```

---

### FLUJO 3: Cambio de Plan

```
1. Owner cambia de Plan Basic → Plan Pro

2. Validación
   - Solo owner_id puede cambiar plan
   - Verificar método de pago

3. Actualización
   UPDATE business_plans SET active=false WHERE id=old_plan
   CREATE business_plans (plan_id='Pro', active=true)
   CREATE business_plan_history (tipo='upgrade')

4. Activar nuevos módulos
   Plan Pro incluye: [POS, Reportes Avanzados]
   CREATE business_modules para nuevos módulos

5. Crear permisos de nuevos módulos
   CREATE business_permissions para permisos de POS y Reportes

6. Los roles existentes pueden ahora usar nuevos permisos
```

---

### FLUJO 4: Request con Multitenancy

```
1. Usuario hace login
   JWT incluye: user_id, email, all_businesses[]

2. Frontend muestra selector de empresas
   Usuario selecciona "Business ABC"

3. Request a API
   Headers:
   - Authorization: Bearer <jwt_token>
   - X-Business-ID: 123

4. Middleware TenantIdentificationMiddleware
   a) Extrae business_id = 123
   b) Valida user_business (user_id=X, business_id=123, is_active=true)
   c) Valida business.flag = true
   d) Setea TenantContext::setCurrentBusiness(123)
   e) Obtiene role del usuario en esta empresa
   f) Agrega al request: current_business_id, current_user_role

5. Query con Tenant Scope
   SELECT * FROM products WHERE business_id = 123

6. Respuesta incluye solo datos del tenant actual
```

---

## 🔐 JERARQUÍA DE PERMISOS

### Nivel 1: Plataforma (Global)

```
SUPERADMIN DE PLATAFORMA (global_roles)
├── Gestionar todos los businesses
├── Crear/editar/eliminar planes
├── Activar/desactivar módulos globalmente
├── Ver métricas de toda la plataforma
└── Soporte técnico a cualquier empresa
```

### Nivel 2: Business Owner (owner_id)

```
FUNDADOR DE EMPRESA (business.owner_id)
├── Transferir ownership
├── Cerrar/eliminar empresa
├── Cambiar plan
├── Cancelar suscripción
├── Ver historial de facturación
└── TODO lo que puede role='superadmin'
```

### Nivel 3: Business Superadmin (role en user_business)

```
SUPERADMIN DE EMPRESA (role='superadmin')
├── Gestionar usuarios
├── Crear/editar/eliminar roles
├── Asignar permisos a roles
├── Ver todos los módulos activos
├── Acceso completo a datos de la empresa
└── ❌ NO puede: cerrar empresa, transferir ownership, cambiar plan
```

### Nivel 4: Business Admin/Manager/Member

```
ROLES PERSONALIZADOS (business_roles)
├── Admin: Gestión completa de módulos asignados
├── Manager: Gestión parcial + reportes
├── Member: Solo acciones básicas
└── Roles custom: Almacenero, Vendedor, Contador, etc
```

---

## 📊 SISTEMA DE PLANES Y MÓDULOS

### Jerarquía Completa (Actualizada v1.2.0)

```
CAPA 1: PLANES (Define QUÉ módulos)
  └── plans (id, name, price, duration)

CAPA 1.5: PLAN_X_MODULES (v1.2.0 - Módulos del plan)
  └── plan_x_modules (plan_id, module_id)
  └── Ejemplo:
      ├── Plan Free → [Inventario básico]
      ├── Plan Basic → [Inventario, Almacén, Cardex]
      ├── Plan Pro → [Inventario, Almacén, POS, Ventas, Reportes]
      └── Plan Enterprise → [TODOS + API]

CAPA 2: BUSINESS_PLANS (Plan contratado)
  └── business_plans (business_id, plan_id, active=true)

CAPA 3: BUSINESS_MODULES (Módulos activos copiados del plan)
  └── business_modules (business_id, module_id, is_active=true)
  └── Se copian desde plan_x_modules al contratar

CAPA 4: MODULE_PERMISSIONS (Permisos por módulo)
  └── module_permissions (module_id, business_permission_id)

CAPA 5: BUSINESS_PERMISSIONS (Permisos disponibles)
  └── business_permissions (business_id, name)

CAPA 6: BUSINESS_ROLES (Roles personalizados)
  └── business_roles (business_id, name)

CAPA 7: BUSINESS_ROLE_X_PERMISSIONS (Asignación)
  └── business_role_x_permissions (role_id, permission_id)

CAPA 8: USER_BUSINESS_X_ROLE (Usuario final)
  └── user_business_x_role (user_business_id, business_role_id)
```

### Ejemplo Completo

```
1. Super Admin crea Plan Pro
   → plans: {id: 3, name: 'Pro', price: 99.00}

2. Super Admin asigna módulos al Plan Pro
   → plan_x_modules:
      ├── (plan_id=3, module_id=1) → Inventario
      ├── (plan_id=3, module_id=2) → Almacén
      ├── (plan_id=3, module_id=3) → POS
      └── (plan_id=3, module_id=4) → Ventas

3. Business "ABC SAC" contrata Plan Pro
   → business_plans: {business_id=5, plan_id=3, active=true}

4. Sistema copia módulos del plan a la empresa
   → business_modules (copiados desde plan_x_modules):
      ├── (business_id=5, module_id=1, is_active=true) → Inventario
      ├── (business_id=5, module_id=2, is_active=true) → Almacén
      ├── (business_id=5, module_id=3, is_active=true) → POS
      └── (business_id=5, module_id=4, is_active=true) → Ventas

3. Módulo "Almacén" define permisos:
   - almacen.view
   - almacen.create
   - almacen.edit
   - almacen.delete
   - almacen.transfer

4. Business crea rol "Almacenero"
   → business_roles: {name: 'Almacenero'}

5. Asigna permisos al rol
   → business_role_x_permissions:
      - almacen.view
      - almacen.create

6. Usuario Juan obtiene rol "Almacenero"
   → user_business_x_role: (juan, business_abc, almacenero)

7. Juan puede:
   ✅ Ver almacén
   ✅ Crear productos
   ❌ Editar productos (no tiene permiso)
   ❌ Eliminar productos (no tiene permiso)
```

---

## 🎯 CASOS DE USO

### Caso 1: Usuario Multi-Empresa

**Escenario:** María es contadora freelance

```
users
└── id: 10, email: maria@email.com

user_business
├── user_id=10, business_id=1 (Empresa A)
│   └── role='admin'
│   └── primary_branch_id=1 (Sucursal Centro)
│
├── user_id=10, business_id=5 (Empresa B)
│   └── role='superadmin'
│   └── primary_branch_id=10 (Oficina Principal)
│
└── user_id=10, business_id=8 (Empresa C)
    └── role='member'
    └── primary_branch_id=15 (Sede Norte)

María usa el selector de empresas para cambiar contexto
```

### Caso 2: Gerente Multi-Sucursal

**Escenario:** Carlos supervisa 3 sucursales

```
user_business
└── user_id=20, business_id=1, role='manager'
    └── primary_branch_id=2 (Sucursal Norte es su base)

user_branches
├── user_id=20, branch_id=2 (Norte) → is_primary=true
├── user_id=20, branch_id=3 (Sur) → is_primary=false
└── user_id=20, branch_id=4 (Este) → is_primary=false

Carlos puede:
- Ver inventarios de las 3 sucursales
- Generar reportes consolidados
- Transferir productos entre sucursales
- Su sucursal predeterminada es Norte
```

### Caso 3: Upgrade de Plan

**Escenario:** Empresa crece y necesita más módulos

```
ANTES (Plan Basic):
└── business_modules: [Almacén, Inventario]
    └── Permisos disponibles: solo de Almacén e Inventario

UPGRADE a Plan Pro:
└── business_modules: [Almacén, Inventario, POS, Ventas, Reportes]
    └── Nuevos permisos disponibles: pos.*, ventas.*, reportes.*

DESPUÉS:
└── Gerente puede asignar nuevos permisos a roles existentes
    └── Rol "Vendedor" ahora puede usar POS
```

---

## 📏 CONVENCIONES Y ESTÁNDARES

### Nomenclatura de Tablas

- Plural en inglés: `users`, `businesses`, `branches`
- Snake_case: `user_business`, `business_plans`
- Tablas pivote: `{tabla1}_x_{tabla2}` (user_x_global_roles)
- Tablas de relación: `{tabla}_x_{tabla}` o `{tabla}_{tabla}`

### Campos de Auditoría

**Estándar:**
```sql
created_at timestamp
updated_at timestamp
created_by bigint (FK a users.id)
updated_by bigint (FK a users.id)
deleted_at timestamp (soft delete)
deleted_by bigint (FK a users.id)
```

**Aplicar en:** Todas las tablas transaccionales (business, roles, permissions, etc)

### Campos de Estado

- `flag`: boolean para active/inactive
- `is_active`: boolean para estados específicos
- `status`: enum para estados múltiples

### Índices

**Reglas:**
1. PK siempre tiene índice automático
2. FK siempre debe tener índice
3. Unique constraints para combinaciones (user_id, business_id)
4. Índices compuestos para queries frecuentes: (business_id, is_active)

---

## 🚀 PRÓXIMOS PASOS

### Fase 1: Implementación Base (ACTUAL)
- ✅ Modelado completo
- ⏳ Migrar a dbdiagram.io
- ⏳ Crear migraciones Laravel
- ⏳ Implementar modelos Eloquent
- ⏳ Implementar middleware TenantIdentificationMiddleware
- ⏳ Testing de multitenancy

### Fase 2: Módulos Transaccionales
- Productos/Servicios
- Inventario
- Ventas
- Compras
- Clientes/Proveedores

### Fase 3: Módulos Avanzados
- Contabilidad
- RRHH
- Reportes avanzados
- API externa

### Fase 4: Optimizaciones
- Caché de permisos
- Índices adicionales según uso real
- Particionamiento si es necesario
- Query optimization

---

## 📝 NOTAS IMPORTANTES

### Decisiones Pendientes de Validar

1. **Soft deletes:** ¿Aplicar en todas las tablas o solo algunas?
2. **Auditoría:** ¿Guardar logs de cambios en tabla separada?
3. **Caché:** ¿Redis para permisos y relaciones user-business?

### Riesgos Identificados

1. **Performance:** Joins múltiples para validar permisos
   - Mitigación: Caché de permisos por usuario-business
   
2. **Complejidad:** Muchas tablas de relación
   - Mitigación: Documentación clara y helpers en modelos
   
3. **Escalabilidad:** Crecimiento de user_business con muchos usuarios
   - Mitigación: Índices optimizados + paginación

---

## 🔍 DIAGRAMA ER SIMPLIFICADO

```
users ──┬──< user_business >──┬── business ──< business_plans >── plans
        │                      │
        │                      ├── business_modules ──> modules
        │                      │
        │                      ├── business_roles ──< business_role_x_permissions >── business_permissions
        │                      │
        │                      └── branches ──< user_branches >── users
        │
        ├──< user_details
        │
        ├──< user_x_global_roles >──┬── global_roles
        │                            │
        └──< refresh_tokens          └──< user_global_role_x_permissions >── global_permissions
```

---

## 📚 REFERENCIAS

- Laravel Multi-tenancy Patterns
- SaaS Database Design Best Practices
- RBAC (Role-Based Access Control) Implementation
- JWT Authentication in Multi-tenant Systems

---

**Fin de la documentación**

**Próxima revisión:** Después de implementar migraciones  
**Responsable:** KuroNeko  
**Versión:** 1.0.0
