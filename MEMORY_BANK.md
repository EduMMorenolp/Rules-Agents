# Memory Bank - Rules-Agents 🧠

## 📋 Resumen del Proyecto

**Rules-Agents** es un conjunto de reglas y mejores prácticas organizadas por lenguajes de programación para asistentes de IA como Amazon Q Developer. Contiene patrones estandarizados y reutilizables para desarrollo de software.

## 🏗️ Arquitectura del Proyecto

### Estructura Principal
```
.amazonq/rules/
├── lenguaje/
│   ├── javascript/          # Node.js, Express, React
│   ├── python/             # Django, Flask, FastAPI  
│   └── java/               # Spring Boot, Maven
├── rol/
│   └── backend/            # Reglas específicas de backend
└── idioma-generic.md       # Reglas de idioma globales
```

### Lenguajes Soportados
- **JavaScript/Node.js** 🟨 - Express, React, Testing
- **Python** 🐍 - Django, Flask, FastAPI
- **Java** ☕ - Spring Boot, JPA, Maven
- **C#** 🔷 - ASP.NET Core, Entity Framework
- **Go** 🐹 - Gin, Echo, GORM
- **PHP** 🐘 - Laravel, Symfony
- **TypeScript** 🔷 - NestJS, Angular

## 🚨 Reglas Críticas Obligatorias

### 1. **Idioma y Comunicación**
```javascript
// ✅ CORRECTO - Comentarios en español
// Validar datos del usuario
const validarUsuario = async (userData) => {
    // Verificar si el usuario ya existe
    if (usuarioExistente) {
        throw new Error('User already exists'); // Errores técnicos en inglés
    }
};

// ❌ PROHIBIDO - Comentarios en inglés
// Validate user data - MAL
```

### 2. **Timestamps Obligatorios (snake_case)**
```javascript
// ✅ CORRECTO - SIEMPRE snake_case
order: [['created_at', 'DESC']]
order: [['updated_at', 'DESC']]

// ❌ PROHIBIDO - NUNCA camelCase
order: [['createdAt', 'DESC']]  // MAL
```

### 3. **CHANGELOG Obligatorio**
```markdown
## [Unreleased]
### Added
- Nueva funcionalidad implementada [DD/MM/YYYY]
### Changed  
- Funcionalidad modificada [DD/MM/YYYY]
### Fixed
- Bugs corregidos [DD/MM/YYYY]
```

## 🎯 Patrones por Tecnología

### **JavaScript/Node.js Backend**

#### ES Modules Obligatorio
```javascript
// ✅ CORRECTO - Siempre con extensión .js
import authService from '../services/authService.js';
import { asyncHandler } from '../utils/errorHandler.js';

// ✅ Export patterns
export default new AuthController();
export { validateEmail, isValidPassword };
```

#### Controller-Service Pattern
```javascript
// Controller - SOLO manejo HTTP
class AuthController {
    register = asyncHandler(async (req, res) => {
        const user = await authService.register(req.body);
        res.status(201).json(user);
    });
}

// Service - TODA la lógica de negocio
class AuthService {
    async register(userData) {
        // Validaciones, operaciones BD, lógica adicional
        return this.formatUserResponse(user);
    }
}
```

#### Database Patterns (Sequelize)
```javascript
// Configuración obligatoria
{
    tableName: 'table_name',
    underscored: true,
    timestamps: true,
    createdAt: 'created_at',
    updatedAt: 'updated_at'
}

// Performance optimization
const [user, posts, reactions] = await Promise.all([
    User.findByPk(userId),
    Post.findAll({ where: { userId } }),
    Reaction.count({ where: { postId } })
]);
```

### **Python Django**

#### Model Definition
```python
class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True

class User(AbstractUser):
    email = models.EmailField(_('email address'), unique=True)
    USERNAME_FIELD = 'email'
    
    class Meta:
        db_table = 'users'
        verbose_name = _('User')
```

#### API Views (DRF)
```python
class ProductListCreateAPIView(generics.ListCreateAPIView):
    queryset = Product.active.all()
    serializer_class = ProductSerializer
    permission_classes = [IsAuthenticated]
    
    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)
```

### **Java Spring Boot**

#### Entity Definition
```java
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
public class User extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    @Email(message = "Email debe tener formato válido")
    private String email;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Product> products;
}
```

