# 🎉 CAMBIOS VERSIÓN 1.1.0 - RESUMEN EJECUTIVO

**Fecha:** 2025-12-14  
**Cambio Principal:** Separación de entidades de negocio (customers, employees) vs usuarios del sistema

---

## 📋 NUEVAS TABLAS AGREGADAS (3)

### 1️⃣ **`document_types`** (Catálogo)

```sql
CREATE TABLE document_types (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL UNIQUE,  -- 'DNI', 'RUC', 'Carnet Extranjería', etc
    code VARCHAR(10) NOT NULL UNIQUE,   -- Código SUNAT: '1', '6', '4', etc
    max_length INT NOT NULL,             -- 11 para RUC, 8 para DNI
    min_length INT NOT NULL,             -- Igual a max_length normalmente
    country_id BIGINT NULL,              -- FK a countries (para específico por país)
    flag BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

**Propósito:**
- Catálogo centralizado de tipos de documento
- Validaciones automáticas de longitud
- Soporte para códigos SUNAT (Perú) u otros estándares

**Ejemplos de datos:**
```sql
INSERT INTO document_types (name, code, max_length, min_length, country_id) VALUES
('DNI', '1', 8, 8, 1),           -- Perú
('RUC', '6', 11, 11, 1),         -- Perú
('Carnet de Extranjería', '4', 12, 12, 1),
('Pasaporte', '7', 12, 12, NULL); -- Internacional
```

---

### 2️⃣ **`customers`** (CRM Module)

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    business_id BIGINT NOT NULL,         -- TENANT
    
    -- Info básica
    customer_code VARCHAR(50) NULL,      -- Código interno
    name VARCHAR(200) NOT NULL,
    
    -- Documento
    document_type_id BIGINT NULL,        -- FK a document_types
    document_number VARCHAR(50) NULL,
    
    -- Contacto (OPCIONALES - pueden no tener)
    email VARCHAR(255) NULL,
    phone VARCHAR(20) NULL,
    address TEXT NULL,
    
    -- Negocio
    credit_limit DECIMAL(12,2) DEFAULT 0.00,
    current_debt DECIMAL(12,2) DEFAULT 0.00,
    
    -- Login OPCIONAL
    user_id BIGINT NULL,                 -- FK a users (NULL = sin acceso)
    
    flag BOOLEAN DEFAULT TRUE,
    
    -- Auditoría completa
    created_at, updated_at, created_by, updated_by, deleted_at, deleted_by
    
    FOREIGN KEY (business_id) REFERENCES business(id) ON DELETE CASCADE,
    FOREIGN KEY (document_type_id) REFERENCES document_types(id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);
```

**Propósito:**
- Clientes de una empresa (NO tienen login por defecto)
- Información comercial (crédito, deuda)
- Email y phone OPCIONALES

**Casos de uso:**
```
Cliente sin acceso:
├── user_id = NULL
└── Solo información en el sistema

Cliente con acceso al portal:
├── user_id = 10
├── Puede ver sus compras/deudas
└── Se le creó usuario para acceso
```

---

### 3️⃣ **`employees`** (HR Module)

```sql
CREATE TABLE employees (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    business_id BIGINT NOT NULL,         -- TENANT
    branch_id BIGINT NULL,               -- Sucursal donde trabaja
    
    -- Info básica
    employee_code VARCHAR(50) NULL,      -- Código de empleado
    names VARCHAR(100) NOT NULL,
    apellido_paterno VARCHAR(100) NOT NULL,
    apellido_materno VARCHAR(100) NULL,
    
    -- Documento
    document_type_id BIGINT NULL,
    document_number VARCHAR(50) NULL,
    
    -- Contacto (OPCIONALES)
    email VARCHAR(255) NULL,
    phone VARCHAR(20) NULL,
    address TEXT NULL,
    
    -- Información laboral
    hire_date DATE NULL,
    termination_date DATE NULL,
    position VARCHAR(100) NULL,          -- Almacenero, Cajero, etc
    salary DECIMAL(10,2) NULL,
    
    -- Login OPCIONAL
    user_id BIGINT NULL,                 -- FK a users (NULL = sin acceso)
    
    flag BOOLEAN DEFAULT TRUE,
    
    -- Auditoría completa
    created_at, updated_at, created_by, updated_by, deleted_at, deleted_by
    
    FOREIGN KEY (business_id) REFERENCES business(id) ON DELETE CASCADE,
    FOREIGN KEY (branch_id) REFERENCES branches(id) ON DELETE SET NULL,
    FOREIGN KEY (document_type_id) REFERENCES document_types(id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);
```

**Propósito:**
- Empleados de planilla (NO tienen login por defecto)
- Información de RRHH (salario, fechas, cargo)
- Email OPCIONAL

**Casos de uso:**
```
Empleado sin acceso:
├── user_id = NULL
└── Solo en planilla/reportes

Empleado con acceso al ERP:
├── user_id = 20
├── Puede usar el sistema
└── Tiene rol asignado vía user_business
```

---

## 🔄 CAMBIOS EN TABLAS EXISTENTES

### **`business_details`** - MODIFICADO

**ANTES:**
```sql
tax_id VARCHAR(50) NULL  -- RUC, RFC, CUIT, etc
```

**AHORA:**
```sql
document_type_id BIGINT NULL,     -- FK a document_types
document_number VARCHAR(50) NULL  -- Número del documento
```

**Razón del cambio:**
- Normalización de datos
- Validaciones consistentes
- Soporte multi-país

---

## 📊 COMPARACIÓN: ANTES vs AHORA

