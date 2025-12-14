# 🚀 GUÍA RÁPIDA - CÓMO USAR ESTOS ARCHIVOS

## 📁 ARCHIVOS ENTREGADOS

```
1. modelado_erp_completo.dbml
   → Script DBML completo para dbdiagram.io
   → Incluye TODAS las tablas, relaciones, índices
   → Listo para importar y visualizar

2. DOCUMENTACION_MODELADO.md
   → Documentación completa del diseño
   → Explicación de decisiones arquitectónicas
   → Casos de uso y flujos de datos
   → Para consulta y revisión futura

3. GUIA_USO.md (este archivo)
   → Instrucciones de uso
```

---

## 🎨 PASO 1: IMPORTAR A DBDIAGRAM.IO

### Opción A: Nueva Página (Recomendado)

1. Ir a https://dbdiagram.io/d
2. Click en "New Diagram"
3. Borrar todo el contenido del editor
4. Copiar TODO el contenido de `modelado_erp_completo.dbml`
5. Pegar en el editor
6. ¡Listo! El diagrama se genera automáticamente

### Opción B: Importar Archivo

1. Ir a https://dbdiagram.io/d
2. Click en menú hamburguesa (≡) → "Import"
3. Seleccionar "From file"
4. Upload `modelado_erp_completo.dbml`
5. ¡Listo!

---

## 🎯 PASO 2: VISUALIZAR Y ORGANIZAR

### Organización Visual

El script ya incluye **TableGroups** para organizar visualmente:

```
📦 authentication
├── users
├── user_details
├── user_detail_numbers
└── refresh_tokens

📦 multitenancy_core
├── business
├── business_details
├── user_business
└── user_business_x_role

📦 branches
├── branches
└── user_branches

... y más grupos
```

### Personalización en dbdiagram.io

**Cambiar colores de grupos:**
```dbml
TableGroup authentication {
  users
  user_details
  
  // Agregar color
  [headercolor: #3498db]
}
```

**Agregar notas visuales:**
```dbml
Table users {
  ...
  Note: '''
    Esta es una nota que aparecerá
    en el diagrama visual
  '''
}
```

**Cambiar colores de tablas:**
```dbml
Table business [headercolor: #e74c3c] {
  id bigint [pk]
  ...
}
```

---

## 📤 PASO 3: EXPORTAR DIAGRAMA

### Exportar como Imagen

1. En dbdiagram.io → Click "Export"
2. Seleccionar formato:
   - **PNG**: Para documentación
   - **SVG**: Para edición vectorial
   - **PDF**: Para presentaciones

### Exportar SQL

1. Click "Export" → "MySQL"
2. Descarga `schema.sql`
3. **⚠️ REVISAR ANTES DE EJECUTAR**
   - dbdiagram.io puede no generar exactamente como Laravel migrations
   - Úsalo como referencia, NO como fuente definitiva

---

## 📝 PASO 4: VERSIONADO EN GITHUB

### Estructura de Repositorio Recomendada

```
modelado-erp/
│
├── README.md
├── CHANGELOG.md
│
├── dbml/
│   ├── modelado_erp_completo.dbml
│   └── versiones/
│       ├── v1.0.0_2025-12-14.dbml
│       └── v1.1.0_2025-XX-XX.dbml
│
├── docs/
│   ├── DOCUMENTACION_MODELADO.md
│   ├── DECISIONES_ARQUITECTONICAS.md
│   └── CASOS_DE_USO.md
│
├── exports/
│   ├── diagrams/
│   │   ├── full_diagram_v1.0.0.png
│   │   └── full_diagram_v1.0.0.pdf
│   │
│   └── sql/
│       ├── schema_v1.0.0.sql
│       └── migrations/ (referencia)
│
└── queries/
    ├── common_queries.sql
    └── performance_analysis.sql
```

### Comandos Git

```bash
# Inicializar repo
git init
git add .
git commit -m "Initial commit: ERP Database Design v1.0.0"

# Crear tag de versión
git tag -a v1.0.0 -m "Version 1.0.0 - Base multitenancy design"

# Subir a GitHub
git remote add origin https://github.com/tu-usuario/modelado-erp.git
git branch -M main
git push -u origin main
git push origin v1.0.0
```

### README.md del Repo

```markdown
# 📊 ERP Multi-Tenant - Database Design

Sistema de base de datos para ERP multi-tenant con Laravel.

## 🎯 Características

- ✅ Multitenancy con user_business pivot
- ✅ Sistema de planes y módulos SaaS
- ✅ Permisos granulares (global + business)
- ✅ Sucursales/branches
- ✅ Auditoría completa

## 📁 Estructura

- `/dbml` - Scripts DBML para dbdiagram.io
- `/docs` - Documentación completa
- `/exports` - Diagramas y SQL exportados

## 🚀 Uso Rápido

1. Importar `dbml/modelado_erp_completo.dbml` en https://dbdiagram.io
2. Leer `docs/DOCUMENTACION_MODELADO.md` para entender el diseño
3. Usar diagramas en `/exports/diagrams` para presentaciones

## 📌 Versión Actual

**v1.0.0** - 2025-12-14
```

---

## 🔄 PASO 5: MANTENER ACTUALIZADO

### Cuando Agregues/Modifiques Tablas

1. **Actualizar DBML:**
   ```dbml
   // En modelado_erp_completo.dbml
   
   Table nueva_tabla {
     id bigint [pk, increment]
     business_id bigint [not null, ref: > business.id]
     ...
   }
   ```

