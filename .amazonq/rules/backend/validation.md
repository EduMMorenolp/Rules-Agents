# AccessAI Backend - Validation Rules

## 🔍 Manual Validation Pattern

### Base Validator Utilities
```javascript
// validators/baseValidator.js

/**
 * Email validation
 */
export const isValidEmail = (email) => {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
};

/**
 * Password validation - 8+ chars, uppercase, number, special char
 */
export const isValidPassword = (password) => {
    const passwordRegex = /^(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*(),.?\":{}|<>]).{8,}$/;
    return passwordRegex.test(password);
};

/**
 * Length validation
 */
export const isValidLength = (value, min = 1, max = null) => {
    if (!value || typeof value !== 'string') return false;
    if (value.length < min) return false;
    if (max && value.length > max) return false;
    return true;
};

/**
 * UUID validation
 */
export const isValidUUID = (id) => {
    const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
    return uuidRegex.test(id);
};

/**
 * URL validation
 */
export const isValidUrl = (url) => {
    try {
        new URL(url);
        return true;
    } catch {
        return false;
    }
};

/**
 * Role validation - Roles específicos AccessAI
 */
export const isValidRole = (roleKey) => {
    const validRoles = ['admin', 'security_officer', 'employee', 'visitor'];
    return validRoles.includes(roleKey);
};

/**
 * Phone validation (basic)
 */
export const isValidPhone = (phone) => {
    const phoneRegex = /^[\+]?[1-9][\d]{0,15}$/;
    return phoneRegex.test(phone);
};

/**
 * Create validation error response
 */
export const createValidationError = (errors) => {
    return {
        error: 'Validation Error',
        message: 'Invalid input data',
        details: Array.isArray(errors) ? errors : [errors]
    };
};
```

## 📝 Auth Validators

