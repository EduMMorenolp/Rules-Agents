# Backend - Database Rules

## 🗄️ Sequelize Patterns

### Model Definition - OBLIGATORIO snake_case timestamps
```javascript
// models/User.js
import { DataTypes } from 'sequelize';

const UserModel = (sequelize) => {
    const User = sequelize.define('User', {
        id: {
            type: DataTypes.UUID,
            defaultValue: DataTypes.UUIDV4,
            primaryKey: true
        },
        email: {
            type: DataTypes.STRING,
            allowNull: false,
            unique: true,
            validate: { 
                isEmail: true,
                len: [1, 255]
            }
        },
        passwordHash: {
            type: DataTypes.STRING,
            allowNull: false
        },
        firstName: {
            type: DataTypes.STRING,
            allowNull: true,
            validate: { len: [1, 100] }
        },
        isDeleted: {
            type: DataTypes.BOOLEAN,
            defaultValue: false
        },
        isActive: {
            type: DataTypes.BOOLEAN,
            defaultValue: true
        }
    }, {
        // Soft delete por defecto
        defaultScope: {
            where: { isDeleted: false }
        },
        // Scope para incluir password (solo cuando necesario)
        scopes: {
            withPassword: {
                attributes: { include: ['passwordHash'] }
            },
            includeDeleted: {
                where: {} // Sin filtro de isDeleted
            }
        },
        // Hooks para password hashing
        hooks: {
            beforeCreate: async (user) => {
                if (user.passwordHash) {
                    user.passwordHash = await bcrypt.hash(user.passwordHash, 12);
                }
            },
            beforeUpdate: async (user) => {
                if (user.changed('passwordHash')) {
                    user.passwordHash = await bcrypt.hash(user.passwordHash, 12);
                }
            }
        },
        // ✅ CONFIGURACIÓN OBLIGATORIA - snake_case timestamps
        tableName: 'users',
        underscored: true, // snake_case en BD
        timestamps: true,
        createdAt: 'created_at',  // ✅ OBLIGATORIO
        updatedAt: 'updated_at'   // ✅ OBLIGATORIO
    });

    return User;
};

export default UserModel;
```

### Model Associations
```javascript
// services/authService.js - Definir asociaciones en services
import { sequelize } from '../../database/connection.js';
import UserModel from '../../database/models/user.js';
import RoleModel from '../../database/models/role.js';
import ResourceModel from '../../database/models/resource.js';

// Inicializar modelos
const User = UserModel(sequelize, sequelize.Sequelize.DataTypes);
const Role = RoleModel(sequelize, sequelize.Sequelize.DataTypes);
const Resource = ResourceModel(sequelize);

// ✅ Definir asociaciones en services (no en modelos)
User.belongsTo(Role, { foreignKey: 'roleId', as: 'role' });
Role.hasMany(User, { foreignKey: 'roleId', as: 'users' });
User.hasMany(Resource, { foreignKey: 'userId', as: 'resources' });
Resource.belongsTo(User, { foreignKey: 'userId', as: 'user' });
```

## 🔢 **REGLA OBLIGATORIA: Numeración de Migraciones**

### ✅ **Verificar Numeración Antes de Crear**
```bash
# SIEMPRE verificar migraciones existentes antes de crear nueva
ls src/database/migrations/ | sort

# Ejemplo de numeración secuencial:
# 20250814100001-create-roles.cjs
# 20250814100002-create-users.cjs  
# 20250814100003-create-resources.cjs
# 20250814100004-create-categories.cjs
# 20250814100005-create-logs.cjs  ← SIGUIENTE NÚMERO
```

### ✅ **Patrón de Numeración Obligatorio**
```javascript
// FORMATO: YYYYMMDDHHMMSS-descripcion.cjs
// EJEMPLO: 20250814100005-create-resources.cjs

// ✅ CORRECTO - Continuar numeración secuencial
20250814100001-create-roles.cjs
20250814100002-create-users.cjs
20250814100003-create-resources.cjs
20250814100004-create-categories.cjs
20250814100005-create-logs.cjs

// ❌ PROHIBIDO - Saltar números o duplicar
20250814100002-create-users.cjs
20250814100002-create-posts.cjs  // ❌ DUPLICADO
20250814100005-create-comments.cjs // ❌ SALTO DE NÚMERO
```

### ✅ **Proceso Obligatorio**
1. **Listar migraciones existentes**: `ls src/database/migrations/ | sort`
2. **Identificar último número**: Buscar el timestamp más alto
3. **Incrementar secuencialmente**: Siguiente número en la secuencia
4. **Crear migración**: Con el número correcto
5. **Verificar orden**: Confirmar que está al final de la lista

## 🔄 Migration Patterns