#### Service Layer
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserService {
    
    private final UserRepository userRepository;
    
    @Transactional
    public UserResponseDTO createUser(UserCreateDTO createDTO) {
        // Validaciones y lógica de negocio
        User user = userMapper.toEntity(createDTO);
        User savedUser = userRepository.save(user);
        return userMapper.toResponseDTO(savedUser);
    }
}
```

## 🔒 Seguridad Obligatoria

### Authentication Patterns
```javascript
// JWT Configuration
export const TOKEN_CONFIG = {
    ACCESS_TOKEN_EXPIRES_IN: '15m',
    REFRESH_TOKEN_EXPIRES_IN: '7d',
    ACCESS_TOKEN_SECRET: process.env.SECRET_JWT_KEY
};

// Password Security
export const hashPassword = async (password) => {
    return await bcrypt.hash(password, 12); // 12 rounds
};
```

### Input Validation
```javascript
// Sanitización obligatoria
export const sanitizeInput = (input) => {
    const escaped = validator.escape(input);
    const clean = DOMPurify.sanitize(escaped);
    return clean.trim();
};

// UUID Validation
export const isValidUUID = (id) => {
    const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
    return uuidRegex.test(id);
};
```

## 📊 API Design Standards

### HTTP Status Codes
```javascript
// Success
200 - Success (operación exitosa)
201 - Created (recurso creado)
204 - No Content (éxito sin contenido)

// Client Errors
400 - Bad Request (solicitud malformada)
401 - Unauthorized (no autenticado)
403 - Forbidden (no autorizado)
404 - Not Found (recurso no encontrado)
422 - Validation Error (error de validación)

// Server Errors
500 - Internal Server Error
503 - Service Unavailable
```

### Response Formats
```javascript
// Success con paginación
{
    data: items,
    pagination: {
        page: parseInt(page),
        limit: parseInt(limit),
        total: count,
        pages: Math.ceil(count / limit)
    }
}

// Error con detalles
{
    error: 'Validation Error',
    message: 'Invalid input data',
    details: ['Email is required', 'Password too weak']
}
```

## 🗄️ Database Best Practices

### Migration Patterns
```javascript
// Numeración secuencial obligatoria
20250814100001-create-roles.cjs
20250814100002-create-users.cjs
20250814100003-create-resources.cjs

// Estructura estándar
module.exports = {
    async up(queryInterface, Sequelize) {
        await queryInterface.createTable('users', {
            id: { type: Sequelize.UUID, primaryKey: true },
            email: { type: Sequelize.STRING, unique: true },
            created_at: { type: Sequelize.DATE },
            updated_at: { type: Sequelize.DATE }
        });
        
        // Índices para performance
        await queryInterface.addIndex('users', ['email']);
    }
};
```

### Seeders Seguros
```javascript
// ✅ CORRECTO - IDs específicos
async down(queryInterface, Sequelize) {
    await queryInterface.bulkDelete('users', {
        id: ['uuid-1', 'uuid-2', 'uuid-3']
    });
}

// ❌ PROHIBIDO - Eliminar todo
await queryInterface.bulkDelete('users', null, {}); // MAL
```

## 🧪 Testing Strategy

### Pirámide de Testing
```javascript
// 70% Unit Tests - Lógica de negocio
// 20% Integration Tests - APIs y BD  
// 10% E2E Tests - Flujos completos

describe('AuthService', () => {
    it('should create user with valid data', async () => {
        // Arrange
        const userData = { email: 'test@example.com' };
        
        // Act
        const result = await authService.register(userData);
        
        // Assert
        expect(result).toHaveProperty('id');
    });
});
```

## 📈 Performance Optimization

### Database Optimization
```javascript
// Promise.all para operaciones paralelas
const [user, posts, followers] = await Promise.all([
    User.findByPk(userId),
    Post.findAll({ where: { userId } }),
    Follow.count({ where: { followingId: userId } })
]);

