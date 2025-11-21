# AccessAI - Reglas de Idioma

## 🌍 **REGLA OBLIGATORIA: Español en Todo**

### ✅ SIEMPRE usar español para:
- **Respuestas del asistente**: Todas las explicaciones en español
- **Comentarios de código**: Documentación en español
- **Documentación**: Archivos .md en español
- **Logs y console**: Mensajes informativos en español

### 🔧 **Mensajes de Error - Regla Específica:**
- **Mensajes técnicos/API**: En inglés (para consistencia con estándares)
- **Mensajes de usuario final**: En español (cuando sea para UI)
- **Validaciones de formulario**: En español
- **Errores de sistema**: En inglés

### 🔧 Código en inglés (estándar):
- **Nombres de variables**: en inglés
- **Nombres de funciones**: en inglés  
- **Nombres de clases**: PascalCase en inglés
- **Nombres de archivos**: kebab-case en inglés
- **APIs y endpoints**: URLs en inglés

### Ejemplos Correctos:

```javascript
// ✅ CORRECTO - Comentarios en español, código en inglés
/**
 * Crear nuevo usuario en el sistema
 * @param {Object} userData - Datos del usuario
 * @returns {Promise<Object>} Usuario creado
 */
const createUser = async (userData) => {
    // Validar que el email no exista
    const existingUser = await User.findOne({ where: { email: userData.email } });
    
    if (existingUser) {
        // ✅ Error técnico/API en inglés
        throw new Error('Email already exists');
    }
    
    // Crear usuario con datos validados
    const newUser = await User.create(userData);
    console.log('Usuario creado exitosamente:', newUser.id);
    
    return newUser;
};

// ✅ Validaciones de formulario en español (para UI)
const validatePassword = (password) => {
    if (!password) {
        return 'La contraseña es requerida';
    }
    if (password.length < 8) {
        return 'La contraseña debe tener al menos 8 caracteres';
    }
    return null;
};

// ✅ Errores de sistema en inglés
if (!database.isConnected()) {
    throw new Error('Database connection failed');
}
```

### ❌ Ejemplos Incorrectos:

```javascript
// ❌ MAL - Comentarios en inglés
/**
 * Create new user in system
 * @param {Object} userData - User data
 */
const createUser = async (userData) => {
    // Check if email exists - MAL: comentario en inglés
    if (existingUser) {
        throw new Error('Email already exists'); // OK: error técnico
    }
};

// ❌ MAL - Variables en español
const crearUsuario = async (datosUsuario) => {
    const usuarioExistente = await Usuario.findOne();
};

// ❌ MAL - Mezclar idiomas inconsistentemente
if (user.emailVerified === false) {
    throw new Error('El email no está verificado'); // Debería ser inglés para API
}
```

## 📝 Documentación

### Estructura de archivos:
- **README.md**: En español
- **Documentación técnica**: En español
- **Comentarios JSDoc**: En español
- **Mensajes de commit**: En español

### Respuestas del asistente:
- **Explicaciones**: Siempre en español
- **Análisis de código**: En español
- **Sugerencias**: En español
- **Resolución de errores**: En español

Esta regla garantiza consistencia en la comunicación y documentación del proyecto AccessAI.

## 🔧 **Estandarización de Mensajes de Error**

### **Regla de Migración Gradual:**
- **Nuevos errores**: SIEMPRE en inglés para APIs
- **Errores existentes**: Migrar gradualmente a inglés
- **Validaciones de UI**: Mantener en español

### **Patrones Estándar:**

#### **✅ Errores Técnicos (Inglés):**
```javascript
// Recursos no encontrados
throw new Error('User not found');
throw new Error('Post not found');
throw new Error('Community not found');

// Validaciones de negocio
throw new Error('Cannot request to join your own community');
throw new Error('Cannot send contact request to yourself');

// Estados inválidos
throw new Error('Request already processed');
throw new Error('Email already exists');

// Permisos
throw new Error('Only community creator can perform this action');
throw new Error('Not authorized to update this request');
```

#### **✅ Validaciones de UI (Español):**
```javascript
// Formularios de usuario
errors.push('El email es requerido');
errors.push('La contraseña debe tener al menos 8 caracteres');
errors.push('El nombre debe tener entre 1 y 100 caracteres');
```

### **Proceso de Estandarización:**
1. **Identificar** mensajes mixtos en errorHandler.js
2. **Decidir** si es error técnico (inglés) o UI (español)
3. **Actualizar** services para usar inglés consistentemente
4. **Mantener** validaciones de formulario en español
5. **Documentar** cambios en commits

Esta regla garantiza consistencia profesional en APIs de AccessAI mientras mantiene UX en español.