### Migration Structure (.cjs files)
```javascript
// migrations/20250814100002-create-users.cjs
module.exports = {
    async up(queryInterface, Sequelize) {
        await queryInterface.createTable('users', {
            id: {
                type: Sequelize.UUID,
                defaultValue: Sequelize.UUIDV4,
                primaryKey: true
            },
            email: {
                type: Sequelize.STRING,
                allowNull: false,
                unique: true
            },
            password_hash: {
                type: Sequelize.STRING,
                allowNull: false
            },
            first_name: {
                type: Sequelize.STRING,
                allowNull: true
            },
            last_name: {
                type: Sequelize.STRING,
                allowNull: true
            },
            role_id: {
                type: Sequelize.INTEGER,
                allowNull: false,
                references: {
                    model: 'roles',
                    key: 'id'
                },
                onUpdate: 'CASCADE',
                onDelete: 'RESTRICT'
            },
            is_active: {
                type: Sequelize.BOOLEAN,
                defaultValue: true
            },
            is_deleted: {
                type: Sequelize.BOOLEAN,
                defaultValue: false
            },
            created_at: {
                type: Sequelize.DATE,
                defaultValue: Sequelize.literal('CURRENT_TIMESTAMP')
            },
            updated_at: {
                type: Sequelize.DATE,
                defaultValue: Sequelize.literal('CURRENT_TIMESTAMP')
            }
        });

        // Agregar índices para performance
        await queryInterface.addIndex('users', ['email']);
        await queryInterface.addIndex('users', ['role_id']);
        await queryInterface.addIndex('users', ['is_deleted']);
    },

    async down(queryInterface, Sequelize) {
        await queryInterface.dropTable('users');
    }
};
```

### Index Optimization
```javascript
// Índices compuestos para queries frecuentes
await queryInterface.addIndex('resources', ['user_id', 'created_at']);
await queryInterface.addIndex('user_resources', ['resource_id', 'user_id'], { unique: true });
await queryInterface.addIndex('refresh_tokens', ['jti'], { unique: true });
await queryInterface.addIndex('refresh_tokens', ['user_id', 'is_revoked']);
```

## 🎯 Query Patterns

### Optimized Queries - OBLIGATORIO snake_case timestamps
```javascript
// ✅ Query con includes específicos y paginación
const getResources = async (page = 1, limit = 10) => {
    const offset = (page - 1) * limit;
    
    const { count, rows } = await Resource.findAndCountAll({
        where: { 
            status: 'active',
            isDeleted: false 
        },
        include: [
            {
                model: User,
                as: 'author',
                attributes: ['id', 'firstName', 'lastName', 'avatarUrl'], // Solo campos necesarios
                include: [{
                    model: Role,
                    as: 'role',
                    attributes: ['key']
                }]
            }
        ],
        attributes: { 
            exclude: ['isDeleted'] // No exponer campos internos
        },
        limit: parseInt(limit),
        offset: parseInt(offset),
        order: [['created_at', 'DESC']], // ✅ OBLIGATORIO snake_case
        distinct: true // Para count correcto con includes
    });

    return {
        data: rows,
        pagination: {
            page: parseInt(page),
            limit: parseInt(limit),
            total: count,
            pages: Math.ceil(count / limit)
        }
    };
};

// ❌ PROHIBIDO - NUNCA usar camelCase en order
order: [['createdAt', 'DESC']]  // ❌ MAL
order: [['updatedAt', 'DESC']]  // ❌ MAL

// ✅ CORRECTO - SIEMPRE usar snake_case
order: [['created_at', 'DESC']] // ✅ BIEN
order: [['updated_at', 'DESC']] // ✅ BIEN
```

### Soft Delete Queries
```javascript
// ✅ Usar scopes para soft delete
const activeUsers = await User.findAll(); // Default scope excluye deleted

const allUsers = await User.scope('includeDeleted').findAll(); // Incluir deleted

const deletedUsers = await User.scope('includeDeleted').findAll({
    where: { isDeleted: true }
});

// ✅ Soft delete operation
await user.update({ isDeleted: true });

// ❌ NUNCA hard delete
await user.destroy(); // NO hacer esto
```

## 🔐 UUID Validation

### UUID Pattern Validation
```javascript
// validators/baseValidator.js
export const isValidUUID = (id) => {
    const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
    return uuidRegex.test(id);
};

// Usar en controllers antes de queries
export const validateUUIDParam = (req, res, next) => {
    const { id } = req.params;
    
    if (!isValidUUID(id)) {
        return res.status(400).json({
            error: 'Bad Request',
            message: 'Invalid ID format'
        });
    }
    
    next();
};
```

## 📊 Seeders Pattern