### ❌ ANTES (Versión 1.0.0)

```
users (para TODO)
├── Usuarios del sistema ✅
├── Clientes ❌ (mezclado, confuso)
└── Empleados ❌ (mezclado, confuso)

Problemas:
- Email required pero clientes pueden no tener
- Password required pero empleados no se loguean
- Tabla users hinchada con registros sin login
```

### ✅ AHORA (Versión 1.1.0)

```
users (SOLO login)
├── Usuarios del sistema ✅
└── Email y password SIEMPRE requeridos

customers (CRM)
├── Clientes de la empresa
├── Email OPCIONAL
├── user_id NULL por defecto
└── Si necesita portal → se crea user

employees (HR)
├── Empleados de planilla
├── Email OPCIONAL
├── user_id NULL por defecto
└── Si necesita ERP → se crea user

document_types (Catálogo)
├── DNI, RUC, CE, Pasaporte
└── Validaciones de longitud
```

---

## 🎯 FLUJOS DE TRABAJO

### FLUJO 1: Registrar Cliente SIN acceso

```php
// 1. Crear cliente
$customer = Customer::create([
    'business_id' => $currentBusiness,
    'name' => 'Juan Pérez',
    'document_type_id' => 1, // DNI
    'document_number' => '12345678',
    'email' => null, // Puede no tener
    'user_id' => null // SIN acceso
]);
```

### FLUJO 2: Dar acceso a Cliente existente

```php
// 1. Cliente existe sin login
$customer = Customer::find(5);

// 2. Crear usuario
$user = User::create([
    'email' => 'juan@email.com',
    'username' => 'juan.perez',
    'password' => Hash::make('temp_password')
]);

// 3. Vincular
$customer->update(['user_id' => $user->id]);

// 4. Dar acceso a la empresa
UserBusiness::create([
    'user_id' => $user->id,
    'business_id' => $customer->business_id,
]);

// 5. Asignar rol
UserBusinessXRole::create([
    'user_business_id' => $userBusiness->id,
    'business_role_id' => $customerRoleId // rol 'customer'
]);
```

### FLUJO 3: Registrar Empleado CON acceso

```php
DB::transaction(function() {
    // 1. Crear usuario
    $user = User::create([
        'email' => 'maria@empresa.com',
        'username' => 'maria.garcia',
        'password' => Hash::make('password')
    ]);
    
    // 2. Crear empleado
    $employee = Employee::create([
        'business_id' => $currentBusiness,
        'branch_id' => 2,
        'names' => 'María',
        'apellido_paterno' => 'García',
        'document_type_id' => 1,
        'document_number' => '87654321',
        'position' => 'Vendedora',
        'salary' => 1500,
        'user_id' => $user->id // CON acceso
    ]);
    
    // 3. Dar acceso a la empresa
    UserBusiness::create([
        'user_id' => $user->id,
        'business_id' => $currentBusiness,
        'primary_branch_id' => 2
    ]);
    
    // 4. Asignar rol
    UserBusinessXRole::create([...]);
});
```

---

## 📋 CHECKLIST DE IMPLEMENTACIÓN

### Paso 1: Crear migraciones (en orden)

```
1. create_document_types_table
2. create_customers_table
3. create_employees_table
4. add_document_type_to_business_details
```

### Paso 2: Seeders

```
1. DocumentTypeSeeder (DNI, RUC, CE, Pasaporte)
2. (Opcional) CustomerSeeder para testing
3. (Opcional) EmployeeSeeder para testing
```

### Paso 3: Modelos Eloquent

```
1. DocumentType model
2. Customer model con relaciones
3. Employee model con relaciones
4. Actualizar Business model
```

### Paso 4: Actualizar dbdiagram.io

```
1. Importar modelado_erp_completo.dbml (v1.1.0)
2. Exportar diagrama PNG actualizado
3. Commit en repo GitHub
```

---

## 🎯 VENTAJAS DE ESTE ENFOQUE

### ✅ Ventajas

1. **Claridad:** `users` solo para login, `customers` y `employees` para negocio
2. **Flexibilidad:** Email opcional en customers/employees
3. **Promoción fácil:** user_id nullable permite dar acceso después
4. **Validaciones:** document_types centraliza lógica de validación
5. **Performance:** Queries más eficientes (menos NULL checks)
6. **Escalabilidad:** Tablas específicas para cada tipo de entidad

### ⚠️ Consideraciones

1. **Joins adicionales:** Queries pueden necesitar más JOINs
2. **Sincronización:** Si employee se convierte en user, mantener datos sincronizados
3. **Duplicación:** Nombre y documento en employee Y en user_details

---

## 📊 ESTADÍSTICAS ACTUALIZADAS

| Componente | v1.0.0 | v1.1.0 | Cambio |
|------------|--------|--------|--------|
| **Tablas** | 33 | 36 | +3 |
| **Relaciones** | 45+ | 51+ | +6 |
| **Módulos** | 8 | 10 | +2 (CRM, HR) |
| **TableGroups** | 8 | 10 | +2 |

---

## 🚀 PRÓXIMOS PASOS

1. ✅ Revisar y aprobar cambios
2. ⏳ Crear migraciones Laravel
3. ⏳ Implementar modelos Eloquent
4. ⏳ Crear seeders de document_types
5. ⏳ Actualizar repo GitHub
6. ⏳ Testing de flujos

---

**Versión:** 1.1.0  
**Estado:** ✅ Listo para implementación  
**Breaking Changes:** ❌ No (cambios son aditivos)
