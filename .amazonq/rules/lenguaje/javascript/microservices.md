# JavaScript - Microservices Architecture Rules

## 🏗️ **REGLA OBLIGATORIA: Microservices Pattern**

### ✅ **Service Structure**
```javascript
// services/user-service/
├── src/
│   ├── controllers/
│   ├── services/
│   ├── models/
│   ├── routes/
│   ├── middleware/
│   └── config/
├── package.json
├── Dockerfile
└── docker-compose.yml
```

## 🌐 **Service Communication**

### **HTTP REST Communication**
```javascript
// utils/serviceClient.js
import axios from 'axios';

class ServiceClient {
    constructor(baseURL, timeout = 5000) {
        this.client = axios.create({
            baseURL,
            timeout,
            headers: {
                'Content-Type': 'application/json'
            }
        });
        
        this.setupInterceptors();
    }
    
    setupInterceptors() {
        this.client.interceptors.request.use(
            (config) => {
                config.headers['X-Request-ID'] = generateRequestId();
                config.headers['X-Service-Name'] = process.env.SERVICE_NAME;
                return config;
            },
            (error) => Promise.reject(error)
        );
        
        this.client.interceptors.response.use(
            (response) => response,
            (error) => {
                console.error('Service call failed:', {
                    service: this.client.defaults.baseURL,
                    error: error.message,
                    status: error.response?.status
                });
                return Promise.reject(error);
            }
        );
    }
    
    async get(endpoint, params = {}) {
        const response = await this.client.get(endpoint, { params });
        return response.data;
    }
    
    async post(endpoint, data) {
        const response = await this.client.post(endpoint, data);
        return response.data;
    }
}

// Uso específico
const userService = new ServiceClient(process.env.USER_SERVICE_URL);
const orderService = new ServiceClient(process.env.ORDER_SERVICE_URL);

export { userService, orderService };
```

### **Event-Driven Communication**
```javascript
// utils/eventBus.js
import { EventEmitter } from 'events';
import Redis from 'ioredis';

class EventBus extends EventEmitter {
    constructor() {
        super();
        this.redis = new Redis(process.env.REDIS_URL);
        this.setupSubscriptions();
    }
    
    async publish(event, data) {
        const message = {
            event,
            data,
            timestamp: new Date().toISOString(),
            service: process.env.SERVICE_NAME
        };
        
        await this.redis.publish('events', JSON.stringify(message));
        console.log('Event published:', event);
    }
    
    setupSubscriptions() {
        this.redis.subscribe('events');
        this.redis.on('message', (channel, message) => {
            const { event, data } = JSON.parse(message);
            this.emit(event, data);
        });
    }
}

export const eventBus = new EventBus();

// Uso en services
eventBus.on('user.created', async (userData) => {
    await profileService.createProfile(userData);
});

eventBus.publish('order.completed', { orderId, userId });
```

## 🔐 **Service Authentication**

### **JWT Service-to-Service**
```javascript
// middleware/serviceAuth.js
import jwt from 'jsonwebtoken';

export const serviceAuth = (req, res, next) => {
    const token = req.headers['x-service-token'];
    
    if (!token) {
        return res.status(401).json({
            error: 'Service token required'
        });
    }
    
    try {
        const decoded = jwt.verify(token, process.env.SERVICE_SECRET);
        req.service = decoded;
        next();
    } catch (error) {
        return res.status(401).json({
            error: 'Invalid service token'
        });
    }
};

export const generateServiceToken = (serviceName) => {
    return jwt.sign(
        { service: serviceName },
        process.env.SERVICE_SECRET,
        { expiresIn: '1h' }
    );
};
```

## 📊 **Service Discovery**

### **Service Registry**
```javascript
// utils/serviceRegistry.js
class ServiceRegistry {
    constructor() {
        this.services = new Map();
        this.healthCheckInterval = 30000;
        this.startHealthChecks();
    }
    
    register(serviceName, url, healthEndpoint = '/health') {
        this.services.set(serviceName, {
            url,
            healthEndpoint,
            healthy: true,
            lastCheck: new Date()
        });
        
        console.log(`Service registered: ${serviceName} at ${url}`);
    }
    
    getService(serviceName) {
        const service = this.services.get(serviceName);
        if (!service || !service.healthy) {
            throw new Error(`Service ${serviceName} not available`);
        }
        return service.url;
    }
    
    async startHealthChecks() {
        setInterval(async () => {
            for (const [name, service] of this.services) {
                try {
                    const response = await fetch(`${service.url}${service.healthEndpoint}`);
                    service.healthy = response.ok;
                    service.lastCheck = new Date();
                } catch (error) {
                    service.healthy = false;
                    console.warn(`Service ${name} health check failed`);
                }
            }
        }, this.healthCheckInterval);
    }
}

export const serviceRegistry = new ServiceRegistry();
```

