# Backend - Monitoring & Observability Rules

## 📊 **REGLA OBLIGATORIA: Observabilidad Completa**

### ✅ **Structured Logging**

#### **Winston Logger Setup**
```javascript
// config/logger.js
import winston from 'winston';

const logFormat = winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json(),
    winston.format.printf(({ timestamp, level, message, ...meta }) => {
        return JSON.stringify({
            timestamp,
            level,
            message,
            ...meta,
            service: process.env.SERVICE_NAME || 'backend',
            version: process.env.APP_VERSION || '1.0.0',
            environment: process.env.NODE_ENV || 'development'
        });
    })
);

export const logger = winston.createLogger({
    level: process.env.LOG_LEVEL || 'info',
    format: logFormat,
    transports: [
        new winston.transports.Console(),
        new winston.transports.File({ 
            filename: 'logs/error.log', 
            level: 'error' 
        }),
        new winston.transports.File({ 
            filename: 'logs/combined.log' 
        })
    ]
});
```

#### **Request Logging Middleware**
```javascript
// middlewares/requestLogger.js
import { logger } from '../config/logger.js';

export const requestLogger = (req, res, next) => {
    const startTime = Date.now();
    const requestId = req.headers['x-request-id'] || generateRequestId();
    
    // Agregar requestId al request
    req.requestId = requestId;
    
    // Log request inicial
    logger.info('Request started', {
        requestId,
        method: req.method,
        url: req.originalUrl,
        userAgent: req.get('User-Agent'),
        ip: req.ip,
        userId: req.user?.userId
    });
    
    // Log response cuando termine
    res.on('finish', () => {
        const duration = Date.now() - startTime;
        
        logger.info('Request completed', {
            requestId,
            method: req.method,
            url: req.originalUrl,
            status: res.statusCode,
            duration: `${duration}ms`,
            userId: req.user?.userId
        });
    });
    
    next();
};

const generateRequestId = () => {
    return Math.random().toString(36).substring(2, 15);
};
```

## 📈 **Application Metrics**

### **Custom Metrics Collection**
```javascript
// utils/metrics.js
class MetricsCollector {
    constructor() {
        this.metrics = {
            requests: { total: 0, byStatus: {} },
            database: { queries: 0, slowQueries: 0 },
            errors: { total: 0, byType: {} },
            performance: { avgResponseTime: 0, p95ResponseTime: 0 }
        };
        this.responseTimes = [];
    }
    
    recordRequest(status, duration) {
        this.metrics.requests.total++;
        this.metrics.requests.byStatus[status] = 
            (this.metrics.requests.byStatus[status] || 0) + 1;
        
        this.responseTimes.push(duration);
        this.updatePerformanceMetrics();
    }
    
    recordDatabaseQuery(duration) {
        this.metrics.database.queries++;
        if (duration > 1000) {
            this.metrics.database.slowQueries++;
        }
    }
    
    recordError(errorType) {
        this.metrics.errors.total++;
        this.metrics.errors.byType[errorType] = 
            (this.metrics.errors.byType[errorType] || 0) + 1;
    }
    
    updatePerformanceMetrics() {
        if (this.responseTimes.length === 0) return;
        
        const sorted = this.responseTimes.sort((a, b) => a - b);
        const avg = sorted.reduce((a, b) => a + b, 0) / sorted.length;
        const p95Index = Math.floor(sorted.length * 0.95);
        
        this.metrics.performance.avgResponseTime = Math.round(avg);
        this.metrics.performance.p95ResponseTime = sorted[p95Index] || 0;
        
        // Mantener solo últimas 1000 mediciones
        if (this.responseTimes.length > 1000) {
            this.responseTimes = this.responseTimes.slice(-1000);
        }
    }
    
    getMetrics() {
        return {
            ...this.metrics,
            timestamp: new Date().toISOString(),
            uptime: process.uptime(),
            memory: process.memoryUsage()
        };
    }
    
    reset() {
        this.metrics = {
            requests: { total: 0, byStatus: {} },
            database: { queries: 0, slowQueries: 0 },
            errors: { total: 0, byType: {} },
            performance: { avgResponseTime: 0, p95ResponseTime: 0 }
        };
        this.responseTimes = [];
    }
}

export const metricsCollector = new MetricsCollector();
```

### **Metrics Endpoint**
```javascript
// routes/metrics.js
import { Router } from 'express';
import { metricsCollector } from '../utils/metrics.js';

const router = Router();

router.get('/metrics', (req, res) => {
    const metrics = metricsCollector.getMetrics();
    res.json(metrics);
});

router.get('/health', async (req, res) => {
    const healthCheck = {
        status: 'healthy',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
        version: process.env.APP_VERSION || '1.0.0',
        environment: process.env.NODE_ENV || 'development',
        services: {}
    };
    
    try {
        // Check database
        await sequelize.authenticate();
        healthCheck.services.database = 'connected';
    } catch (error) {
        healthCheck.services.database = 'disconnected';
        healthCheck.status = 'unhealthy';
    }
    
    // Check Redis (if used)
    try {
        if (redis) {
            await redis.ping();
            healthCheck.services.redis = 'connected';
        }
    } catch (error) {
        healthCheck.services.redis = 'disconnected';
        healthCheck.status = 'unhealthy';
    }
    
    const statusCode = healthCheck.status === 'healthy' ? 200 : 503;
    res.status(statusCode).json(healthCheck);
});

export default router;
```

