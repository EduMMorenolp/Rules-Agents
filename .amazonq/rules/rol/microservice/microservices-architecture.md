# Microservices - Architecture Rules

## 🏗️ **REGLA OBLIGATORIA: Microservices Design Patterns**

### ✅ **Service Decomposition Strategy**
```
Business Capability Decomposition:
├── User Management Service
├── Product Catalog Service  
├── Order Processing Service
├── Payment Service
├── Notification Service
└── Analytics Service

Domain-Driven Design:
├── Bounded Context per Service
├── Aggregate Root per Service
├── Domain Events for Communication
└── Shared Kernel (minimal)
```

## 🌐 **Communication Patterns**

### **Synchronous Communication**
```javascript
// API Gateway Pattern
// gateway/routes.js
import { createProxyMiddleware } from 'http-proxy-middleware';

export const serviceRoutes = {
  '/api/users': {
    target: process.env.USER_SERVICE_URL,
    changeOrigin: true,
    pathRewrite: { '^/api/users': '' }
  },
  '/api/orders': {
    target: process.env.ORDER_SERVICE_URL,
    changeOrigin: true,
    pathRewrite: { '^/api/orders': '' }
  },
  '/api/payments': {
    target: process.env.PAYMENT_SERVICE_URL,
    changeOrigin: true,
    pathRewrite: { '^/api/payments': '' }
  }
};

// Service-to-Service Communication
// utils/serviceClient.js
class ServiceClient {
    constructor(serviceName, baseURL) {
        this.serviceName = serviceName;
        this.client = axios.create({
            baseURL,
            timeout: 5000,
            headers: {
                'X-Service-Name': process.env.SERVICE_NAME,
                'X-Request-ID': () => uuidv4()
            }
        });
        
        this.setupCircuitBreaker();
        this.setupRetry();
    }
    
    setupCircuitBreaker() {
        this.circuitBreaker = new CircuitBreaker(
            this.client.request.bind(this.client),
            {
                timeout: 3000,
                errorThresholdPercentage: 50,
                resetTimeout: 30000
            }
        );
    }
    
    async get(endpoint, params = {}) {
        return await this.circuitBreaker.fire({
            method: 'GET',
            url: endpoint,
            params
        });
    }
}
```

### **Asynchronous Communication**
```javascript
// Event-Driven Architecture
// events/eventBus.js
import { EventEmitter } from 'events';

class EventBus extends EventEmitter {
    constructor(messageBroker) {
        super();
        this.broker = messageBroker; // RabbitMQ, Kafka, etc.
        this.setupSubscriptions();
    }
    
    async publish(eventType, data, options = {}) {
        const event = {
            id: uuidv4(),
            type: eventType,
            data,
            timestamp: new Date().toISOString(),
            source: process.env.SERVICE_NAME,
            version: '1.0',
            ...options
        };
        
        await this.broker.publish(eventType, event);
        this.emit('event.published', event);
    }
    
    subscribe(eventType, handler) {
        this.broker.subscribe(eventType, async (event) => {
            try {
                await handler(event);
                this.emit('event.processed', event);
            } catch (error) {
                this.emit('event.failed', { event, error });
                throw error;
            }
        });
    }
}

// Domain Events
// events/userEvents.js
export const UserEvents = {
    USER_CREATED: 'user.created',
    USER_UPDATED: 'user.updated',
    USER_DELETED: 'user.deleted',
    USER_VERIFIED: 'user.verified'
};

// Event Handlers
// handlers/orderEventHandlers.js
export class OrderEventHandlers {
    constructor(orderService, emailService) {
        this.orderService = orderService;
        this.emailService = emailService;
    }
    
    async handleUserCreated(event) {
        // Crear carrito de compras para nuevo usuario
        await this.orderService.createShoppingCart(event.data.userId);
    }
    
    async handlePaymentCompleted(event) {
        // Actualizar estado del pedido
        await this.orderService.markAsCompleted(event.data.orderId);
        
        // Enviar confirmación
        await this.emailService.sendOrderConfirmation(event.data);
    }
}
```

## 🗄️ **Data Management Patterns**

### **Database per Service**
```javascript
// Database Configuration per Service
// user-service/config/database.js
export const userServiceDB = {
    host: process.env.USER_DB_HOST,
    database: 'user_service',
    username: process.env.USER_DB_USER,
    password: process.env.USER_DB_PASSWORD,
    dialect: 'postgres',
    pool: {
        max: 10,
        min: 2,
        acquire: 30000,
        idle: 10000
    }
};

// order-service/config/database.js  
export const orderServiceDB = {
    host: process.env.ORDER_DB_HOST,
    database: 'order_service',
    username: process.env.ORDER_DB_USER,
    password: process.env.ORDER_DB_PASSWORD,
    dialect: 'mysql',
    pool: {
        max: 15,
        min: 3,
        acquire: 30000,
        idle: 10000
    }
};
```

