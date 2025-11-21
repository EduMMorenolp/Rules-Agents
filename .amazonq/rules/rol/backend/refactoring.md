# AccessAI Backend - Refactoring Rules

## 🔧 **Database Manager Pattern Obligatorio**

### Configuración Centralizada
```javascript
// database/manager.js - Patrón implementado
import { Sequelize } from 'sequelize';
import config from './config.js';

class DatabaseManager {
    constructor() {
        const env = process.env.NODE_ENV || 'development';
        const dbConfig = config[env];
        
        this.sequelize = new Sequelize(dbConfig.database, dbConfig.username, dbConfig.password, dbConfig);
        this.models = {};
        this.initializeModels();
        this.setupAssociations();
    }

    initializeModels() {
        // Inicializar todos los modelos
        this.models.User = UserModel(this.sequelize, Sequelize.DataTypes);
        this.models.Alert = AlertModel(this.sequelize);
        this.models.Device = DeviceModel(this.sequelize);
        this.models.Zone = ZoneModel(this.sequelize);
        // ... otros modelos
    }

    setupAssociations() {
        // ✅ TODAS las asociaciones centralizadas aquí
        const { User, Alert, Device, Zone, AccessLog } = this.models;
        
        User.hasMany(Alert, { foreignKey: 'userId', as: 'alerts' });
        Device.hasMany(Alert, { foreignKey: 'deviceId', as: 'alerts' });
        Zone.hasMany(Device, { foreignKey: 'zoneId', as: 'devices' });
        // ... todas las asociaciones
    }
}

export default new DatabaseManager();
```

### Uso en Services
```javascript
// ✅ CORRECTO - Usar manager centralizado
import dbManager from '../../database/manager.js';
const { User, Alert, Device, Zone } = dbManager.models;

// ❌ PROHIBIDO - Inicializar modelos en cada service
const User = UserModel(sequelize, DataTypes);
const Post = PostModel(sequelize);
```

## 📋 **Response Formatters Obligatorios**

### Formatters Centralizados
```javascript
// utils/responseFormatter.js - Patrón implementado
export const formatAlertResponse = (alert, device, user = null) => {
    const reactionCounts = reactions.reduce((acc, reaction) => {
        acc[reaction.type] = (acc[reaction.type] || 0) + 1;
        return acc;
    }, { like: 0, love: 0, laugh: 0, angry: 0, sad: 0 });

    return {
        id: alert.id,
        type: alert.type,
        message: alert.message,
        severity: alert.severity,
        resolved: alert.resolved,
        device: {
            id: device.id,
            name: device.name,
            location: device.location
        },
        user: user ? {
            id: user.id,
            firstName: user.firstName,
            lastName: user.lastName
        } : null,
        createdAt: alert.createdAt
    };
};

export const formatCommentResponse = (comment, user, reactions = [], userReaction = null) => {
    // Estructura similar para comentarios
};

export const formatReplyResponse = (reply, user, reactions = [], userReaction = null) => {
    // Estructura similar para replies
};
```

### Uso en Services
```javascript
// ✅ CORRECTO - Usar formatters
import { formatPostResponse } from '../utils/responseFormatter.js';

const [post, user] = await Promise.all([
    Post.create(data),
    User.findByPk(userId)
]);

return formatPostResponse(post, user);

// ❌ PROHIBIDO - Construcción manual
return {
    id: post.id,
    userId: post.userId,
    // ... construcción repetitiva
};
```

## 🎯 **Service Refactoring Patterns**

### Métodos Concisos Obligatorios
```javascript
// ✅ CORRECTO - Métodos cortos y específicos
class PostService {
    async createPost(userId, { content, mediaFile, visibility = 'public' }) {
        if (!content && !mediaFile) {
            throw new Error('Either content or media file is required');
        }

        let mediaUrl = null;
        if (mediaFile) {
            mediaUrl = await this._uploadMedia(mediaFile, userId);
        }

        const [post, user] = await Promise.all([
            Post.create({ userId, content, mediaUrl, visibility }),
            this._getUserData(userId)
        ]);

        return formatPostResponse(post, user);
    }

    // ✅ Métodos helper privados
    async _uploadMedia(mediaFile, userId) {
        return process.env.NODE_ENV === 'test' 
            ? 'https://test-bucket.s3.amazonaws.com/test-file.jpg'
            : await uploadToS3('posts', mediaFile, `${userId}-${Date.now()}`);
    }

    async _getUserData(userId) {
        return User.findByPk(userId, { 
            attributes: ['id', 'firstName', 'lastName', 'avatarUrl'] 
        });
    }
}
```