## 🚨 **Error Tracking**

### **Error Handler with Tracking**
```javascript
// middlewares/errorHandler.js
import { logger } from '../config/logger.js';
import { metricsCollector } from '../utils/metrics.js';

export const errorHandler = (err, req, res, next) => {
    const errorId = generateErrorId();
    
    // Log error con contexto completo
    logger.error('Application error', {
        errorId,
        requestId: req.requestId,
        message: err.message,
        stack: err.stack,
        url: req.originalUrl,
        method: req.method,
        userId: req.user?.userId,
        body: sanitizeBody(req.body),
        query: req.query,
        params: req.params
    });
    
    // Registrar métrica de error
    metricsCollector.recordError(err.name || 'UnknownError');
    
    // Respuesta según tipo de error
    if (err.name === 'ValidationError') {
        return res.status(422).json({
            error: 'Validation Error',
            message: err.message,
            errorId
        });
    }
    
    if (err.name === 'SequelizeUniqueConstraintError') {
        return res.status(409).json({
            error: 'Conflict',
            message: 'Resource already exists',
            errorId
        });
    }
    
    // Error genérico
    res.status(500).json({
        error: 'Internal Server Error',
        message: 'An unexpected error occurred',
        errorId
    });
};

const generateErrorId = () => {
    return `err_${Date.now()}_${Math.random().toString(36).substring(2, 8)}`;
};

const sanitizeBody = (body) => {
    if (!body) return body;
    
    const sanitized = { ...body };
    const sensitiveFields = ['password', 'token', 'secret'];
    
    sensitiveFields.forEach(field => {
        if (sanitized[field]) {
            sanitized[field] = '[REDACTED]';
        }
    });
    
    return sanitized;
};
```

## 📊 **Performance Monitoring**

### **Performance Middleware**
```javascript
// middlewares/performanceMonitor.js
import { logger } from '../config/logger.js';
import { metricsCollector } from '../utils/metrics.js';

export const performanceMonitor = (req, res, next) => {
    const startTime = Date.now();
    const startMemory = process.memoryUsage();
    
    res.on('finish', () => {
        const duration = Date.now() - startTime;
        const endMemory = process.memoryUsage();
        const memoryDelta = endMemory.heapUsed - startMemory.heapUsed;
        
        // Registrar métricas
        metricsCollector.recordRequest(res.statusCode, duration);
        
        // Log performance data
        const performanceData = {
            requestId: req.requestId,
            method: req.method,
            url: req.originalUrl,
            status: res.statusCode,
            duration: `${duration}ms`,
            memoryDelta: `${Math.round(memoryDelta / 1024)}KB`,
            userId: req.user?.userId
        };
        
        // Log requests lentos
        if (duration > 1000) {
            logger.warn('Slow request detected', performanceData);
        } else {
            logger.info('Request performance', performanceData);
        }
    });
    
    next();
};
```

## 🔍 **Database Query Monitoring**

### **Query Performance Tracker**
```javascript
// utils/queryMonitor.js
import { logger } from '../config/logger.js';
import { metricsCollector } from '../utils/metrics.js';

export const setupQueryMonitoring = (sequelize) => {
    sequelize.addHook('beforeQuery', (options) => {
        options.startTime = Date.now();
    });
    
    sequelize.addHook('afterQuery', (options, query) => {
        const duration = Date.now() - options.startTime;
        
        // Registrar métrica
        metricsCollector.recordDatabaseQuery(duration);
        
        // Log query performance
        const queryData = {
            sql: query.sql,
            duration: `${duration}ms`,
            type: options.type,
            model: options.model?.name
        };
        
        if (duration > 500) {
            logger.warn('Slow database query', queryData);
        } else if (process.env.LOG_LEVEL === 'debug') {
            logger.debug('Database query', queryData);
        }
    });
};
```

## 📱 **Alerting System**

