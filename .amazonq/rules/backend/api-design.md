# AccessAI Backend - API Design Rules

## 🎯 HTTP Status Codes

### Success Codes
```javascript
// 200 - Success (operación exitosa)
res.status(200).json(userData);

// 201 - Created (recurso creado)
res.status(201).json(newUser);

// 204 - No Content (éxito sin contenido)
res.status(204).send();
```

### Client Error Codes
```javascript
// 400 - Bad Request (solicitud malformada)
res.status(400).json({
    error: 'Bad Request',
    message: 'Invalid request format'
});

// 401 - Unauthorized (no autenticado)
res.status(401).json({
    error: 'Unauthorized',
    message: 'Authentication required'
});

// 403 - Forbidden (no autorizado)
res.status(403).json({
    error: 'Forbidden',
    message: 'Insufficient permissions'
});

// 404 - Not Found (recurso no encontrado)
res.status(404).json({
    error: 'Not Found',
    message: 'User not found'
});

// 409 - Conflict (conflicto, ej: email duplicado)
res.status(409).json({
    error: 'Conflict',
    message: 'Email already exists'
});

// 422 - Validation Error (error de validación)
res.status(422).json({
    error: 'Validation Error',
    message: 'Invalid input data',
    details: ['Email is required', 'Password too weak']
});

// 429 - Too Many Requests (rate limiting)
res.status(429).json({
    error: 'Too Many Requests',
    message: 'Rate limit exceeded'
});
```

### Server Error Codes
```javascript
// 500 - Internal Server Error
res.status(500).json({
    error: 'Internal Server Error',
    message: 'An unexpected error occurred'
});

// 503 - Service Unavailable (mantenimiento, BD down)
res.status(503).json({
    error: 'Service Unavailable',
    message: 'Service temporarily unavailable'
});
```

## 📋 Response Formats

### Success Responses
```javascript
// ✅ Respuesta simple de datos
res.status(200).json({
    id: user.id,
    email: user.email,
    firstName: user.firstName,
    roleKey: user.role.key
});

// ✅ Respuesta con paginación
res.status(200).json({
    data: posts,
    pagination: {
        page: parseInt(page),
        limit: parseInt(limit),
        total: count,
        pages: Math.ceil(count / limit)
    }
});

// ✅ Respuesta con metadata
res.status(200).json({
    data: posts,
    meta: {
        total: count,
        hasMore: page * limit < count,
        timestamp: new Date().toISOString()
    }
});
```

### Error Responses
```javascript
// ✅ Error simple
res.status(404).json({
    error: 'Not Found',
    message: 'User not found'
});

// ✅ Error con detalles (validación)
res.status(422).json({
    error: 'Validation Error',
    message: 'Invalid input data',
    details: [
        'Email is required',
        'Password must be at least 8 characters',
        'Invalid phone format'
    ]
});

// ✅ Error con código específico
res.status(409).json({
    error: 'Conflict',
    message: 'Email already exists',
    code: 'EMAIL_DUPLICATE'
});
```

## 📄 Pagination Standards

### Pagination Parameters
```javascript
// Parámetros estándar con validación
const validatePagination = (req, res, next) => {
    let { page = 1, limit = 10 } = req.query;
    
    // Convertir a números y validar
    page = parseInt(page);
    limit = parseInt(limit);
    
    // Validaciones
    if (isNaN(page) || page < 1) {
        return res.status(400).json({
            error: 'Bad Request',
            message: 'Page must be a positive integer'
        });
    }
    
    if (isNaN(limit) || limit < 1 || limit > 100) {
        return res.status(400).json({
            error: 'Bad Request',
            message: 'Limit must be between 1 and 100'
        });
    }
    
    req.pagination = { page, limit, offset: (page - 1) * limit };
    next();
};
```

### Pagination Implementation
```javascript
// services/postService.js
async getPaginatedPosts(page = 1, limit = 10, filters = {}) {
    const offset = (page - 1) * limit;
    
    const { count, rows } = await Post.findAndCountAll({
        where: {
            ...filters,
            isDeleted: false
        },
        include: [{
            model: User,
            as: 'author',
            attributes: ['id', 'firstName', 'lastName', 'avatarUrl']
        }],
        limit: parseInt(limit),
        offset: parseInt(offset),
        order: [['createdAt', 'DESC']],
        distinct: true // Para count correcto con includes
    });
    
    return {
        data: rows,
        pagination: {
            page: parseInt(page),
            limit: parseInt(limit),
            total: count,
            pages: Math.ceil(count / limit),
            hasNext: page * limit < count,
            hasPrev: page > 1
        }
    };
}
```

## 🔍 Filtering & Sorting