### **Saga Pattern for Distributed Transactions**
```javascript
// sagas/orderSaga.js
export class OrderSaga {
    constructor(services) {
        this.userService = services.userService;
        this.inventoryService = services.inventoryService;
        this.paymentService = services.paymentService;
        this.orderService = services.orderService;
    }
    
    async processOrder(orderData) {
        const sagaId = uuidv4();
        const compensations = [];
        
        try {
            // Step 1: Validate user
            const user = await this.userService.validateUser(orderData.userId);
            compensations.push(() => this.userService.releaseUser(orderData.userId));
            
            // Step 2: Reserve inventory
            const reservation = await this.inventoryService.reserveItems(orderData.items);
            compensations.push(() => this.inventoryService.cancelReservation(reservation.id));
            
            // Step 3: Process payment
            const payment = await this.paymentService.processPayment({
                userId: orderData.userId,
                amount: orderData.total,
                sagaId
            });
            compensations.push(() => this.paymentService.refundPayment(payment.id));
            
            // Step 4: Create order
            const order = await this.orderService.createOrder({
                ...orderData,
                paymentId: payment.id,
                reservationId: reservation.id,
                sagaId
            });
            
            return { success: true, orderId: order.id };
            
        } catch (error) {
            // Execute compensations in reverse order
            await this.executeCompensations(compensations.reverse());
            throw new Error(`Order processing failed: ${error.message}`);
        }
    }
    
    async executeCompensations(compensations) {
        for (const compensation of compensations) {
            try {
                await compensation();
            } catch (error) {
                console.error('Compensation failed:', error);
            }
        }
    }
}
```

### **CQRS Pattern**
```javascript
// cqrs/commandHandlers.js
export class UserCommandHandlers {
    constructor(userRepository, eventBus) {
        this.repository = userRepository;
        this.eventBus = eventBus;
    }
    
    async createUser(command) {
        // Validate command
        if (!command.email || !command.password) {
            throw new Error('Invalid user data');
        }
        
        // Create user
        const user = await this.repository.create({
            id: uuidv4(),
            email: command.email,
            passwordHash: await bcrypt.hash(command.password, 12),
            createdAt: new Date()
        });
        
        // Publish event
        await this.eventBus.publish(UserEvents.USER_CREATED, {
            userId: user.id,
            email: user.email,
            createdAt: user.createdAt
        });
        
        return user;
    }
}

// cqrs/queryHandlers.js
export class UserQueryHandlers {
    constructor(readRepository) {
        this.readRepository = readRepository;
    }
    
    async getUserById(query) {
        return await this.readRepository.findById(query.userId);
    }
    
    async getUsersByFilters(query) {
        return await this.readRepository.findByFilters({
            email: query.email,
            status: query.status,
            limit: query.limit || 10,
            offset: query.offset || 0
        });
    }
}
```

## 🔐 **Security Patterns**

### **Service Mesh Security**
```yaml
# istio/security-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: service-to-service-auth
spec:
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/api-gateway"]
    to:
    - operation:
        methods: ["GET", "POST"]
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/user-service"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/orders/*"]
```

### **JWT Service Authentication**
```javascript
// security/serviceAuth.js
export class ServiceAuthenticator {
    constructor(secretKey) {
        this.secretKey = secretKey;
        this.tokenCache = new Map();
    }
    
    generateServiceToken(serviceName, permissions = []) {
        return jwt.sign(
            {
                service: serviceName,
                permissions,
                iat: Math.floor(Date.now() / 1000),
                exp: Math.floor(Date.now() / 1000) + (60 * 60) // 1 hour
            },
            this.secretKey
        );
    }
    
    validateServiceToken(token) {
        try {
            const decoded = jwt.verify(token, this.secretKey);
            return {
                valid: true,
                service: decoded.service,
                permissions: decoded.permissions
            };
        } catch (error) {
            return { valid: false, error: error.message };
        }
    }
    
    middleware() {
        return (req, res, next) => {
            const token = req.headers['x-service-token'];
            
            if (!token) {
                return res.status(401).json({
                    error: 'Service authentication required'
                });
            }
            
            const validation = this.validateServiceToken(token);
            if (!validation.valid) {
                return res.status(401).json({
                    error: 'Invalid service token'
                });
            }
            
            req.service = validation;
            next();
        };
    }
}
```

