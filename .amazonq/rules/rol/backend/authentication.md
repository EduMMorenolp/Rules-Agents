# AccessAI Backend - Authentication Rules

## 🔐 JWT with Refresh Tokens

### Token Configuration
```javascript
// config/tokens.js
export const TOKEN_CONFIG = {
    ACCESS_TOKEN_EXPIRES_IN: '15m',
    REFRESH_TOKEN_EXPIRES_IN: '7d',
    REFRESH_TOKEN_EXPIRES_MS: 7 * 24 * 60 * 60 * 1000, // 7 días
    ACCESS_TOKEN_SECRET: process.env.SECRET_JWT_KEY,
    REFRESH_TOKEN_SECRET: process.env.SECRET_JWT_KEY
};
```

### Token Generation with JTI
```javascript
// services/authService.js
import { v4 as uuidv4 } from 'uuid';
import jwt from 'jsonwebtoken';

class AuthService {
    /**
     * Generate access token (15 minutes)
     */
    generateAccessToken(user) {
        return jwt.sign(
            {
                userId: user.id,
                email: user.email,
                roleId: user.roleId
            },
            TOKEN_CONFIG.ACCESS_TOKEN_SECRET,
            { expiresIn: TOKEN_CONFIG.ACCESS_TOKEN_EXPIRES_IN }
        );
    }

    /**
     * Generate refresh token with unique JTI
     */
    generateRefreshToken(user, jti) {
        return jwt.sign(
            {
                userId: user.id,
                email: user.email,
                roleId: user.roleId,
                jti // JWT ID único para tracking
            },
            TOKEN_CONFIG.REFRESH_TOKEN_SECRET,
            { expiresIn: TOKEN_CONFIG.REFRESH_TOKEN_EXPIRES_IN }
        );
    }

    /**
     * Generate token pair with rotation
     */
    async generateTokenPair(user) {
        const jti = uuidv4();
        const accessToken = this.generateAccessToken(user);
        const refreshToken = this.generateRefreshToken(user, jti);

        // Almacenar refresh token hasheado en BD
        await RefreshToken.create({
            userId: user.id,
            tokenHash: refreshToken, // Se hashea automáticamente por hook
            jti,
            expiresAt: new Date(Date.now() + TOKEN_CONFIG.REFRESH_TOKEN_EXPIRES_MS)
        });

        return { accessToken, refreshToken };
    }
}
```

## 🍪 Cookie Management

### Cookie Configuration
```javascript
// Configuración estándar de cookies
const cookieOptions = {
    httpOnly: true, // No accesible desde JavaScript
    secure: process.env.NODE_ENV === 'production', // HTTPS en producción
    sameSite: 'strict' // Protección CSRF
};

// Set access token (15 minutos)
res.cookie('accessToken', accessToken, {
    ...cookieOptions,
    maxAge: 15 * 60 * 1000
});

// Set refresh token (7 días, path específico)
const apiVersion = process.env.VERSION || '1';
res.cookie('refreshToken', refreshToken, {
    ...cookieOptions,
    maxAge: TOKEN_CONFIG.REFRESH_TOKEN_EXPIRES_MS,
    path: `/api/v${apiVersion}/auth/refresh` // Path específico
});
```

### Cookie Clearing
```javascript
// controllers/authController.js
async logout(req, res) {
    // Revocar todos los refresh tokens del usuario
    await authService.revokeAllRefreshTokens(req.user.userId);
    
    // Limpiar cookies con mismas opciones
    res.clearCookie('accessToken', {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict'
    });
    
    const apiVersion = process.env.VERSION || '1';
    res.clearCookie('refreshToken', {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict',
        path: `/api/v${apiVersion}/auth/refresh`
    });
    
    res.status(200).json({ message: 'Logout successful' });
}
```

## 🔄 Token Rotation

### Refresh Token Flow
```javascript
// services/authService.js
async refreshTokens(oldRefreshToken) {
    try {
        // Verificar token viejo
        const decoded = jwt.verify(oldRefreshToken, TOKEN_CONFIG.REFRESH_TOKEN_SECRET);

        // Buscar en BD por JTI
        const storedToken = await RefreshToken.findOne({
            where: { jti: decoded.jti, isRevoked: false },
            include: [{ model: User, as: 'user' }]
        });

        if (!storedToken) {
            throw new Error('Invalid refresh token');
        }

        // Validar hash del token
        const isValidToken = await storedToken.validateToken(oldRefreshToken);
        if (!isValidToken) {
            throw new Error('Invalid refresh token');
        }

        // Verificar expiración
        if (storedToken.expiresAt < new Date()) {
            throw new Error('Refresh token expired');
        }

        // REVOCAR token viejo (rotación)
        await storedToken.update({ isRevoked: true });

        // Generar nuevo par de tokens
        const { accessToken, refreshToken } = await this.generateTokenPair(storedToken.user);

        return { accessToken, refreshToken, user: storedToken.user };
    } catch (error) {
        if (error.name === 'TokenExpiredError') {
            throw new Error('Refresh token expired');
        }
        if (error.name === 'JsonWebTokenError') {
            throw new Error('Invalid refresh token');
        }
        throw error;
    }
}
```