### Registration Validation - Basado en Postman Collection
```javascript
// validators/authValidators.js
import {
    isValidEmail,
    isValidPassword,
    isValidLength,
    isValidRole,
    isValidUrl,
    isValidPhone,
    createValidationError
} from './baseValidator.js';

/**
 * Validate user registration - AccessAI system
 */
export const validateUserRegistration = (req, res, next) => {
    const {
        email, password, firstName, lastName, roleKey = 'employee',
        phone, department, position, badgeNumber,
        // Campos específicos para visitantes
        company, visitPurpose, validFrom, validTo,
        // Zonas autorizadas
        authorizedZones
    } = req.body;
    
    const errors = [];

    // Common required fields
    if (!email) {
        errors.push('Email is required');
    } else {
        // Normalize email
        req.body.email = email.toLowerCase().trim();
        
        if (!isValidEmail(req.body.email)) {
            errors.push('Invalid email format');
        }
    }

    if (!password) {
        errors.push('Password is required');
    } else if (!isValidPassword(password)) {
        errors.push('Password must be at least 8 characters with uppercase, number and special character');
    }

    // Role validation
    if (!isValidRole(roleKey)) {
        errors.push('Invalid role key');
    }

    // Phone validation (optional)
    if (phone && !isValidPhone(phone)) {
        errors.push('Invalid phone format');
    }

    // Role-specific validations - Según estructura AccessAI
    if (roleKey === 'employee' || roleKey === 'security_officer') {
        // Required fields for employees
        if (!firstName) errors.push('First name is required');
        if (!lastName) errors.push('Last name is required');
        if (!department) errors.push('Department is required');

        // Length validations
        if (firstName && !isValidLength(firstName, 1, 100)) {
            errors.push('First name must be between 1 and 100 characters');
        }
        if (lastName && !isValidLength(lastName, 1, 100)) {
            errors.push('Last name must be between 1 and 100 characters');
        }
        
        // Optional fields validation para personas
        if (province && !isValidLength(province, 1, 100)) {
            errors.push('Province must be between 1 and 100 characters');
        }
        if (city && !isValidLength(city, 1, 100)) {
            errors.push('City must be between 1 and 100 characters');
        }
        if (industry && !isValidLength(industry, 1, 100)) {
            errors.push('Industry must be between 1 and 100 characters');
        }
        if (company && !isValidLength(company, 1, 255)) {
            errors.push('Company must be between 1 and 255 characters');
        }
        
    } else if (roleKey === 'visitor') {
        // Required fields for visitors
        if (!firstName) errors.push('First name is required');
        if (!lastName) errors.push('Last name is required');
        if (!company) errors.push('Company is required');
        if (!visitPurpose) errors.push('Visit purpose is required');
        if (!validFrom) errors.push('Valid from date is required');
        if (!validTo) errors.push('Valid to date is required');

        // Length validations for business
        if (businessName && !isValidLength(businessName, 1, 255)) {
            errors.push('Business name must be between 1 and 255 characters');
        }
        if (representative && !isValidLength(representative, 1, 255)) {
            errors.push('Representative must be between 1 and 255 characters');
        }
        if (province && !isValidLength(province, 1, 100)) {
            errors.push('Province must be between 1 and 100 characters');
        }
        if (locality && !isValidLength(locality, 1, 100)) {
            errors.push('Locality must be between 1 and 100 characters');
        }
        if (address && !isValidLength(address, 1, 500)) {
            errors.push('Address must be between 1 and 500 characters');
        }
        if (industry && !isValidLength(industry, 1, 100)) {
            errors.push('Industry must be between 1 and 100 characters');
        }
        
        // Validación de archivos opcionales (base64)
        if (logoFile && typeof logoFile !== 'string') {
            errors.push('Logo file must be base64 string');
        }
        if (statuteFile && typeof statuteFile !== 'string') {
            errors.push('Statute file must be base64 string');
        }
    }

    // Optional field validations (all roles)
    if (employeesCount !== undefined) {
        const count = parseInt(employeesCount);
        if (isNaN(count) || count < 0 || count > 999999) {
            errors.push('Employees count must be between 0 and 999999');
        }
    }

    if (websiteUrl && !isValidUrl(websiteUrl)) {
        errors.push('Invalid website URL format');
    }

    if (linkedinUrl && !isValidUrl(linkedinUrl)) {
        errors.push('Invalid LinkedIn URL format');
    }

    if (bio && !isValidLength(bio, 1, 500)) {
        errors.push('Bio must be between 1 and 500 characters');
    }

    // Industrial sectors validation (if provided)
    if (industrialSectors) {
        if (!Array.isArray(industrialSectors)) {
            errors.push('Industrial sectors must be an array');
        } else if (industrialSectors.length > 10) {
            errors.push('Maximum 10 industrial sectors allowed');
        }
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};

/**
 * Validate user login data
 */
export const validateUserLogin = (req, res, next) => {
    const { email, password } = req.body;
    const errors = [];

    if (!email) {
        errors.push('Email is required');
    } else {
        // Normalize email
        req.body.email = email.toLowerCase().trim();
        
        if (!isValidEmail(req.body.email)) {
            errors.push('Invalid email format');
        }
    }

    if (!password) {
        errors.push('Password is required');
    } else if (!isValidLength(password, 1)) {
        errors.push('Password cannot be empty');
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};

/**
 * Validate Google authentication - Endpoint /auth/google
 */
export const validateGoogleAuth = (req, res, next) => {
    const { idToken } = req.body;
    const errors = [];

    if (!idToken) {
        errors.push('Google ID token is required');
    } else if (typeof idToken !== 'string') {
        errors.push('ID token must be a string');
    } else if (idToken.length < 100) {
        errors.push('Invalid Google ID token format');
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};

/**
 * Validate email verification - Endpoint /auth/verify-email
 */
export const validateEmailVerification = (req, res, next) => {
    const { token } = req.body;
    const errors = [];

    if (!token) {
        errors.push('Verification token is required');
    } else if (typeof token !== 'string') {
        errors.push('Token must be a string');
    } else if (token.length < 32) {
        errors.push('Invalid token format');
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};

/**
 * Validate password reset - Endpoint /auth/password/reset
 */
export const validatePasswordReset = (req, res, next) => {
    const { token, newPassword } = req.body;
    const errors = [];

    if (!token) {
        errors.push('Reset token is required');
    } else if (typeof token !== 'string') {
        errors.push('Token must be a string');
    }

    if (!newPassword) {
        errors.push('New password is required');
    } else if (!isValidPassword(newPassword)) {
        errors.push('New password must be at least 8 characters with uppercase, number and special character');
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};
```

## 📄 Alert and Device Validation