### Query Parameters
```javascript
// GET /api/v1/posts?visibility=public&author=123&sort=createdAt&order=desc&page=1&limit=10
// GET /api/v1/contacts/requests/received?status=pending&page=1&limit=10

const buildPostFilters = (query) => {
    const filters = { isDeleted: false };
    
    // Filtro por visibilidad
    if (query.visibility && ['public', 'private', 'connections'].includes(query.visibility)) {
        filters.visibility = query.visibility;
    }
    
    // Filtro por autor
    if (query.author && isValidUUID(query.author)) {
        filters.userId = query.author;
    }
    
    // Filtro por fecha
    if (query.dateFrom) {
        filters.createdAt = { [Op.gte]: new Date(query.dateFrom) };
    }
    
    return filters;
};

const buildSortOrder = (query) => {
    const validSortFields = ['createdAt', 'updatedAt', 'title'];
    const validOrders = ['ASC', 'DESC'];
    
    const sort = validSortFields.includes(query.sort) ? query.sort : 'createdAt';
    const order = validOrders.includes(query.order?.toUpperCase()) ? query.order.toUpperCase() : 'DESC';
    
    return [[sort, order]];
};
```

## 🔗 RESTful Endpoints

### AccessAI API Structure
```javascript
// ✅ Autenticación
POST   /api/v1/auth/register           # Registro de usuarios
POST   /api/v1/auth/login              # Login con email/password
GET    /api/v1/auth/verify             # Verificar token actual
POST   /api/v1/auth/refresh            # Renovar tokens
POST   /api/v1/auth/logout             # Cerrar sesión

// ✅ Gestión de usuarios
GET    /api/v1/users                   # Listar usuarios
GET    /api/v1/users/:id               # Usuario específico
POST   /api/v1/users                   # Crear usuario
PUT    /api/v1/users/:id               # Actualizar usuario
DELETE /api/v1/users/:id               # Eliminar usuario

// ✅ Dispositivos y zonas
GET    /api/v1/devices                 # Listar dispositivos
GET    /api/v1/devices/:id             # Dispositivo específico
POST   /api/v1/devices                 # Registrar dispositivo
PUT    /api/v1/devices/:id             # Actualizar dispositivo
DELETE /api/v1/devices/:id             # Eliminar dispositivo
GET    /api/v1/zones                   # Listar zonas
POST   /api/v1/zones                   # Crear zona
PUT    /api/v1/zones/:id               # Actualizar zona

// ✅ Alertas y monitoreo
GET    /api/v1/alerts                  # Listar alertas
GET    /api/v1/alerts/:id              # Alerta específica
POST   /api/v1/alerts                  # Crear alerta
PATCH  /api/v1/alerts/:id/resolve      # Resolver alerta
DELETE /api/v1/alerts/:id              # Eliminar alerta
GET    /api/v1/alerts/stats            # Estadísticas de alertas

// ✅ Visitantes temporales
GET    /api/v1/visitors                # Listar visitantes
GET    /api/v1/visitors/:id            # Visitante específico
POST   /api/v1/visitors                # Registrar visitante
PUT    /api/v1/visitors/:id            # Actualizar visitante
DELETE /api/v1/visitors/:id            # Eliminar visitante
GET    /api/v1/visitors/:id/qr         # Generar QR de acceso

// ✅ Logs de acceso
GET    /api/v1/access-logs             # Historial de accesos
GET    /api/v1/access-logs/:id         # Log específico
POST   /api/v1/access-logs             # Registrar acceso
GET    /api/v1/access-logs/stats       # Estadísticas de acceso

// ✅ Integración con IA
POST   /api/v1/ai/process-frame        # Procesar frame de cámara
GET    /api/v1/ai/models               # Listar modelos IA
POST   /api/v1/ai/models               # Crear modelo IA
PUT    /api/v1/ai/models/:id           # Actualizar modelo
GET    /api/v1/ai/face-embeddings      # Embeddings faciales
POST   /api/v1/ai/face-embeddings      # Crear embedding

// ✅ Streaming y WebSocket
GET    /api/v1/streaming/status        # Estado de streams
POST   /api/v1/streaming/start         # Iniciar stream
POST   /api/v1/streaming/stop          # Detener stream
WS     /ws/alerts                      # WebSocket alertas tiempo real
WS     /ws/monitoring                  # WebSocket monitoreo

// ✅ Sistema
GET    /api/v1/health                  # Health check
GET    /api/v1/metrics                 # Métricas del sistema
```

## 📊 API Versioning

### URL Versioning
```javascript
// ✅ Versioning en URL
const apiVersion = process.env.VERSION || '1';
app.use(`/api/v${apiVersion}`, routes);

// Estructura de rutas versionadas
/api/v1/users
/api/v1/alerts
/api/v1/auth/login
```

### Version Middleware
```javascript
// middlewares/version.js
export const versionMiddleware = (req, res, next) => {
    const version = req.baseUrl.match(/\/api\/v(\d+)/)?.[1];
    req.apiVersion = version || '1';
    next();
};
```

