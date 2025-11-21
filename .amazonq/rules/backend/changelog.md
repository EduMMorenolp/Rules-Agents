# AccessAI Backend - Changelog Rules (OBLIGATORIO)

## 🚨 **REGLA CRÍTICA: CHANGELOG Obligatorio**

### ✅ **SIEMPRE actualizar CHANGELOG.md cuando:**
- Se implemente **nueva funcionalidad**
- Se modifique **funcionalidad existente**
- Se corrija **bug o error**
- Se refactorice **código significativo**
- Se agregue **nuevo endpoint**
- Se modifique **estructura de BD**

### ❌ **NUNCA hacer commit sin actualizar CHANGELOG**
```bash
# ❌ PROHIBIDO - Commit sin changelog
git commit -m "feat: nuevo endpoint de comunidades"

# ✅ CORRECTO - Actualizar changelog primero
# 1. Editar CHANGELOG.md
# 2. Luego hacer commit
git commit -m "feat: nuevo endpoint de comunidades"
```

## 📋 **Formato Obligatorio**

### **Estructura del CHANGELOG.md**
```markdown
# Changelog - AccessAI Backend

## [Unreleased]

### Added
- Nueva funcionalidad implementada

### Changed  
- Funcionalidad modificada

### Fixed
- Bugs corregidos

### Security
- Mejoras de seguridad

## [1.0.1] - 2024-01-20
### Added
- Endpoint POST /devices
- Sistema de alertas en tiempo real
```

### **Categorías Obligatorias**
- **Added**: Nueva funcionalidad
- **Changed**: Cambios en funcionalidad existente
- **Fixed**: Corrección de bugs
- **Security**: Mejoras de seguridad
- **Deprecated**: Funcionalidad obsoleta
- **Removed**: Funcionalidad eliminada

## 🔄 **Workflow Obligatorio**

### **Proceso de Desarrollo**
1. **Implementar funcionalidad**
2. **Actualizar CHANGELOG.md** en sección `[Unreleased]`
3. **Hacer commit** con cambios + changelog
4. **Incrementar versión** cuando sea necesario

### **Ejemplo de Actualización**
```markdown
## [Unreleased]

### Added
- Endpoint POST /api/v1/devices para registrar dispositivos [28/10/2025]
- Validación de permisos de administrador [28/10/2025]
- Alertas automáticas para accesos no autorizados [28/10/2025]

### Changed
- Refactorización de alertService con Database Manager [28/10/2025]
- Optimización de queries con Promise.all [28/10/2025]

### Fixed
- Corrección de timestamps en snake_case para queries [28/10/2025]
- Manejo de errores en uploadToS3 [28/10/2025]
```

## 🚀 **Scripts de Versionado**

### **Comandos con Changelog**
```bash
# 1. Actualizar CHANGELOG.md manualmente
# 2. Incrementar versión
npm run version:patch    # Para fixes
npm run version:minor    # Para features
npm run version:major    # Para breaking changes

# 3. Crear release con tag
npm run version:tag
```

### **Verificación Pre-Commit**
```bash
# Verificar que CHANGELOG fue actualizado
git diff --name-only | grep CHANGELOG.md
```

## 📝 **Ejemplos por Tipo de Cambio**

### **Nueva Funcionalidad**
```markdown
### Added
- Endpoint POST /api/v1/visitors para registrar visitantes temporales [28/10/2025]
- Sistema de generación de QR para acceso [28/10/2025]
- Validación de horarios de acceso por zona [28/10/2025]
```

### **Modificación Existente**
```markdown
### Changed
- Refactorización de authService para usar Database Manager [28/10/2025]
- Optimización de queries de alertas con includes específicos [28/10/2025]
- Mejora en formateo de respuestas con responseFormatter [28/10/2025]
```

### **Corrección de Bugs**
```markdown
### Fixed
- Corrección de error en validación de UUID en deviceController [28/10/2025]
- Fix en manejo de streams de cámara en aiController [28/10/2025]
- Resolución de problema con timestamps en queries de ordenamiento [28/10/2025]
```

### **Seguridad**
```markdown
### Security
- Implementación de rate limiting en endpoints de autenticación [28/10/2025]
- Validación mejorada de streams de cámara [28/10/2025]
- Sanitización de entrada en endpoints de IA [28/10/2025]
```

## 🎯 **Checklist Pre-Commit**

- [ ] ✅ CHANGELOG.md actualizado en sección `[Unreleased]`
- [ ] ✅ Cambios categorizados correctamente (Added/Changed/Fixed/Security)
- [ ] ✅ Descripción clara y específica de los cambios
- [ ] ✅ Endpoints nuevos documentados con método HTTP
- [ ] ✅ Refactorizaciones importantes mencionadas
- [ ] ✅ Cada cambio incluye fecha en formato [DD/MM/YYYY]

## 🚨 **Recordatorio Automático**

### **Mensaje para el Desarrollador**
```
⚠️  RECORDATORIO OBLIGATORIO:
📝 ¿Actualizaste CHANGELOG.md con los cambios implementados?
🔄 Sección [Unreleased] debe incluir:
   - Added: Nueva funcionalidad
   - Changed: Modificaciones
   - Fixed: Correcciones
   
✅ Solo después de actualizar CHANGELOG, procede con el commit
```

Esta regla garantiza documentación completa y trazabilidad de todos los cambios en el backend AccessAI.