### Alert and Device Validation
```javascript
// validators/alertValidators.js
import { isValidLength, createValidationError } from './baseValidator.js';

/**
 * Alert limits - Según documentación AccessAI
 */
const ALERT_LIMITS = {
    MESSAGE_MIN: 1,
    MESSAGE_MAX: 500,
    DESCRIPTION_MIN: 1,
    DESCRIPTION_MAX: 1000
};

/**
 * Valid alert types
 */
const VALID_ALERT_TYPES = ['unauthorized_access', 'unknown_person', 'after_hours', 'zone_violation'];

/**
 * Valid alert severities
 */
const VALID_SEVERITIES = ['low', 'medium', 'high', 'critical'];

/**
 * Validate alert creation - Endpoint POST /alerts
 */
export const validateAlertCreation = (req, res, next) => {
    const { type, message, severity = 'medium', deviceId } = req.body;
    const errors = [];

    // Type validation
    if (!type) {
        errors.push('Alert type is required');
    } else if (!VALID_ALERT_TYPES.includes(type)) {
        errors.push(`Invalid alert type. Must be one of: ${VALID_ALERT_TYPES.join(', ')}`);
    }

    // Message validation
    if (!message) {
        errors.push('Alert message is required');
    } else if (!isValidLength(message, ALERT_LIMITS.MESSAGE_MIN, ALERT_LIMITS.MESSAGE_MAX)) {
        errors.push(`Message must be between ${ALERT_LIMITS.MESSAGE_MIN} and ${ALERT_LIMITS.MESSAGE_MAX} characters`);
    }

    // Severity validation
    if (!VALID_SEVERITIES.includes(severity)) {
        errors.push(`Invalid severity. Must be one of: ${VALID_SEVERITIES.join(', ')}`);
    }

    // Device ID validation
    if (!deviceId) {
        errors.push('Device ID is required');
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};

/**
 * Validate device creation - Endpoint POST /devices
 */
export const validateDeviceCreation = (req, res, next) => {
    const { name, location, zoneId, type = 'camera' } = req.body;
    const errors = [];

    if (!name) {
        errors.push('Device name is required');
    } else if (!isValidLength(name, 1, 100)) {
        errors.push('Device name must be between 1 and 100 characters');
    }

    if (!location) {
        errors.push('Device location is required');
    } else if (!isValidLength(location, 1, 255)) {
        errors.push('Location must be between 1 and 255 characters');
    }

    if (!zoneId) {
        errors.push('Zone ID is required');
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};

/**
 * Validate visitor creation - Endpoint POST /visitors
 */
export const validateVisitorCreation = (req, res, next) => {
    const { firstName, lastName, company, visitPurpose, validFrom, validTo } = req.body;
    const errors = [];

    if (!firstName) {
        errors.push('First name is required');
    }

    if (!lastName) {
        errors.push('Last name is required');
    }

    if (!company) {
        errors.push('Company is required');
    }

    if (!visitPurpose) {
        errors.push('Visit purpose is required');
    }

    if (!validFrom) {
        errors.push('Valid from date is required');
    }

    if (!validTo) {
        errors.push('Valid to date is required');
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};
```

## 👤 Profile Validators - Gestión de Perfiles

