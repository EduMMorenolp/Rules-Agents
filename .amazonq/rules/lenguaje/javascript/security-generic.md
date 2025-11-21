# Backend - Security Rules

## 🔒 **REGLA OBLIGATORIA: Security First**

### ✅ **Security Headers Obligatorios**
```javascript
// middlewares/security.js
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

export const securityMiddleware = (app) => {
    // Helmet para headers de seguridad
    app.use(helmet({
        contentSecurityPolicy: {
            directives: {
                defaultSrc: ["'self'"],
                styleSrc: ["'self'", "'unsafe-inline'"],
                scriptSrc: ["'self'"],
                imgSrc: ["'self'", "data:", "https:"]
            }
        },
        hsts: {
            maxAge: 31536000,
            includeSubDomains: true,
            preload: true
        }
    }));

    // Rate limiting global
    app.use(rateLimit({
        windowMs: 15 * 60 * 1000, // 15 minutos
        max: 100, // 100 requests por IP
        message: 'Too many requests from this IP'
    }));
};
```

## 🛡️ **Input Validation & Sanitization**

### **Sanitización Obligatoria**
```javascript
// utils/sanitizer.js
import DOMPurify from 'isomorphic-dompurify';
import validator from 'validator';

export const sanitizeInput = (input) => {
    if (typeof input !== 'string') return input;
    
    // Escapar HTML
    const escaped = validator.escape(input);
    
    // Limpiar con DOMPurify
    const clean = DOMPurify.sanitize(escaped);
    
    return clean.trim();
};

export const sanitizeObject = (obj) => {
    const sanitized = {};
    
    for (const [key, value] of Object.entries(obj)) {
        if (typeof value === 'string') {
            sanitized[key] = sanitizeInput(value);
        } else if (typeof value === 'object' && value !== null) {
            sanitized[key] = sanitizeObject(value);
        } else {
            sanitized[key] = value;
        }
    }
    
    return sanitized;
};
```

### **Middleware de Sanitización**
```javascript
// middlewares/sanitize.js
export const sanitizeRequest = (req, res, next) => {
    if (req.body) {
        req.body = sanitizeObject(req.body);
    }
    
    if (req.query) {
        req.query = sanitizeObject(req.query);
    }
    
    if (req.params) {
        req.params = sanitizeObject(req.params);
    }
    
    next();
};
```

## 🔐 **Authentication Security**

### **Password Security**
```javascript
// utils/passwordSecurity.js
import bcrypt from 'bcrypt';
import zxcvbn from 'zxcvbn';

export const SALT_ROUNDS = 12;

export const hashPassword = async (password) => {
    return await bcrypt.hash(password, SALT_ROUNDS);
};

export const verifyPassword = async (password, hash) => {
    return await bcrypt.compare(password, hash);
};

export const checkPasswordStrength = (password) => {
    const result = zxcvbn(password);
    
    return {
        score: result.score, // 0-4
        feedback: result.feedback,
        isStrong: result.score >= 3
    };
};
```

### **JWT Security**
```javascript
// utils/jwtSecurity.js
import jwt from 'jsonwebtoken';
import crypto from 'crypto';

export const generateSecureToken = () => {
    return crypto.randomBytes(32).toString('hex');
};

export const signJWT = (payload, expiresIn = '15m') => {
    return jwt.sign(payload, process.env.JWT_SECRET, {
        expiresIn,
        issuer: process.env.APP_NAME,
        audience: process.env.APP_DOMAIN,
        algorithm: 'HS256'
    });
};

export const verifyJWT = (token) => {
    return jwt.verify(token, process.env.JWT_SECRET, {
        issuer: process.env.APP_NAME,
        audience: process.env.APP_DOMAIN,
        algorithms: ['HS256']
    });
};
```

## 🚫 **SQL Injection Prevention**

### **Parameterized Queries**
```javascript
// ✅ CORRECTO - Usar parámetros
const getUserByEmail = async (email) => {
    return await sequelize.query(
        'SELECT * FROM users WHERE email = :email',
        {
            replacements: { email },
            type: QueryTypes.SELECT
        }
    );
};

// ❌ PROHIBIDO - Concatenación directa
const getUserByEmail = async (email) => {
    return await sequelize.query(
        `SELECT * FROM users WHERE email = '${email}'` // VULNERABLE
    );
};
```

