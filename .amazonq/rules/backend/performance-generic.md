# Backend - Performance Rules

## ⚡ **REGLA OBLIGATORIA: Performance First**

### ✅ **Database Query Optimization**

#### **Promise.all para Operaciones Paralelas**
```javascript
// ✅ CORRECTO - Operaciones paralelas
const [user, posts, followers] = await Promise.all([
    User.findByPk(userId),
    Post.findAll({ where: { userId } }),
    Follow.count({ where: { followingId: userId } })
]);

// ❌ PROHIBIDO - Operaciones secuenciales
const user = await User.findByPk(userId);
const posts = await Post.findAll({ where: { userId } });
const followers = await Follow.count({ where: { followingId: userId } });
```

#### **Attributes Específicos**
```javascript
// ✅ CORRECTO - Solo campos necesarios
const users = await User.findAll({
    attributes: ['id', 'firstName', 'lastName', 'email'],
    include: [{
        model: Role,
        as: 'role',
        attributes: ['key', 'name']
    }]
});

// ❌ PROHIBIDO - Todos los campos
const users = await User.findAll({
    include: [{ model: Role, as: 'role' }]
});
```

#### **Paginación Eficiente**
```javascript
// ✅ CORRECTO - Paginación con offset/limit
const getPaginatedData = async (page = 1, limit = 10) => {
    const offset = (page - 1) * limit;
    
    const { count, rows } = await Model.findAndCountAll({
        limit: parseInt(limit),
        offset: parseInt(offset),
        distinct: true // Para count correcto con includes
    });
    
    return {
        data: rows,
        pagination: {
            page: parseInt(page),
            total: count,
            pages: Math.ceil(count / limit),
            hasNext: offset + limit < count
        }
    };
};
```

## 🗄️ **Database Indexing**

### **Índices Obligatorios**
```javascript
// migrations/add-performance-indexes.cjs
module.exports = {
    async up(queryInterface, Sequelize) {
        // Índices simples para búsquedas frecuentes
        await queryInterface.addIndex('users', ['email']);
        await queryInterface.addIndex('users', ['is_active']);
        await queryInterface.addIndex('users', ['created_at']);
        
        // Índices compuestos para queries complejas
        await queryInterface.addIndex('posts', ['user_id', 'created_at']);
        await queryInterface.addIndex('posts', ['status', 'visibility']);
        
        // Índices únicos para constraints
        await queryInterface.addIndex('user_sessions', ['user_id', 'token'], { unique: true });
    }
};
```

### **Query Analysis**
```javascript
// utils/queryAnalyzer.js
export const analyzeQuery = async (query, replacements = {}) => {
    if (process.env.NODE_ENV === 'development') {
        const startTime = Date.now();
        
        const result = await sequelize.query(query, {
            replacements,
            type: QueryTypes.SELECT,
            logging: (sql, timing) => {
                const duration = Date.now() - startTime;
                if (duration > 1000) { // Queries > 1s
                    console.warn(`SLOW QUERY (${duration}ms):`, sql);
                }
            }
        });
        
        return result;
    }
    
    return await sequelize.query(query, { replacements, type: QueryTypes.SELECT });
};
```

## 🚀 **Caching Strategies**

### **Redis Caching**
```javascript
// utils/cache.js
import Redis from 'ioredis';

const redis = new Redis({
    host: process.env.REDIS_HOST || 'localhost',
    port: process.env.REDIS_PORT || 6379,
    retryDelayOnFailover: 100,
    maxRetriesPerRequest: 3
});

export const cache = {
    async get(key) {
        try {
            const data = await redis.get(key);
            return data ? JSON.parse(data) : null;
        } catch (error) {
            console.error('Cache get error:', error);
            return null;
        }
    },

    async set(key, data, ttl = 3600) {
        try {
            await redis.setex(key, ttl, JSON.stringify(data));
        } catch (error) {
            console.error('Cache set error:', error);
        }
    },

    async del(key) {
        try {
            await redis.del(key);
        } catch (error) {
            console.error('Cache delete error:', error);
        }
    },

    async flush() {
        try {
            await redis.flushall();
        } catch (error) {
            console.error('Cache flush error:', error);
        }
    }
};
```

### **Cache Middleware**
```javascript
// middlewares/cache.js
export const cacheMiddleware = (ttl = 300) => {
    return async (req, res, next) => {
        const key = `cache:${req.method}:${req.originalUrl}`;
        
        try {
            const cached = await cache.get(key);
            if (cached) {
                return res.json(cached);
            }
            
            // Override res.json para cachear respuesta
            const originalJson = res.json;
            res.json = function(data) {
                cache.set(key, data, ttl);
                return originalJson.call(this, data);
            };
            
            next();
        } catch (error) {
            next();
        }
    };
};
```

## 📊 **Memory Management**

