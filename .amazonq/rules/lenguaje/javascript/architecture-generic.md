# Backend - Architecture Rules

## 🌍 **REGLA OBLIGATORIA: Respuestas en Español**

### ✅ SIEMPRE responder en español
- **Comentarios de código**: En español
- **Mensajes de error**: En español  
- **Documentación**: En español
- **Explicaciones**: En español
- **Variables y funciones**: En inglés (estándar de programación)

```javascript
// ✅ CORRECTO - Comentarios en español
// Validar datos del usuario
const validarUsuario = async (userData) => {
    // Verificar si el usuario ya existe
    const usuarioExistente = await User.findOne({ where: { email } });
    if (usuarioExistente) {
        throw new Error('User already exists');
    }
};

// ❌ PROHIBIDO - Comentarios en inglés
// Validate user data
const validateUser = async (userData) => {
    // Check if email already exists
    throw new Error('Email already exists');
};
```

## 🏗️ ES Modules Obligatorio

### Import/Export Patterns
```javascript
// ✅ CORRECTO - Siempre con extensión .js
import authService from '../services/authService.js';
import { asyncHandler } from '../utils/errorHandler.js';
import UserModel from '../models/User.js';
import ResourceModel from '../models/Resource.js';

// ✅ Export default para clases singleton
export default new AuthController();

// ✅ Named exports para utilidades
export { validateEmail, isValidPassword };

// ❌ PROHIBIDO - CommonJS
const authService = require('../services/authService');
module.exports = AuthController;
```

### File Extensions
- **Obligatorio**: `.js` en todos los imports
- **Migraciones**: `.cjs` para Sequelize CLI
- **Configuración**: `.js` para código, `.cjs` para CLI

## 📅 **REGLA OBLIGATORIA: Timestamps en snake_case**

### ✅ SIEMPRE usar snake_case para timestamps
```javascript
// ✅ CORRECTO - SIEMPRE usar snake_case
order: [['created_at', 'DESC']]
order: [['updated_at', 'DESC']]

// ❌ PROHIBIDO - NUNCA usar camelCase
order: [['createdAt', 'DESC']]  // ❌ MAL
order: [['updatedAt', 'DESC']]  // ❌ MAL
```

### Configuración de Modelos
```javascript
// ✅ CORRECTO - Configuración obligatoria
{
    tableName: 'table_name',
    underscored: true,
    timestamps: true,
    createdAt: 'created_at',
    updatedAt: 'updated_at'
}
```

## 🎯 Controller-Service Pattern

### Controller Layer (HTTP Only)
```javascript
// controllers/authController.js
import authService from '../services/authService.js';
import { asyncHandler } from '../utils/errorHandler.js';

class AuthController {
    /**
     * Register user - SOLO manejo HTTP
     * @param {import('express').Request} req 
     * @param {import('express').Response} res 
     */
    register = asyncHandler(async (req, res) => {
        const user = await authService.register(req.body);
        res.status(201).json(user);
    });

    login = asyncHandler(async (req, res) => {
        const userData = await authService.login(req.body, res);
        res.status(200).json(userData);
    });
}

export default new AuthController();
```

### Service Layer (Business Logic)
```javascript
// services/authService.js
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import { sequelize } from '../../database/connection.js';

class AuthService {
    /**
     * Register user - TODA la lógica de negocio
     */
    async register(userData) {
        const { email, password, roleKey } = userData;
        
        // Validaciones de negocio
        const existingUser = await User.findOne({ where: { email } });
        if (existingUser) {
            throw new Error('Email already exists');
        }

        // Operaciones de BD
        const user = await User.create({
            email: email.toLowerCase().trim(),
            passwordHash: password, // Hook lo hashea
            roleId: role.id
        });

        // Lógica adicional (emails, tokens, etc.)
        await this.sendVerificationEmail(user);
        
        return this.formatUserResponse(user);
    }
}

export default new AuthService();
```

## 🔄 AsyncHandler Pattern

### Wrapper Obligatorio
```javascript
// utils/errorHandler.js
export const asyncHandler = (fn) => (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
};

// ✅ SIEMPRE usar en controllers
methodName = asyncHandler(async (req, res) => {
    // Lógica sin try/catch
    const result = await service.method(req.body);
    res.status(200).json(result);
});

// ❌ NUNCA manejar errores manualmente en controllers
async methodName(req, res) {
    try {
        // NO hacer esto
    } catch (error) {
        // NO manejar aquí
    }
}
```