### Eliminación de Código Duplicado
```javascript
// ✅ CORRECTO - Reutilizar lógica común
class PostService {
    async _validatePostOwnership(postId, userId) {
        const post = await Post.findByPk(postId);
        if (!post || post.userId !== userId) {
            throw new Error('Post not found or unauthorized');
        }
        return post;
    }

    async updatePost(postId, userId, data) {
        const post = await this._validatePostOwnership(postId, userId);
        // ... lógica de actualización
    }

    async deletePost(postId, userId) {
        const post = await this._validatePostOwnership(postId, userId);
        return await post.update({ isDeleted: true });
    }
}

// ❌ PROHIBIDO - Duplicar validaciones
async updatePost(postId, userId, data) {
    const post = await Post.findByPk(postId);
    if (!post || post.userId !== userId) {
        throw new Error('Post not found or unauthorized');
    }
    // ...
}

async deletePost(postId, userId) {
    const post = await Post.findByPk(postId);
    if (!post || post.userId !== userId) {
        throw new Error('Post not found or unauthorized');
    }
    // ...
}
```

## 🚀 **Performance Optimization**

### Promise.all Obligatorio
```javascript
// ✅ CORRECTO - Operaciones paralelas
const [post, user, reactions] = await Promise.all([
    Post.create(data),
    User.findByPk(userId),
    Reaction.findAll({ where: { postId } })
]);

// ❌ PROHIBIDO - Operaciones secuenciales innecesarias
const post = await Post.create(data);
const user = await User.findByPk(userId);
const reactions = await Reaction.findAll({ where: { postId } });
```

### Queries Optimizadas
```javascript
// ✅ CORRECTO - Incluir solo datos necesarios
const posts = await Post.findAll({
    attributes: ['id', 'content', 'mediaUrl', 'created_at'], // Solo campos necesarios
    include: [{
        model: User,
        as: 'author',
        attributes: ['id', 'firstName', 'lastName', 'avatarUrl'] // Solo campos necesarios
    }],
    where: { isDeleted: false },
    order: [['created_at', 'DESC']]
});

// ❌ PROHIBIDO - Traer todos los campos
const posts = await Post.findAll({
    include: [{ model: User, as: 'author' }] // Trae todos los campos
});
```

## 📊 **Métricas de Refactoring**

### Objetivos Obligatorios
- **Reducir líneas de código en 40-60%**
- **Eliminar duplicación de lógica**
- **Centralizar configuración de BD**
- **Estandarizar formateo de respuestas**
- **Optimizar queries con Promise.all**

### Checklist de Refactoring
- [ ] ✅ Database Manager implementado
- [ ] ✅ Response Formatters creados
- [ ] ✅ Métodos helper privados extraídos
- [ ] ✅ Validaciones comunes centralizadas
- [ ] ✅ Promise.all para operaciones paralelas
- [ ] ✅ Attributes específicos en queries
- [ ] ✅ Imports estáticos (no dinámicos)

## 🔄 **Proceso de Refactoring**

### Pasos Obligatorios
1. **Analizar service actual** - Identificar duplicación
2. **Crear Database Manager** - Si no existe
3. **Extraer Response Formatters** - Para entidades principales
4. **Refactorizar métodos largos** - Dividir en helper methods
5. **Optimizar queries** - Promise.all y attributes específicos
6. **Eliminar imports dinámicos** - Usar imports estáticos
7. **Validar funcionalidad** - Tests deben pasar

### Ejemplo de Refactoring
```javascript
// ANTES - 600+ líneas, lógica duplicada
class PostService {
    async createPost(userId, data) {
        // 50 líneas de lógica mezclada
        const post = await Post.create(data);
        const user = await User.findByPk(userId);
        return {
            id: post.id,
            userId: post.userId,
            // ... 20 líneas de construcción manual
        };
    }
}

// DESPUÉS - 300 líneas, lógica separada
class PostService {
    async createPost(userId, data) {
        // 10 líneas de lógica clara
        const [post, user] = await Promise.all([
            Post.create(data),
            this._getUserData(userId)
        ]);
        
        return formatPostResponse(post, user);
    }
}
```

Esta configuración de refactoring garantiza código más limpio, mantenible y performante siguiendo los estándares AccessAI.