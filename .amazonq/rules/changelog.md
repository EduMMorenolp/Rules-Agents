# AccessAI AI Service - Changelog Rules (OBLIGATORIO)

## 🚨 **REGLA CRÍTICA: CHANGELOG Obligatorio**

### ✅ **SIEMPRE actualizar CHANGELOG.md cuando:**
- Se implemente **nueva funcionalidad de IA**
- Se modifique **sistema de reconocimiento facial**
- Se agregue **nuevo modelo de detección**
- Se corrija **bug en procesamiento de video**
- Se refactorice **código del servicio IA**
- Se modifique **integración con backend**

### ❌ **NUNCA hacer commit sin actualizar CHANGELOG**
```bash
# ❌ PROHIBIDO - Commit sin changelog
git commit -m "feat: nuevo tipo de notificación"

# ✅ CORRECTO - Actualizar changelog primero
# 1. Editar CHANGELOG.md
# 2. Luego hacer commit
git commit -m "feat: nuevo tipo de notificación"
```

## 📅 **REGLA OBLIGATORIA: Formato de Fechas**

### ✅ **SIEMPRE usar formato DD-MM-YYYY**
```markdown
## [1.0.0] - 27-01-2025

# ✅ CORRECTO
- 27-01-2025 (día-mes-año)
- 15-12-2024
- 01-03-2025

# ❌ PROHIBIDO
- 2025-01-27 (año-mes-día)
- 01/27/2025 (mes/día/año)
- 27/01/2025 (con barras)
```

## 📋 **Formato Obligatorio**

### **Estructura del CHANGELOG.md**
```markdown
# Changelog - AccessAI AI Processing Service

## [Unreleased]

### Added
- Nuevo tipo de notificación: community_post_tag
- Endpoint GET /notifications/:userId para historial

### Changed  
- Optimización de conexiones WebSocket
- Mejora en manejo de notificaciones masivas

### Fixed
- Corrección de desconexión automática de sockets
- Fix en formato de payload de notificaciones

## [1.0.1] - 2024-01-20
### Added
- Sistema de notificaciones en tiempo real
- WebSocket para chat en vivo
```

## 🔄 **Workflow Obligatorio**

### **Proceso de Desarrollo**
1. **Implementar funcionalidad**
2. **Actualizar CHANGELOG.md** en sección `[Unreleased]`
3. **Hacer commit** con cambios + changelog
4. **Incrementar versión** cuando sea necesario

## 📝 **Ejemplos por Tipo de Cambio**

### **Nuevas Notificaciones**
```markdown
### Added
- Notificación community_member_join para nuevos miembros
- Notificación post_share para compartidos de posts
- Endpoint POST /notifications/bulk para notificaciones masivas
```

### **Mejoras de Chat**
```markdown
### Changed
- Optimización de rooms de Socket.IO por comunidad
- Mejora en persistencia de mensajes de chat
- Refactorización de chatService para mejor performance
```

### **Correcciones WebSocket**
```markdown
### Fixed
- Corrección de memory leaks en conexiones Socket.IO
- Fix en autenticación de usuarios en WebSocket
- Resolución de problema con rooms duplicados
```

### **Integración Backend**
```markdown
### Changed
- Actualización de URL de integración con backend principal
- Mejora en manejo de errores de API calls al backend
- Optimización de payload de notificaciones enviadas
```

## 🎯 **Checklist Pre-Commit**

- [ ] ✅ CHANGELOG.md actualizado en sección `[Unreleased]`
- [ ] ✅ Cambios categorizados correctamente
- [ ] ✅ Nuevos tipos de notificación documentados
- [ ] ✅ Cambios en WebSocket/Socket.IO mencionados
- [ ] ✅ Integraciones con backend documentadas

## 🚨 **Recordatorio Automático**

### **Mensaje para el Desarrollador**
```
⚠️  RECORDATORIO OBLIGATORIO - MICROSERVICIO:
📝 ¿Actualizaste CHANGELOG.md con los cambios implementados?
🔄 Sección [Unreleased] debe incluir:
   - Added: Nuevas notificaciones/funcionalidades
   - Changed: Mejoras en chat/WebSocket
   - Fixed: Correcciones de conexiones/notificaciones
   
✅ Solo después de actualizar CHANGELOG, procede con el commit
```

Esta regla garantiza documentación completa de todos los cambios en el servicio IA de AccessAI.