## 📊 **Observability Patterns**

### **Distributed Tracing**
```javascript
// tracing/tracer.js
import { NodeTracerProvider } from '@opentelemetry/sdk-node';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

export class DistributedTracer {
    constructor(serviceName) {
        this.provider = new NodeTracerProvider({
            resource: new Resource({
                [SemanticResourceAttributes.SERVICE_NAME]: serviceName,
                [SemanticResourceAttributes.SERVICE_VERSION]: process.env.SERVICE_VERSION
            })
        });
        
        this.setupExporters();
        this.tracer = this.provider.getTracer(serviceName);
    }
    
    setupExporters() {
        // Jaeger exporter
        const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
        const jaegerExporter = new JaegerExporter({
            endpoint: process.env.JAEGER_ENDPOINT
        });
        
        this.provider.addSpanProcessor(
            new BatchSpanProcessor(jaegerExporter)
        );
    }
    
    createSpan(name, attributes = {}) {
        return this.tracer.startSpan(name, {
            attributes: {
                'service.name': process.env.SERVICE_NAME,
                ...attributes
            }
        });
    }
    
    middleware() {
        return (req, res, next) => {
            const span = this.createSpan(`${req.method} ${req.path}`, {
                'http.method': req.method,
                'http.url': req.url,
                'http.user_agent': req.get('User-Agent')
            });
            
            req.span = span;
            
            res.on('finish', () => {
                span.setAttributes({
                    'http.status_code': res.statusCode
                });
                span.end();
            });
            
            next();
        };
    }
}
```

### **Service Metrics**
```javascript
// monitoring/metrics.js
import { createPrometheusMetrics } from 'prom-client';

export class ServiceMetrics {
    constructor(serviceName) {
        this.serviceName = serviceName;
        this.setupMetrics();
    }
    
    setupMetrics() {
        this.httpRequestDuration = new Histogram({
            name: 'http_request_duration_seconds',
            help: 'Duration of HTTP requests in seconds',
            labelNames: ['method', 'route', 'status_code', 'service'],
            buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
        });
        
        this.httpRequestTotal = new Counter({
            name: 'http_requests_total',
            help: 'Total number of HTTP requests',
            labelNames: ['method', 'route', 'status_code', 'service']
        });
        
        this.serviceCallDuration = new Histogram({
            name: 'service_call_duration_seconds',
            help: 'Duration of service-to-service calls',
            labelNames: ['target_service', 'method', 'status', 'service']
        });
    }
    
    recordHttpRequest(method, route, statusCode, duration) {
        const labels = {
            method,
            route,
            status_code: statusCode,
            service: this.serviceName
        };
        
        this.httpRequestDuration.observe(labels, duration);
        this.httpRequestTotal.inc(labels);
    }
    
    recordServiceCall(targetService, method, status, duration) {
        this.serviceCallDuration.observe({
            target_service: targetService,
            method,
            status,
            service: this.serviceName
        }, duration);
    }
}
```

## 🚀 **Deployment Patterns**

### **Kubernetes Deployment**
```yaml
# k8s/user-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: myregistry/user-service:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: user-service-secrets
              key: db-host
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP
```

### **Service Discovery**
```javascript
// discovery/serviceRegistry.js
export class ServiceRegistry {
    constructor(consulClient) {
        this.consul = consulClient;
        this.services = new Map();
    }
    
    async registerService(serviceName, serviceId, address, port, healthCheck) {
        const registration = {
            ID: serviceId,
            Name: serviceName,
            Address: address,
            Port: port,
            Check: {
                HTTP: `http://${address}:${port}${healthCheck}`,
                Interval: '10s',
                Timeout: '3s'
            }
        };
        
        await this.consul.agent.service.register(registration);
        console.log(`Service registered: ${serviceName} (${serviceId})`);
    }
    
    async discoverService(serviceName) {
        const services = await this.consul.health.service({
            service: serviceName,
            passing: true
        });
        
        if (services.length === 0) {
            throw new Error(`No healthy instances of ${serviceName} found`);
        }
        
        // Load balancing - round robin
        const service = services[Math.floor(Math.random() * services.length)];
        return {
            address: service.Service.Address,
            port: service.Service.Port
        };
    }
}
```

Esta configuración proporciona patrones fundamentales para arquitectura de microservicios robusta y escalable.