### **Memory Monitoring**
```javascript
// utils/memoryMonitor.js
export const monitorMemory = () => {
    const usage = process.memoryUsage();
    
    const metrics = {
        rss: Math.round(usage.rss / 1024 / 1024), // MB
        heapTotal: Math.round(usage.heapTotal / 1024 / 1024),
        heapUsed: Math.round(usage.heapUsed / 1024 / 1024),
        external: Math.round(usage.external / 1024 / 1024),
        timestamp: new Date().toISOString()
    };
    
    // Alertar si uso de memoria es alto
    if (metrics.heapUsed > 500) { // 500MB
        console.warn('High memory usage detected:', metrics);
    }
    
    return metrics;
};

// Monitorear cada 30 segundos
setInterval(monitorMemory, 30000);
```

### **Connection Pool Optimization**
```javascript
// database/connection.js
export const sequelize = new Sequelize(config.database, config.username, config.password, {
    ...config,
    pool: {
        max: process.env.NODE_ENV === 'production' ? 20 : 5,
        min: process.env.NODE_ENV === 'production' ? 5 : 1,
        acquire: 30000, // 30s timeout
        idle: 10000,    // 10s idle timeout
        evict: 1000     // Check every 1s for idle connections
    },
    dialectOptions: {
        connectTimeout: 60000,
        acquireTimeout: 60000,
        timeout: 60000
    }
});
```

## 🔄 **Async/Await Optimization**

### **Batch Processing**
```javascript
// utils/batchProcessor.js
export const processBatch = async (items, processor, batchSize = 10) => {
    const results = [];
    
    for (let i = 0; i < items.length; i += batchSize) {
        const batch = items.slice(i, i + batchSize);
        const batchResults = await Promise.all(
            batch.map(item => processor(item))
        );
        results.push(...batchResults);
        
        // Pequeña pausa entre batches para no sobrecargar
        if (i + batchSize < items.length) {
            await new Promise(resolve => setTimeout(resolve, 10));
        }
    }
    
    return results;
};
```

### **Stream Processing**
```javascript
// utils/streamProcessor.js
import { Transform } from 'stream';

export const createDataTransform = (transformer) => {
    return new Transform({
        objectMode: true,
        transform(chunk, encoding, callback) {
            try {
                const result = transformer(chunk);
                callback(null, result);
            } catch (error) {
                callback(error);
            }
        }
    });
};

// Procesar grandes datasets
export const processLargeDataset = async (query, transformer) => {
    const stream = sequelize.query(query, {
        type: QueryTypes.SELECT,
        raw: true
    });
    
    return new Promise((resolve, reject) => {
        const results = [];
        
        stream
            .pipe(createDataTransform(transformer))
            .on('data', (data) => results.push(data))
            .on('end', () => resolve(results))
            .on('error', reject);
    });
};
```

## 📈 **Performance Monitoring**

### **Response Time Tracking**
```javascript
// middlewares/performanceMonitor.js
export const performanceMonitor = (req, res, next) => {
    const startTime = Date.now();
    
    res.on('finish', () => {
        const duration = Date.now() - startTime;
        
        // Log requests lentos
        if (duration > 1000) {
            console.warn(`SLOW REQUEST (${duration}ms):`, {
                method: req.method,
                url: req.originalUrl,
                status: res.statusCode,
                duration
            });
        }
        
        // Métricas para monitoreo
        if (process.env.NODE_ENV === 'production') {
            // Enviar métricas a servicio de monitoreo
            sendMetrics({
                endpoint: req.originalUrl,
                method: req.method,
                duration,
                status: res.statusCode
            });
        }
    });
    
    next();
};
```

### **Database Performance**
```javascript
// utils/dbPerformance.js
export const trackQueryPerformance = () => {
    const originalQuery = sequelize.query;
    
    sequelize.query = async function(...args) {
        const startTime = Date.now();
        
        try {
            const result = await originalQuery.apply(this, args);
            const duration = Date.now() - startTime;
            
            if (duration > 500) { // Queries > 500ms
                console.warn(`SLOW DB QUERY (${duration}ms):`, args[0]);
            }
            
            return result;
        } catch (error) {
            const duration = Date.now() - startTime;
            console.error(`FAILED DB QUERY (${duration}ms):`, args[0], error.message);
            throw error;
        }
    };
};
```

## 🔧 **Code Optimization**

### **Lazy Loading**
```javascript
// utils/lazyLoader.js
export const lazyLoad = (modulePath) => {
    let module = null;
    
    return async (...args) => {
        if (!module) {
            module = await import(modulePath);
        }
        return module.default(...args);
    };
};

// Uso en services
const heavyProcessor = lazyLoad('./heavyProcessor.js');

export const processData = async (data) => {
    if (data.requiresHeavyProcessing) {
        return await heavyProcessor(data);
    }
    return simpleProcess(data);
};
```

### **Object Pooling**
```javascript
// utils/objectPool.js
export class ObjectPool {
    constructor(createFn, resetFn, maxSize = 10) {
        this.createFn = createFn;
        this.resetFn = resetFn;
        this.maxSize = maxSize;
        this.pool = [];
    }
    
    acquire() {
        if (this.pool.length > 0) {
            return this.pool.pop();
        }
        return this.createFn();
    }
    
    release(obj) {
        if (this.pool.length < this.maxSize) {
            this.resetFn(obj);
            this.pool.push(obj);
        }
    }
}
```

Esta configuración garantiza rendimiento óptimo y escalabilidad del backend.