## 🔄 **Circuit Breaker Pattern**

### **Circuit Breaker Implementation**
```javascript
// utils/circuitBreaker.js
class CircuitBreaker {
    constructor(service, options = {}) {
        this.service = service;
        this.failureThreshold = options.failureThreshold || 5;
        this.timeout = options.timeout || 60000;
        
        this.state = 'CLOSED';
        this.failureCount = 0;
        this.nextAttempt = Date.now();
    }
    
    async call(method, ...args) {
        if (this.state === 'OPEN') {
            if (Date.now() < this.nextAttempt) {
                throw new Error('Circuit breaker is OPEN');
            }
            this.state = 'HALF_OPEN';
        }
        
        try {
            const result = await this.service[method](...args);
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }
    
    onSuccess() {
        this.failureCount = 0;
        if (this.state === 'HALF_OPEN') {
            this.state = 'CLOSED';
        }
    }
    
    onFailure() {
        this.failureCount++;
        if (this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
            this.nextAttempt = Date.now() + this.timeout;
        }
    }
}

// Uso
const userServiceBreaker = new CircuitBreaker(userService, {
    failureThreshold: 3,
    timeout: 30000
});

const userData = await userServiceBreaker.call('getUserById', userId);
```

## 📈 **Distributed Tracing**

### **Request Tracing**
```javascript
// middleware/tracing.js
import { v4 as uuidv4 } from 'uuid';

export const tracingMiddleware = (req, res, next) => {
    req.traceId = req.headers['x-trace-id'] || uuidv4();
    req.spanId = uuidv4();
    
    res.setHeader('X-Trace-ID', req.traceId);
    res.setHeader('X-Span-ID', req.spanId);
    
    console.log('Request started', {
        traceId: req.traceId,
        spanId: req.spanId,
        service: process.env.SERVICE_NAME,
        method: req.method,
        url: req.originalUrl
    });
    
    next();
};

export const propagateTrace = (req) => {
    return {
        'X-Trace-ID': req.traceId,
        'X-Parent-Span-ID': req.spanId,
        'X-Service-Name': process.env.SERVICE_NAME
    };
};
```

## 🗄️ **Database per Service**

### **Service-Specific Database**
```javascript
// config/database.js
export const getDatabaseConfig = () => {
    const serviceName = process.env.SERVICE_NAME;
    
    return {
        host: process.env.DB_HOST,
        port: process.env.DB_PORT,
        database: `${serviceName}_${process.env.NODE_ENV}`,
        username: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        dialect: 'postgres',
        logging: false,
        pool: {
            max: 5,
            min: 1,
            acquire: 30000,
            idle: 10000
        }
    };
};
```

## 📊 **Service Metrics**

### **Monitoring Implementation**
```javascript
// utils/metrics.js
class ServiceMetrics {
    constructor() {
        this.metrics = {
            requests: { total: 0, errors: 0 },
            dependencies: new Map(),
            performance: { avgResponseTime: 0 }
        };
    }
    
    recordRequest(success = true, duration = 0) {
        this.metrics.requests.total++;
        if (!success) this.metrics.requests.errors++;
        
        const current = this.metrics.performance.avgResponseTime;
        this.metrics.performance.avgResponseTime = 
            (current + duration) / 2;
    }
    
    recordDependencyCall(service, success = true, duration = 0) {
        if (!this.metrics.dependencies.has(service)) {
            this.metrics.dependencies.set(service, {
                calls: 0, errors: 0, avgDuration: 0
            });
        }
        
        const dep = this.metrics.dependencies.get(service);
        dep.calls++;
        if (!success) dep.errors++;
        dep.avgDuration = (dep.avgDuration + duration) / 2;
    }
    
    getHealthStatus() {
        const errorRate = this.metrics.requests.errors / this.metrics.requests.total;
        
        return {
            status: errorRate > 0.1 ? 'unhealthy' : 'healthy',
            metrics: this.metrics,
            timestamp: new Date().toISOString()
        };
    }
}

export const serviceMetrics = new ServiceMetrics();
```

Esta configuración proporciona una base sólida para arquitectura de microservicios con Node.js.