### Profile Update Validation
```javascript
// validators/profileValidators.js
import {
    isValidLength,
    isValidUrl,
    isValidPhone,
    isValidEmail,
    isValidPassword,
    createValidationError
} from './baseValidator.js';

/**
 * Validate profile update - Endpoint PUT /profile/me
 */
export const validateProfileUpdate = (req, res, next) => {
    const {
        firstName, lastName, phone, province, city, industry, 
        company, bio, websiteUrl, linkedinUrl, representative,
        locality, address, employeesCount, industrialSectors
    } = req.body;
    
    const errors = [];

    // Basic field validations
    if (firstName && !isValidLength(firstName, 1, 100)) {
        errors.push('First name must be between 1 and 100 characters');
    }

    if (lastName && !isValidLength(lastName, 1, 100)) {
        errors.push('Last name must be between 1 and 100 characters');
    }

    if (phone && !isValidPhone(phone)) {
        errors.push('Invalid phone format');
    }

    if (province && !isValidLength(province, 1, 100)) {
        errors.push('Province must be between 1 and 100 characters');
    }

    if (city && !isValidLength(city, 1, 100)) {
        errors.push('City must be between 1 and 100 characters');
    }

    if (industry && !isValidLength(industry, 1, 100)) {
        errors.push('Industry must be between 1 and 100 characters');
    }

    if (company && !isValidLength(company, 1, 255)) {
        errors.push('Company must be between 1 and 255 characters');
    }

    if (bio && !isValidLength(bio, 1, 500)) {
        errors.push('Bio must be between 1 and 500 characters');
    }

    // URL validations
    if (websiteUrl && !isValidUrl(websiteUrl)) {
        errors.push('Invalid website URL format');
    }

    if (linkedinUrl && !isValidUrl(linkedinUrl)) {
        errors.push('Invalid LinkedIn URL format');
    }

    // Business profile fields
    if (representative && !isValidLength(representative, 1, 255)) {
        errors.push('Representative must be between 1 and 255 characters');
    }

    if (locality && !isValidLength(locality, 1, 100)) {
        errors.push('Locality must be between 1 and 100 characters');
    }

    if (address && !isValidLength(address, 1, 500)) {
        errors.push('Address must be between 1 and 500 characters');
    }

    if (employeesCount !== undefined) {
        const count = parseInt(employeesCount);
        if (isNaN(count) || count < 0 || count > 999999) {
            errors.push('Employees count must be between 0 and 999999');
        }
    }

    // Industrial sectors validation
    if (industrialSectors) {
        if (!Array.isArray(industrialSectors)) {
            errors.push('Industrial sectors must be an array');
        } else if (industrialSectors.length > 10) {
            errors.push('Maximum 10 industrial sectors allowed');
        }
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};

/**
 * Validate password change - Endpoint PATCH /profile/me/password
 */
export const validatePasswordChange = (req, res, next) => {
    const { currentPassword, newPassword } = req.body;
    const errors = [];

    if (!currentPassword) {
        errors.push('Current password is required');
    }

    if (!newPassword) {
        errors.push('New password is required');
    } else if (!isValidPassword(newPassword)) {
        errors.push('New password must be at least 8 characters with uppercase, number and special character');
    }

    if (currentPassword && newPassword && currentPassword === newPassword) {
        errors.push('New password must be different from current password');
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};

/**
 * Validate email change - Endpoint PATCH /profile/me/email
 */
export const validateEmailChange = (req, res, next) => {
    const { email } = req.body; // Según Postman es 'email', no 'newEmail'
    const errors = [];

    if (!email) {
        errors.push('New email is required');
    } else {
        // Normalize email
        req.body.email = email.toLowerCase().trim();
        
        if (!isValidEmail(req.body.email)) {
            errors.push('Invalid email format');
        }
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};

/**
 * Validate file upload - Endpoints PATCH /profile/me/avatar|banner
 */
export const validateFileUpload = (fileField) => {
    return (req, res, next) => {
        const fileData = req.body[fileField];
        const errors = [];

        if (!fileData) {
            errors.push(`${fileField} is required`);
        } else if (typeof fileData !== 'string') {
            errors.push(`${fileField} must be base64 string`);
        } else if (!fileData.startsWith('data:image/')) {
            errors.push(`${fileField} must be a valid image data URL`);
        }

        if (errors.length > 0) {
            return res.status(422).json(createValidationError(errors));
        }

        next();
    };
};
```

## 🔧 Parameter Validators

### UUID and Pagination Validation
```javascript
// validators/paramValidators.js
import { isValidUUID, createValidationError } from './baseValidator.js';

/**
 * Validate UUID parameter
 */
export const validateUUIDParam = (paramName = 'id') => {
    return (req, res, next) => {
        const id = req.params[paramName];
        
        if (!id) {
            return res.status(400).json({
                error: 'Bad Request',
                message: `${paramName} parameter is required`
            });
        }

        if (!isValidUUID(id)) {
            return res.status(400).json({
                error: 'Bad Request',
                message: `Invalid ${paramName} format`
            });
        }

        next();
    };
};

/**
 * Validate pagination parameters - Usado en múltiples endpoints
 */
export const validatePagination = (req, res, next) => {
    let { page = 1, limit = 10 } = req.query;
    const errors = [];

    // Convert to numbers
    page = parseInt(page);
    limit = parseInt(limit);

    // Validate page
    if (isNaN(page) || page < 1) {
        errors.push('Page must be a positive integer');
    }

    // Validate limit (máximo 100 según documentación)
    if (isNaN(limit) || limit < 1 || limit > 100) {
        errors.push('Limit must be between 1 and 100');
    }

    if (errors.length > 0) {
        return res.status(400).json(createValidationError(errors));
    }

    // Add to request for use in controllers
    req.pagination = {
        page,
        limit,
        offset: (page - 1) * limit
    };

    next();
};

/**
 * Validate search parameters - Endpoint GET /users
 */
export const validateUserSearch = (req, res, next) => {
    const { search, role, industry } = req.query;
    const errors = [];

    // Search term validation (opcional)
    if (search && !isValidLength(search, 2, 100)) {
        errors.push('Search term must be between 2 and 100 characters');
    }

    // Role filter validation (opcional)
    if (role && !isValidRole(role)) {
        errors.push('Invalid role filter');
    }

    // Industry filter validation (opcional)
    if (industry && !isValidLength(industry, 1, 100)) {
        errors.push('Industry filter must be between 1 and 100 characters');
    }

    if (errors.length > 0) {
        return res.status(422).json(createValidationError(errors));
    }

    next();
};
```