## 🔒 Password Security

### Password Validation
```javascript
// validators/baseValidator.js
export const isValidPassword = (password) => {
    // Mínimo 8 chars, mayúscula, número, carácter especial
    const passwordRegex = /^(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*(),.?\":{}|<>]).{8,}$/;
    return passwordRegex.test(password);
};
```

### Password Hashing
```javascript
// models/user.js - Hook de Sequelize
const User = sequelize.define('User', {
    passwordHash: {
        type: DataTypes.STRING,
        allowNull: false
    }
}, {
    hooks: {
        beforeCreate: async (user) => {
            if (user.passwordHash) {
                user.passwordHash = await bcrypt.hash(user.passwordHash, 12); // 12 rounds
            }
        },
        beforeUpdate: async (user) => {
            if (user.changed('passwordHash')) {
                user.passwordHash = await bcrypt.hash(user.passwordHash, 12);
            }
        }
    }
});

// Método de instancia para verificar password
User.prototype.checkPassword = async function(password) {
    return await bcrypt.compare(password, this.passwordHash);
};
```

## 🛡️ Auth Middleware

### JWT Verification Middleware
```javascript
// middlewares/auth.js
import jwt from 'jsonwebtoken';
import { TOKEN_CONFIG } from '../config/tokens.js';

export const authenticateToken = (req, res, next) => {
    // Obtener token de header o cookie
    let token = req.headers.authorization?.split(' ')[1]; // Bearer token
    
    if (!token && req.cookies) {
        token = req.cookies.accessToken; // Cookie fallback
    }

    if (!token) {
        return res.status(401).json({
            error: 'Unauthorized',
            message: 'Access token required'
        });
    }

    try {
        const decoded = jwt.verify(token, TOKEN_CONFIG.ACCESS_TOKEN_SECRET);
        req.user = decoded; // { userId, email, roleId }
        next();
    } catch (error) {
        if (error.name === 'TokenExpiredError') {
            return res.status(401).json({
                error: 'Unauthorized',
                message: 'Token expired'
            });
        }
        
        return res.status(401).json({
            error: 'Unauthorized',
            message: 'Invalid token'
        });
    }
};
```

### Role-Based Authorization
```javascript
// middlewares/auth.js
export const requireRole = (allowedRoles) => {
    return async (req, res, next) => {
        try {
            const user = await User.findByPk(req.user.userId, {
                include: [{ model: Role, as: 'role' }]
            });

            if (!user || !allowedRoles.includes(user.role.key)) {
                return res.status(403).json({
                    error: 'Forbidden',
                    message: 'Insufficient permissions'
                });
            }

            req.userRole = user.role.key;
            next();
        } catch (error) {
            return res.status(500).json({
                error: 'Internal Server Error',
                message: 'Authorization check failed'
            });
        }
    };
};

// Uso en rutas
router.post('/admin-only', 
    authenticateToken, 
    requireRole(['super_admin']), 
    controller.adminMethod
);
```

## 📧 Email Verification

### Verification Token System
```javascript
// services/tokenService.js
class TokenService {
    /**
     * Create email verification token (24h expiry)
     */
    async createEmailVerificationToken(userId) {
        const token = crypto.randomBytes(32).toString('hex');
        const hashedToken = await bcrypt.hash(token, 10);
        
        await UserToken.create({
            userId,
            tokenHash: hashedToken,
            type: 'email_verification',
            expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000) // 24h
        });
        
        return token;
    }

    /**
     * Verify email verification token
     */
    async verifyEmailToken(token) {
        const userTokens = await UserToken.findAll({
            where: {
                type: 'email_verification',
                usedAt: null,
                expiresAt: { [Op.gt]: new Date() }
            }
        });

        for (const userToken of userTokens) {
            const isValid = await bcrypt.compare(token, userToken.tokenHash);
            if (isValid) {
                // Marcar como usado
                await userToken.update({ usedAt: new Date() });
                return userToken.userId;
            }
        }

        throw new Error('Invalid or expired token');
    }
}
```

## 🔐 Security Headers & CORS

### CORS Configuration
```javascript
// config/cors.js
export const corsOptions = {
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    credentials: true, // Permitir cookies
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    exposedHeaders: ['Set-Cookie']
};
```

### Security Best Practices
```javascript
// Rate limiting para auth endpoints
import rateLimit from 'express-rate-limit';

export const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutos
    max: 5, // 5 intentos por IP
    message: {
        error: 'Too Many Requests',
        message: 'Too many login attempts, please try again later'
    },
    standardHeaders: true,
    legacyHeaders: false
});

// Aplicar en rutas sensibles
router.post('/login', authLimiter, validateUserLogin, authController.login);
router.post('/register', authLimiter, validateUserRegistration, authController.register);
```

Esta configuración de autenticación garantiza seguridad robusta con tokens JWT, rotación automática y protección contra ataques comunes en el sistema AccessAI.