### Seeder Structure
```javascript
// seeders/20250814100002-demo-users.cjs
const bcrypt = require('bcrypt');

module.exports = {
    async up(queryInterface, Sequelize) {
        // Obtener roles
        const roles = await queryInterface.sequelize.query(
            'SELECT id, key FROM roles;',
            { type: Sequelize.QueryTypes.SELECT }
        );

        const adminRole = roles.find(r => r.key === 'admin');
        const userRole = roles.find(r => r.key === 'user');

        const users = [
            {
                id: '550e8400-e29b-41d4-a716-446655440001',
                email: 'admin@example.com',
                password_hash: await bcrypt.hash('Password123!', 12),
                first_name: 'Admin',
                last_name: 'User',
                role_id: adminRole.id,
                is_active: true,
                created_at: new Date(),
                updated_at: new Date()
            },
            {
                id: '550e8400-e29b-41d4-a716-446655440002',
                email: 'user@example.com',
                password_hash: await bcrypt.hash('Password123!', 12),
                first_name: 'Regular',
                last_name: 'User',
                role_id: userRole.id,
                is_active: true,
                created_at: new Date(),
                updated_at: new Date()
            }
        ];

        await queryInterface.bulkInsert('users', users);
    },

    async down(queryInterface, Sequelize) {
        await queryInterface.bulkDelete('users', null, {});
    }
};
```

## 🔄 Transaction Patterns

### Transaction Usage
```javascript
// services/authService.js
async register(userData) {
    const transaction = await sequelize.transaction();
    
    try {
        // Crear usuario
        const user = await User.create({
            email: userData.email.toLowerCase().trim(),
            passwordHash: userData.password,
            roleId: role.id
        }, { transaction });

        // Crear perfil adicional si es necesario
        if (userData.profileData) {
            await UserProfile.create({
                userId: user.id,
                ...userData.profileData
            }, { transaction });
        }

        // Crear token de verificación
        await UserToken.create({
            userId: user.id,
            tokenHash: hashedToken,
            type: 'email_verification'
        }, { transaction });

        await transaction.commit();
        return user;
    } catch (error) {
        await transaction.rollback();
        throw error;
    }
}
```

## 📈 Performance Optimization

### Connection Pool
```javascript
// database/connection.js
import { Sequelize } from 'sequelize';

export const sequelize = new Sequelize(
    process.env.DB_NAME,
    process.env.DB_USER,
    process.env.DB_PASSWORD,
    {
        host: process.env.DB_HOST,
        port: process.env.DB_PORT,
        dialect: 'postgres',
        logging: process.env.NODE_ENV === 'development' ? console.log : false,
        pool: {
            max: 20,        // Máximo 20 conexiones
            min: 5,         // Mínimo 5 conexiones
            acquire: 30000, // 30s timeout para obtener conexión
            idle: 10000     // 10s antes de cerrar conexión idle
        },
        define: {
            underscored: true,    // snake_case en BD
            freezeTableName: true, // No pluralizar nombres
            timestamps: true
        }
    }
);
```

### Database Sync Configuration
```javascript
// app/index.js - Configuración de sincronización
const isDev = process.env.NODE_ENV === 'development';

await databaseManager.sequelize.sync({
    force: isDev, // true en desarrollo, false en producción
    logging: false
});

// ✅ DESARROLLO: force = true (recrea tablas)
// ✅ PRODUCCIÓN: force = false (preserva datos)
```

## 🔄 **Implicaciones del Force Sync**

### **Desarrollo (force: true)**
- ✅ Tablas se recrean en cada inicio
- ✅ Cambios de modelo se aplican automáticamente
- ✅ No necesitas migraciones para desarrollo
- ⚠️ Datos se pierden en cada reinicio
- ⚠️ Usar seeders para datos de prueba

### **Producción (force: false)**
- ✅ Datos se preservan
- ✅ Tablas existentes no se modifican
- ❌ Cambios de modelo requieren migraciones
- ✅ Seguro para datos reales

### Query Optimization
```javascript
// ✅ Limitar campos retornados
const users = await User.findAll({
    attributes: ['id', 'firstName', 'lastName', 'email'], // Solo campos necesarios
    include: [{
        model: Role,
        as: 'role',
        attributes: ['key', 'name'] // Solo campos necesarios
    }]
});

// ✅ Usar raw queries para operaciones complejas
const stats = await sequelize.query(`
    SELECT 
        COUNT(*) as total_resources,
        COUNT(CASE WHEN created_at > NOW() - INTERVAL '7 days' THEN 1 END) as recent_resources
    FROM resources 
    WHERE is_deleted = false
`, { type: QueryTypes.SELECT });
```

## 🚨 **REGLA CRÍTICA: Seeders Obligatorios en Desarrollo**

### **Con force: true, los seeders son ESENCIALES**
```bash
# Después de cada inicio en desarrollo
npm run db:seed
```

### **Workflow de Desarrollo**
1. **Modificar modelo** → Cambios se aplican automáticamente
2. **Reiniciar servidor** → Tablas se recrean
3. **Ejecutar seeders** → Poblar datos de prueba
4. **Continuar desarrollo** → Con datos consistentes

### **Ventajas del Force Sync**
- ✅ No hay conflictos de migraciones en desarrollo
- ✅ Cambios de modelo instantáneos
- ✅ Base de datos siempre limpia
- ✅ Estructura consistente entre desarrolladores

Esta configuración garantiza desarrollo ágil y producción segura del sistema backend.