### Error Handler Global
```javascript
// middlewares/errorHandler.js
export const globalErrorHandler = (err, req, res, next) => {
    console.error('Error:', err);

    // Errores de validación
    if (err.name === 'ValidationError') {
        return res.status(422).json({
            error: 'Validation Error',
            message: err.message,
            details: err.details || []
        });
    }

    // Errores de autenticación
    if (err.message === 'Unauthorized') {
        return res.status(401).json({
            error: 'Unauthorized',
            message: 'Authentication required'
        });
    }

    // Error genérico
    res.status(500).json({
        error: 'Internal Server Error',
        message: 'An unexpected error occurred'
    });
};
```

## 📁 Directory Structure

### Estructura Obligatoria
```
src/
├── config/
│   ├── database.js       # Configuración PostgreSQL
│   └── logger.js         # Configuración Winston
├── controllers/          # SOLO manejo HTTP
│   ├── authController.js
│   ├── userController.js
│   └── resourceController.js
├── services/            # Lógica de negocio
│   ├── authService.js
│   ├── userService.js
│   └── resourceService.js
├── middleware/          # Middleware personalizado
│   ├── auth.js
│   ├── rateLimiter.js
│   └── validation.js
├── routes/              # Definición de rutas
│   ├── auth.js
│   ├── users.js
│   ├── resources.js
│   └── index.js
├── models/              # Modelos Sequelize
│   ├── User.js
│   ├── Resource.js
│   └── index.js
└── seeders/             # Datos de prueba
    ├── demoSeeder.js
    └── index.js
```

## 🔧 Model Initialization

### Pattern de Inicialización
```javascript
// services/authService.js
import { User, Resource } from '../models/index.js';

// Asociaciones ya definidas en models/index.js
// User.hasMany(Resource, { foreignKey: 'userId', as: 'resources' });
```

## 📋 JSDoc Obligatorio

### Documentation Pattern
```javascript
/**
 * Register a new user
 * @param {import('express').Request} req - Express request object
 * @param {import('express').Response} res - Express response object
 * @returns {Promise<void>}
 */
async register(req, res) {
    // Implementation
}

/**
 * Validate user registration data
 * @param {Object} userData - User registration data
 * @param {string} userData.email - User email
 * @param {string} userData.password - User password
 * @returns {Promise<Object>} Created user data
 * @throws {Error} When validation fails
 */
async register(userData) {
    // Implementation
}
```

## 🚀 Performance Patterns

### Lazy Loading
```javascript
// ✅ Import solo cuando se necesita
class AuthService {
    async sendEmail(user) {
        // Solo importar cuando se usa
        const emailService = await import('./emailService.js');
        return emailService.default.send(user.email);
    }
}
```

### Connection Sharing
```javascript
// ✅ Una sola conexión compartida
import { sequelize } from '../../database/connection.js';

// ❌ NO crear múltiples conexiones
const sequelize = new Sequelize(/* config */);
```

## 🔄 **Database Sync Configuration**

### **Force Sync Pattern**
```javascript
// app/index.js - Configuración obligatoria
if (!process.env.SKIP_SERVER_START) {
    const isDev = process.env.NODE_ENV === 'development';

    await databaseManager.sequelize.sync({
        force: isDev, // true en desarrollo, false en producción
        logging: false
    });
}
```

### **Implicaciones por Entorno**
```javascript
// ✅ DESARROLLO
// - force: true
// - Tablas se recrean en cada inicio
// - Cambios de modelo automáticos
// - Seeders obligatorios para datos

// ✅ PRODUCCIÓN  
// - force: false
// - Datos preservados
// - Migraciones requeridas para cambios
// - Seguridad de datos garantizada
```

### **Desarrollo Workflow**
1. **Modificar modelo** → Reiniciar servidor
2. **Estructura actualizada** → Ejecutar seeders
3. **Datos de prueba** → Continuar desarrollo

Esta arquitectura garantiza desarrollo ágil y producción segura del backend.