2. **Actualizar Documentación:**
   ```markdown
   // En DOCUMENTACION_MODELADO.md
   
   ### MÓDULO X: Nueva Funcionalidad
   
   **Tablas:**
   - nueva_tabla - Descripción
   
   **Propósito:** ...
   ```

3. **Crear Nueva Versión:**
   ```bash
   # Guardar versión anterior
   cp dbml/modelado_erp_completo.dbml dbml/versiones/v1.0.0_2025-12-14.dbml
   
   # Editar versión actual
   # ... hacer cambios ...
   
   # Commit
   git add .
   git commit -m "Add: Nueva tabla para funcionalidad X"
   git tag -a v1.1.0 -m "Added nueva_tabla"
   ```

4. **Actualizar CHANGELOG:**
   ```markdown
   ## [1.1.0] - 2025-12-XX
   
   ### Added
   - Tabla `nueva_tabla` para funcionalidad X
   - Campo `nuevo_campo` en tabla `business`
   
   ### Changed
   - Índice compuesto en `user_business`
   
   ### Removed
   - Campo deprecated `old_field`
   ```

---

## 🎓 PASO 6: DOCUMENTAR CAMBIOS

### Template para Decisiones Arquitectónicas

Cuando tomes una decisión importante, documéntala:

```markdown
## DECISIÓN: [Nombre de la decisión]

**Fecha:** 2025-12-XX
**Contexto:** ¿Por qué necesitamos decidir esto?
**Opciones consideradas:**
1. Opción A - Pros y contras
2. Opción B - Pros y contras

**Decisión:** Elegimos Opción B

**Razones:**
- Razón 1
- Razón 2

**Consecuencias:**
- Impacto en X
- Requiere cambiar Y

**Responsable:** KuroNeko
```

Guarda estas decisiones en `docs/DECISIONES_ARQUITECTONICAS.md`

---

## 🔍 PASO 7: CONSULTAR DOCUMENTACIÓN

### Cuándo Revisar el Modelado

**Antes de desarrollar nueva funcionalidad:**
1. Revisar `DOCUMENTACION_MODELADO.md` → Sección del módulo
2. Verificar en dbdiagram.io → Relaciones visuales
3. Leer decisiones arquitectónicas → Entender el "por qué"

**Cuando vuelvas después de tiempo:**
1. Leer `README.md` → Visión general
2. Ver diagrama PNG exportado → Estructura rápida
3. Revisar `CHANGELOG.md` → Qué cambió
4. Leer `DOCUMENTACION_MODELADO.md` → Detalles completos

**Al incorporar nuevo desarrollador:**
1. `README.md` → Intro
2. Diagrama visual → Comprensión gráfica
3. `CASOS_DE_USO.md` → Ejemplos prácticos
4. `DOCUMENTACION_MODELADO.md` → Referencia completa

---

## 🛠️ HERRAMIENTAS ÚTILES

### Para Editar DBML
- **VS Code** con extensión "DBML Syntax"
- **Sublime Text** con syntax highlighting
- **dbdiagram.io** directamente en navegador

### Para Versionar
- **Git** + **GitHub**
- Tags semánticos: v1.0.0, v1.1.0, v2.0.0

### Para Diagramas
- **dbdiagram.io** - Generación automática
- **draw.io** - Diagramas manuales adicionales
- **Lucidchart** - Diagramas de flujo

---

## ⚡ ATAJOS Y TIPS

### En dbdiagram.io

**Navegación:**
- `Ctrl + F`: Buscar tabla
- `Ctrl + /`: Toggle comentarios
- Zoom: Rueda del mouse
- Pan: Arrastrar con click derecho

**Exportar rápido:**
- `Ctrl + E`: Abrir menú de exportación

**Validar:**
- Panel derecho muestra errores en tiempo real
- Líneas rojas = syntax error

### Mantenimiento del Repo

```bash
# Ver cambios en modelado
git diff dbml/modelado_erp_completo.dbml

# Ver versiones anteriores
git log --oneline dbml/

# Comparar versiones
git diff v1.0.0 v1.1.0 -- dbml/modelado_erp_completo.dbml

# Revertir a versión anterior
git checkout v1.0.0 -- dbml/modelado_erp_completo.dbml
```

---

## 🎯 CHECKLIST DE CALIDAD

Antes de commitear cambios en el modelado:

- [ ] DBML es válido (no errores en dbdiagram.io)
- [ ] Documentación actualizada
- [ ] CHANGELOG actualizado
- [ ] Diagrama exportado (PNG/PDF)
- [ ] Decisiones arquitectónicas documentadas (si aplica)
- [ ] Tag de versión creado
- [ ] README actualizado (si cambió estructura)

---

## 📞 SOPORTE

Si tienes dudas sobre:
- **DBML syntax:** https://dbml.dbdiagram.io/docs/
- **dbdiagram.io:** https://dbdiagram.io/docs
- **Git versionado:** https://git-scm.com/docs

---

## 🎉 ¡LISTO!

Ahora tienes:
✅ Modelado completo en DBML
✅ Documentación exhaustiva
✅ Guía de uso y mantenimiento
✅ Estructura para versionado

**Siguiente paso:** Importar a dbdiagram.io y crear tu repo en GitHub

¡Éxito con tu proyecto! 🚀