### **ORM Security**
```javascript
// ✅ Usar validaciones de Sequelize
const User = sequelize.define('User', {
    email: {
        type: DataTypes.STRING,
        allowNull: false,
        validate: {
            isEmail: true,
            len: [1, 255]
        }
    },
    age: {
        type: DataTypes.INTEGER,
        validate: {
            min: 0,
            max: 150
        }
    }
});
```

## 🔍 **CORS Security**

### **CORS Configuration**
```javascript
// config/cors.js
export const corsOptions = {
    origin: (origin, callback) => {
        const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [];
        
        if (!origin || allowedOrigins.includes(origin)) {
            callback(null, true);
        } else {
            callback(new Error('Not allowed by CORS'));
        }
    },
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    maxAge: 86400 // 24 horas
};
```

## 📝 **Logging Security**

### **Secure Logging**
```javascript
// utils/secureLogger.js
import winston from 'winston';

// Campos sensibles que NO deben loggearse
const SENSITIVE_FIELDS = [
    'password', 'passwordHash', 'token', 'secret',
    'authorization', 'cookie', 'ssn', 'creditCard'
];

export const sanitizeLogData = (data) => {
    if (typeof data !== 'object' || data === null) return data;
    
    const sanitized = { ...data };
    
    for (const field of SENSITIVE_FIELDS) {
        if (sanitized[field]) {
            sanitized[field] = '[REDACTED]';
        }
    }
    
    return sanitized;
};

export const secureLogger = winston.createLogger({
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
        winston.format.printf(info => {
            return JSON.stringify(sanitizeLogData(info));
        })
    ),
    transports: [
        new winston.transports.File({ filename: 'logs/security.log' })
    ]
});
```

## 🔒 **Environment Security**

### **Environment Variables**
```javascript
// config/environment.js
import dotenv from 'dotenv';

// Cargar variables de entorno
dotenv.config();

// Validar variables críticas
const requiredEnvVars = [
    'JWT_SECRET',
    'DB_PASSWORD',
    'ENCRYPTION_KEY'
];

for (const envVar of requiredEnvVars) {
    if (!process.env[envVar]) {
        throw new Error(`Missing required environment variable: ${envVar}`);
    }
}

// Validar longitud de secretos
if (process.env.JWT_SECRET.length < 32) {
    throw new Error('JWT_SECRET must be at least 32 characters long');
}
```

## 🛡️ **File Upload Security**

### **Secure File Upload**
```javascript
// utils/fileUploadSecurity.js
import path from 'path';
import crypto from 'crypto';

const ALLOWED_MIME_TYPES = [
    'image/jpeg',
    'image/png',
    'image/webp',
    'application/pdf'
];

const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB

export const validateFileUpload = (file) => {
    // Validar tipo MIME
    if (!ALLOWED_MIME_TYPES.includes(file.mimetype)) {
        throw new Error('File type not allowed');
    }
    
    // Validar tamaño
    if (file.size > MAX_FILE_SIZE) {
        throw new Error('File size exceeds limit');
    }
    
    // Validar extensión
    const ext = path.extname(file.originalname).toLowerCase();
    const allowedExts = ['.jpg', '.jpeg', '.png', '.webp', '.pdf'];
    
    if (!allowedExts.includes(ext)) {
        throw new Error('File extension not allowed');
    }
    
    return true;
};

export const generateSecureFilename = (originalName) => {
    const ext = path.extname(originalName);
    const hash = crypto.randomBytes(16).toString('hex');
    return `${hash}${ext}`;
};
```

## 🔐 **Data Encryption**

### **Encryption Utilities**
```javascript
// utils/encryption.js
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');

export const encrypt = (text) => {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipher(ALGORITHM, KEY);
    cipher.setAAD(Buffer.from('additional-data'));
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
        encrypted,
        iv: iv.toString('hex'),
        authTag: authTag.toString('hex')
    };
};

export const decrypt = (encryptedData) => {
    const decipher = crypto.createDecipher(ALGORITHM, KEY);
    decipher.setAAD(Buffer.from('additional-data'));
    decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
    
    let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
};
```

Esta configuración garantiza seguridad robusta en todos los aspectos del backend.