// Attributes específicos
const users = await User.findAll({
    attributes: ['id', 'firstName', 'lastName'],
    include: [{ model: Role, attributes: ['key'] }]
});
```

### Caching Strategies
```javascript
// Redis caching
export const cache = {
    async get(key) {
        const data = await redis.get(key);
        return data ? JSON.parse(data) : null;
    },
    
    async set(key, data, ttl = 3600) {
        await redis.setex(key, ttl, JSON.stringify(data));
    }
};
```

## 📊 Monitoring & Observability

### Structured Logging
```javascript
// Winston logger setup
export const logger = winston.createLogger({
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.Console(),
        new winston.transports.File({ filename: 'logs/error.log' })
    ]
});
```

### Metrics Collection
```javascript
class MetricsCollector {
    recordRequest(status, duration) {
        this.metrics.requests.total++;
        this.metrics.requests.byStatus[status]++;
        this.responseTimes.push(duration);
    }
    
    getMetrics() {
        return {
            ...this.metrics,
            timestamp: new Date().toISOString(),
            uptime: process.uptime()
        };
    }
}
```

## 🔄 Refactoring Patterns

### Database Manager Pattern
```javascript
class DatabaseManager {
    constructor() {
        this.sequelize = new Sequelize(config);
        this.models = {};
        this.initializeModels();
        this.setupAssociations();
    }
    
    setupAssociations() {
        const { User, Post, Comment } = this.models;
        User.hasMany(Post, { foreignKey: 'userId' });
        Post.hasMany(Comment, { foreignKey: 'postId' });
    }
}
```

### Response Formatters
```javascript
export const formatUserResponse = (user) => {
    if (user.get) {
        const plainUser = user.get({ plain: true });
        return { id: plainUser.id, email: plainUser.email };
    }
    
    return {
        id: user.id,
        email: user.email || user.email_address
    };
};
```

## 🚀 Deployment & DevOps

### Environment Configuration
```javascript
// Validar variables críticas
const requiredEnvVars = ['JWT_SECRET', 'DB_PASSWORD'];
for (const envVar of requiredEnvVars) {
    if (!process.env[envVar]) {
        throw new Error(`Missing: ${envVar}`);
    }
}
```

### Health Checks
```javascript
router.get('/health', async (req, res) => {
    const healthCheck = {
        status: 'healthy',
        timestamp: new Date().toISOString(),
        services: {}
    };
    
    try {
        await sequelize.authenticate();
        healthCheck.services.database = 'connected';
    } catch (error) {
        healthCheck.services.database = 'disconnected';
        healthCheck.status = 'unhealthy';
    }
    
    const statusCode = healthCheck.status === 'healthy' ? 200 : 503;
    res.status(statusCode).json(healthCheck);
});
```

## 📚 Casos de Uso Específicos

### AccessAI Backend
- **Sistema de alertas** en tiempo real
- **Dispositivos y zonas** de seguridad
- **Visitantes temporales** con QR
- **Logs de acceso** y monitoreo
- **Integración con IA** para reconocimiento

### Microservicios
- **Notificaciones** en tiempo real
- **Chat en vivo** con WebSocket
- **Procesamiento de video** streams
- **Embeddings faciales** para IA

## 🎯 Checklist de Implementación

### Pre-Commit Obligatorio
- [ ] ✅ CHANGELOG.md actualizado
- [ ] ✅ Timestamps en snake_case
- [ ] ✅ Comentarios en español
- [ ] ✅ Response formatters usados
- [ ] ✅ Validaciones implementadas
- [ ] ✅ Tests escritos
- [ ] ✅ Seeders seguros (IDs específicos)
- [ ] ✅ Error handler verificado

### Arquitectura Validada
- [ ] ✅ Controller-Service pattern
- [ ] ✅ Database Manager centralizado
- [ ] ✅ AsyncHandler en controllers
- [ ] ✅ Promise.all para operaciones paralelas
- [ ] ✅ Attributes específicos en queries
- [ ] ✅ Soft delete implementado

## 🔧 Herramientas y Configuración

### Scripts Esenciales
```json
{
    "scripts": {
        "dev": "nodemon src/app/index.js",
        "db:migrate": "sequelize-cli db:migrate",
        "db:seed": "sequelize-cli db:seed:all",
        "test": "vitest",
        "test:coverage": "vitest --coverage"
    }
}
```

### Configuración de Desarrollo
```javascript
// Force sync en desarrollo
const isDev = process.env.NODE_ENV === 'development';
await sequelize.sync({ force: isDev, logging: false });

// Seeders obligatorios después de sync
if (isDev) {
    console.log('🌱 Ejecutar: npm run db:seed');
}
```

---

**Memory Bank actualizado con todos los patrones, reglas y mejores prácticas del proyecto Rules-Agents.**