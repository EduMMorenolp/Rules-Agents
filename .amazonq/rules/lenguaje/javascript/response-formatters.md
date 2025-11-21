# AccessAI Backend - Response Formatters Rules

## 🎯 **REGLA OBLIGATORIA: Formatters Centralizados**

### ✅ **SIEMPRE usar formatters para respuestas**
- **NUNCA** construir respuestas manualmente en services
- **SIEMPRE** usar formatters de `utils/responseFormatter.js`
- **OBLIGATORIO** crear formatter específico para cada entidad
- **CONSISTENCIA** en estructura de respuestas

```javascript
// ✅ CORRECTO - Usar formatter
import { formatContactUserResponse } from '../utils/responseFormatter.js';

const users = await User.findAll();
return users.map(formatContactUserResponse);

// ❌ PROHIBIDO - Construcción manual
return users.map(user => ({
    id: user.id,
    firstName: user.firstName,
    // ... construcción repetitiva
}));
```

## 📋 **Formatters Disponibles**

### **Usuarios y Autenticación**
```javascript
// Para usuarios completos (perfil, auth)
formatUserResponse(user, role)

// Para listados de usuarios
formatUserListResponse(user)

// Para registro de usuarios
formatRegisterResponse(user, role)
```

### **Sistema de Alertas y Monitoreo**
```javascript
// Para alertas
formatAlertResponse(alert, device, user = null)

// Para dispositivos
formatDeviceResponse(device, zone, alerts = [])

// Para logs de acceso
formatAccessLogResponse(accessLog, user, device)

// Para visitantes
formatVisitorResponse(visitor, zones = [])

// Para métricas del sistema
formatMetricsResponse(metrics, timestamp)
```

### **Zonas y Dispositivos**
```javascript
// Para zonas
formatZoneResponse(zone, devices = [])

// Para embeddings faciales
formatFaceEmbeddingResponse(embedding, user)
```

## 🔧 **Patrón de Formatters**

### **Estructura Estándar**
```javascript
/**
 * Formatear respuesta de [entidad]
 */
export const format[Entity]Response = (entity, ...additionalData) => {
    // Manejar objetos Sequelize
    if (entity.get) {
        const plainEntity = entity.get({ plain: true });
        return {
            // Campos formateados
        };
    }
    
    // Manejar objetos planos (SQL raw)
    return {
        id: entity.id,
        field1: entity.field1 || entity.field_1, // snake_case fallback
        field2: entity.field2 || entity.field_2,
        // ... otros campos
    };
};
```

### **Campos Obligatorios por Entidad**

#### **formatUserListResponse**
```javascript
{
    id: string,
    firstName: string,
    lastName: string,
    email: string,
    role: string,
    isActive: boolean,
    lastLogin: date,
    createdAt: date
}
```

#### **formatAlertResponse**
```javascript
{
    id: string,
    type: string,
    message: string,
    severity: string,
    resolved: boolean,
    deviceId: string,
    userId: string | null,
    created_at: date,
    updated_at: date,
    device: {
        id: string,
        name: string,
        location: string,
        zone: string
    },
    user: {
        id: string,
        firstName: string,
        lastName: string
    } | null
}
```

## 🚀 **Uso en Services**

### **Importación Obligatoria**
```javascript
// services/contactService.js
import { formatContactUserResponse } from '../utils/responseFormatter.js';

// services/postService.js
import { formatPostResponse, formatCommentResponse } from '../utils/responseFormatter.js';
```

### **Aplicación en Queries**
```javascript
// ✅ CORRECTO - Sequelize queries
const { count, rows } = await User.findAndCountAll({
    // ... query config
});

return {
    data: rows.map(formatContactUserResponse), // ✅ Usar formatter
    pagination: { /* ... */ }
};

// ✅ CORRECTO - SQL raw queries
const rows = await sequelize.query(query, { /* ... */ });
return rows.map(formatContactUserResponse); // ✅ Usar formatter
```

## 📊 **Manejo de Datos Mixtos**

### **Objetos Sequelize vs SQL Raw**
```javascript
export const formatContactUserResponse = (user) => {
    // Para objetos Sequelize
    if (user.get) {
        const plainUser = user.get({ plain: true });
        return {
            id: plainUser.id,
            firstName: plainUser.firstName,
            // ... campos camelCase
        };
    }
    
    // Para objetos planos (SQL raw)
    return {
        id: user.id,
        firstName: user.firstName || user.first_name, // Fallback snake_case
        lastName: user.lastName || user.last_name,
        // ... otros campos con fallback
    };
};
```

## 🔄 **Proceso de Creación de Formatters**

### **Cuando Crear Nuevo Formatter**
1. **Nueva entidad** en el sistema
2. **Estructura de respuesta específica** requerida
3. **Campos diferentes** a formatters existentes
4. **Contexto específico** (ej: contactos vs perfil completo)

### **Pasos Obligatorios**
1. **Crear función** en `utils/responseFormatter.js`
2. **Documentar campos** obligatorios en JSDoc
3. **Manejar ambos tipos** de objetos (Sequelize y planos)
4. **Importar en services** que lo necesiten
5. **Actualizar esta documentación**

## 📝 **Ejemplos de Uso**

### **Contactos - Múltiples Formatters**
```javascript
// services/contactService.js
async getContactSuggestions(userId, page, limit) {
    // Query SQL raw
    const rows = await sequelize.query(query);
    
    // ✅ Usar formatter para SQL raw
    return rows.map(formatContactUserResponse);
}

async getAvailableUsers(userId, page, limit) {
    // Query Sequelize
    const { rows } = await User.findAndCountAll();
    
    // ✅ Usar formatter para Sequelize
    return rows.map(formatContactUserResponse);
}
```

### **Posts - Formatter Complejo**
```javascript
// services/postService.js
async createPost(userId, data) {
    const [post, user, reactions] = await Promise.all([
        Post.create(data),
        User.findByPk(userId),
        Reaction.findAll({ where: { postId } })
    ]);
    
    // ✅ Usar formatter con datos adicionales
    return formatPostResponse(post, user, reactions);
}
```

## 🚨 **Validaciones Obligatorias**

### **Pre-Commit Checklist**
- [ ] ✅ Todos los services usan formatters
- [ ] ✅ No hay construcción manual de respuestas
- [ ] ✅ Formatters manejan ambos tipos de objetos
- [ ] ✅ Campos obligatorios incluidos
- [ ] ✅ Imports correctos en services

### **Comando de Verificación**
```bash
# Buscar construcción manual de respuestas
grep -r "return.*{.*id:" src/app/services/
grep -r "\.map.*=>" src/app/services/ | grep -v "formatters"
```

## 📚 **Beneficios de Formatters**

### **Consistencia**
- ✅ Estructura uniforme en todas las respuestas
- ✅ Campos estandarizados por entidad
- ✅ Manejo consistente de datos nulos

### **Mantenibilidad**
- ✅ Cambios centralizados en un solo lugar
- ✅ Reutilización entre múltiples endpoints
- ✅ Fácil testing de estructura de respuestas

### **Performance**
- ✅ Lógica de formateo optimizada
- ✅ Manejo eficiente de objetos Sequelize
- ✅ Reducción de código duplicado

Esta configuración garantiza respuestas consistentes, mantenibles y optimizadas en todo el backend AccessAI.