## 🎯 Validaciones Específicas del Dominio AccessAI

### Sistema de Roles
```javascript
// Roles válidos según el sistema AccessAI
const VALID_ROLES = {
    PERSONA: 'persona',           // Usuario individual
    EMPRESA: 'empresa',           // Empresa comercial  
    PARQUE_INDUSTRIAL: 'parque_industrial', // Parque industrial
    SUPER_ADMIN: 'super_admin'    // Administrador sistema
};

// Validación de campos por rol
export const validateByRole = (roleKey, data) => {
    switch (roleKey) {
        case 'persona':
            return validatePersonaFields(data);
        case 'empresa':
        case 'parque_industrial':
            return validateBusinessFields(data);
        default:
            throw new Error('Invalid role key');
    }
};
```

### Límites de Sistema AccessAI
```javascript
// Límites según la documentación AccessAI
const SYSTEM_LIMITS = {
    ALERT_MESSAGE_MIN: 1,
    ALERT_MESSAGE_MAX: 500,
    DEVICE_NAME_MIN: 1,
    DEVICE_NAME_MAX: 100,
    ZONE_NAME_MIN: 1,
    ZONE_NAME_MAX: 100,
    VISITOR_PURPOSE_MAX: 255
};

// Tipos de dispositivo válidos
const DEVICE_TYPES = ['camera', 'sensor', 'access_point', 'scanner'];

// Estados de alerta
const ALERT_STATUSES = ['pending', 'resolved', 'dismissed'];
```

### Validación de Streams y Datos
```javascript
// Configuración de streams de video
const STREAM_CONFIG = {
    supportedFormats: ['rtsp', 'http', 'rtmp'],
    maxResolution: '1920x1080',
    maxFPS: 30,
    maxBitrate: 5000 // kbps
};

// Configuración de embeddings faciales
const FACE_EMBEDDING_CONFIG = {
    dimensions: 128,
    confidence_threshold: 0.8,
    maxFaces: 10
};
```

## 🚨 **REGLA OBLIGATORIA: Verificar Error Handler**

### ✅ **SIEMPRE verificar errorHandler.js cuando se agrega validación**

**Proceso obligatorio:**
1. **Implementar validación** en service/controller
2. **Agregar throw new Error('mensaje específico')** 
3. **VERIFICAR** que el mensaje esté en `utils/errorHandler.js`
4. **AGREGAR** manejo específico si no existe
5. **CONFIRMAR** código HTTP correcto (400, 404, 409, 403, etc.)

### **Ejemplo de implementación:**
```javascript
// 1. En service - Agregar validación
if (device.zoneId !== user.authorizedZones) {
    throw new Error('User not authorized for this zone');
}

// 2. En errorHandler.js - Agregar manejo
if (err.message === 'User not authorized for this zone') {
    return res.status(403).json({
        error: 'Forbidden',
        message: err.message
    });
}
```

### **Códigos HTTP por tipo de error:**
- **400 Bad Request**: Lógica de negocio inválida
- **404 Not Found**: Recurso no encontrado
- **409 Conflict**: Conflicto de estado (duplicados)
- **403 Forbidden**: Sin permisos
- **422 Unprocessable Entity**: Validación de formato

### **Mensajes que requieren manejo específico:**
- Validaciones de lógica de negocio
- Restricciones de permisos
- Conflictos de estado
- Errores de autorización

## 📚 Documentación de Endpoints

### Actualización Obligatoria
Cada vez que se agreguen nuevos validators:

1. **Actualizar documento** correspondiente en `documentacion/`
2. **Incluir validaciones** en ejemplos de request/response
3. **Documentar errores** 422 con detalles específicos
4. **Actualizar Postman** con casos de validación
5. **✅ VERIFICAR errorHandler.js** para manejo correcto

Esta configuración de validación garantiza entrada de datos segura, consistente y robusta específicamente para el sistema AccessAI.