### **Alert Manager**
```javascript
// utils/alertManager.js
import { logger } from '../config/logger.js';

class AlertManager {
    constructor() {
        this.thresholds = {
            errorRate: 0.05, // 5% error rate
            responseTime: 2000, // 2 seconds
            memoryUsage: 0.8, // 80% memory usage
            diskUsage: 0.9 // 90% disk usage
        };
        
        this.alertCooldown = new Map();
    }
    
    checkMetrics(metrics) {
        const alerts = [];
        
        // Check error rate
        const errorRate = metrics.errors.total / metrics.requests.total;
        if (errorRate > this.thresholds.errorRate) {
            alerts.push({
                type: 'HIGH_ERROR_RATE',
                message: `Error rate is ${(errorRate * 100).toFixed(2)}%`,
                severity: 'critical'
            });
        }
        
        // Check response time
        if (metrics.performance.p95ResponseTime > this.thresholds.responseTime) {
            alerts.push({
                type: 'HIGH_RESPONSE_TIME',
                message: `P95 response time is ${metrics.performance.p95ResponseTime}ms`,
                severity: 'warning'
            });
        }
        
        // Check memory usage
        const memoryUsage = metrics.memory.heapUsed / metrics.memory.heapTotal;
        if (memoryUsage > this.thresholds.memoryUsage) {
            alerts.push({
                type: 'HIGH_MEMORY_USAGE',
                message: `Memory usage is ${(memoryUsage * 100).toFixed(2)}%`,
                severity: 'warning'
            });
        }
        
        // Send alerts
        alerts.forEach(alert => this.sendAlert(alert));
        
        return alerts;
    }
    
    sendAlert(alert) {
        const alertKey = `${alert.type}_${alert.severity}`;
        const now = Date.now();
        const lastAlert = this.alertCooldown.get(alertKey);
        
        // Cooldown de 5 minutos para evitar spam
        if (lastAlert && now - lastAlert < 5 * 60 * 1000) {
            return;
        }
        
        this.alertCooldown.set(alertKey, now);
        
        logger.error('System alert triggered', {
            ...alert,
            timestamp: new Date().toISOString(),
            service: process.env.SERVICE_NAME || 'backend'
        });
        
        // Aquí se puede integrar con servicios como Slack, PagerDuty, etc.
        if (process.env.SLACK_WEBHOOK_URL) {
            this.sendSlackAlert(alert);
        }
    }
    
    async sendSlackAlert(alert) {
        try {
            const payload = {
                text: `🚨 ${alert.type}`,
                attachments: [{
                    color: alert.severity === 'critical' ? 'danger' : 'warning',
                    fields: [{
                        title: 'Message',
                        value: alert.message,
                        short: false
                    }, {
                        title: 'Service',
                        value: process.env.SERVICE_NAME || 'backend',
                        short: true
                    }, {
                        title: 'Environment',
                        value: process.env.NODE_ENV || 'development',
                        short: true
                    }]
                }]
            };
            
            await fetch(process.env.SLACK_WEBHOOK_URL, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });
        } catch (error) {
            logger.error('Failed to send Slack alert', { error: error.message });
        }
    }
}

export const alertManager = new AlertManager();
```

## 📊 **Monitoring Dashboard Data**

### **Dashboard Endpoint**
```javascript
// routes/dashboard.js
import { Router } from 'express';
import { metricsCollector } from '../utils/metrics.js';
import { alertManager } from '../utils/alertManager.js';

const router = Router();

router.get('/dashboard', (req, res) => {
    const metrics = metricsCollector.getMetrics();
    const alerts = alertManager.checkMetrics(metrics);
    
    const dashboardData = {
        overview: {
            status: alerts.some(a => a.severity === 'critical') ? 'critical' : 
                   alerts.some(a => a.severity === 'warning') ? 'warning' : 'healthy',
            uptime: formatUptime(metrics.uptime),
            version: process.env.APP_VERSION || '1.0.0',
            environment: process.env.NODE_ENV || 'development'
        },
        requests: {
            total: metrics.requests.total,
            successRate: calculateSuccessRate(metrics.requests.byStatus),
            avgResponseTime: metrics.performance.avgResponseTime,
            p95ResponseTime: metrics.performance.p95ResponseTime
        },
        database: {
            totalQueries: metrics.database.queries,
            slowQueries: metrics.database.slowQueries,
            slowQueryRate: metrics.database.queries > 0 ? 
                (metrics.database.slowQueries / metrics.database.queries * 100).toFixed(2) : 0
        },
        errors: {
            total: metrics.errors.total,
            errorRate: metrics.requests.total > 0 ? 
                (metrics.errors.total / metrics.requests.total * 100).toFixed(2) : 0,
            byType: metrics.errors.byType
        },
        system: {
            memory: {
                used: Math.round(metrics.memory.heapUsed / 1024 / 1024),
                total: Math.round(metrics.memory.heapTotal / 1024 / 1024),
                usage: ((metrics.memory.heapUsed / metrics.memory.heapTotal) * 100).toFixed(2)
            }
        },
        alerts: alerts
    };
    
    res.json(dashboardData);
});

const formatUptime = (seconds) => {
    const days = Math.floor(seconds / 86400);
    const hours = Math.floor((seconds % 86400) / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    
    return `${days}d ${hours}h ${minutes}m`;
};

const calculateSuccessRate = (statusCounts) => {
    const total = Object.values(statusCounts).reduce((a, b) => a + b, 0);
    const successful = Object.entries(statusCounts)
        .filter(([status]) => status.startsWith('2'))
        .reduce((sum, [, count]) => sum + count, 0);
    
    return total > 0 ? ((successful / total) * 100).toFixed(2) : 100;
};

export default router;
```

Esta configuración proporciona observabilidad completa del sistema backend.