## 🔒 Security Headers

### Standard Security Headers
```javascript
// middlewares/security.js
export const securityHeaders = (req, res, next) => {
    // Prevenir XSS
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('X-Frame-Options', 'DENY');
    res.setHeader('X-XSS-Protection', '1; mode=block');
    
    // HTTPS enforcement en producción
    if (process.env.NODE_ENV === 'production') {
        res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    }
    
    next();
};
```

## 📈 Rate Limiting

### Rate Limiting Configuration
```javascript
// config/rateLimiting.js
import rateLimit from 'express-rate-limit';

// Rate limiting general
export const generalLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutos
    max: 100, // 100 requests por IP
    message: {
        error: 'Too Many Requests',
        message: 'Too many requests from this IP'
    }
});

// Rate limiting para auth
export const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutos
    max: 5, // 5 intentos de login por IP
    message: {
        error: 'Too Many Requests',
        message: 'Too many login attempts'
    }
});

// Rate limiting para creación de alertas
export const alertLimiter = rateLimit({
    windowMs: 60 * 1000, // 1 minuto
    max: 20, // 20 alertas por minuto
    message: {
        error: 'Too Many Requests',
        message: 'Too many alerts created'
    }
});
```

## 🏥 Health Check

### Health Check Endpoint
```javascript
// routes/healthRoutes.js
import { Router } from 'express';
import { sequelize } from '../../database/connection.js';

const router = Router();

router.get('/health', async (req, res) => {
    const healthCheck = {
        status: 'healthy',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
        services: {}
    };
    
    try {
        // Verificar conexión a BD
        await sequelize.authenticate();
        healthCheck.services.database = 'connected';
    } catch (error) {
        healthCheck.services.database = 'disconnected';
        healthCheck.status = 'unhealthy';
    }
    
    // Verificar memoria
    const memUsage = process.memoryUsage();
    healthCheck.memory = {
        used: Math.round(memUsage.heapUsed / 1024 / 1024) + ' MB',
        total: Math.round(memUsage.heapTotal / 1024 / 1024) + ' MB'
    };
    
    const statusCode = healthCheck.status === 'healthy' ? 200 : 503;
    res.status(statusCode).json(healthCheck);
});

export default router;
```

## 📝 Request/Response Logging

### Logging Middleware
```javascript
// middlewares/logger.js
export const requestLogger = (req, res, next) => {
    const start = Date.now();
    const timestamp = new Date().toISOString();
    
    // Log request
    console.log({
        type: 'REQUEST',
        timestamp,
        method: req.method,
        url: req.url,
        ip: req.ip,
        userAgent: req.get('User-Agent')
    });
    
    // Log response cuando termine
    res.on('finish', () => {
        const duration = Date.now() - start;
        console.log({
            type: 'RESPONSE',
            timestamp: new Date().toISOString(),
            method: req.method,
            url: req.url,
            status: res.statusCode,
            duration: `${duration}ms`
        });
    });
    
    next();
};
```

## 📚 Documentación de Endpoints

### Documentos Separados por Sección
Cada vez que se modifique o agregue funcionalidad, actualizar los documentos correspondientes:

```
documentacion/
├── AUTENTICACION-ENDPOINTS.md          # 🔐 Auth endpoints
├── USUARIOS-ENDPOINTS.md               # 👥 User management
├── DISPOSITIVOS-ENDPOINTS.md           # 📹 Device management
├── ALERTAS-ENDPOINTS.md                # 🚨 Alert system
├── VISITANTES-ENDPOINTS.md             # 👤 Visitor management
├── ACCESO-LOGS-ENDPOINTS.md            # 📊 Access logs
├── IA-ENDPOINTS.md                     # 🤖 AI integration
├── STREAMING-ENDPOINTS.md              # 📡 Video streaming
├── ZONAS-ENDPOINTS.md                  # 🏢 Zone management
└── SISTEMA-ENDPOINTS.md                # 📊 System health
```

### Estructura de Documentación
Cada documento debe incluir:
- **Para qué sirve** cada endpoint
- **Datos de entrada** con ejemplos JSON
- **Respuestas** con códigos HTTP y ejemplos
- **Características específicas** del sistema
- **Validaciones y restricciones**

### Proceso de Actualización Obligatorio
1. **Implementar funcionalidad** (controller, service, routes, validators)
2. **Actualizar Postman collection** (AccessAI_postman.json) con nuevos endpoints
3. **Crear/actualizar documento** específico de la sección correspondiente
4. **Actualizar database-er-diagram.md** si hay cambios de base de datos
5. **Verificar consistencia** entre documentación, Postman y código
6. **Ejecutar tests** para validar funcionalidad

Esta configuración de API garantiza consistencia, escalabilidad y mantenibilidad en el diseño de endpoints del backend AccessAI.