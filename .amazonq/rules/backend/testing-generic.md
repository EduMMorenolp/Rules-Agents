# Backend - Testing Rules

## 🧪 **REGLA OBLIGATORIA: Testing Strategy**

### ✅ **Pirámide de Testing**
```javascript
// 70% Unit Tests - Lógica de negocio
// 20% Integration Tests - APIs y BD
// 10% E2E Tests - Flujos completos
```

### **Estructura de Tests**
```
tests/
├── unit/
│   ├── services/
│   ├── utils/
│   └── validators/
├── integration/
│   ├── controllers/
│   ├── routes/
│   └── database/
└── e2e/
    ├── auth.test.js
    └── resources.test.js
```

## 🔧 **Unit Tests - Services**

### **Test Pattern Obligatorio**
```javascript
// tests/unit/services/authService.test.js
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import authService from '../../../src/services/authService.js';

describe('AuthService', () => {
    beforeEach(() => {
        // Setup antes de cada test
    });

    afterEach(() => {
        // Cleanup después de cada test
    });

    describe('register', () => {
        it('should create user with valid data', async () => {
            // Arrange
            const userData = {
                email: 'test@example.com',
                password: 'Password123!',
                firstName: 'Test'
            };

            // Act
            const result = await authService.register(userData);

            // Assert
            expect(result).toHaveProperty('id');
            expect(result.email).toBe('test@example.com');
        });

        it('should throw error for duplicate email', async () => {
            // Arrange
            const userData = { email: 'existing@example.com' };

            // Act & Assert
            await expect(authService.register(userData))
                .rejects.toThrow('Email already exists');
        });
    });
});
```

## 🌐 **Integration Tests - Controllers**

### **API Testing Pattern**
```javascript
// tests/integration/controllers/authController.test.js
import request from 'supertest';
import app from '../../../src/app.js';

describe('Auth Controller', () => {
    describe('POST /auth/register', () => {
        it('should register user successfully', async () => {
            const response = await request(app)
                .post('/api/v1/auth/register')
                .send({
                    email: 'test@example.com',
                    password: 'Password123!',
                    firstName: 'Test'
                });

            expect(response.status).toBe(201);
            expect(response.body).toHaveProperty('id');
        });

        it('should return 422 for invalid data', async () => {
            const response = await request(app)
                .post('/api/v1/auth/register')
                .send({ email: 'invalid-email' });

            expect(response.status).toBe(422);
            expect(response.body.error).toBe('Validation Error');
        });
    });
});
```

## 🗄️ **Database Testing**

### **Test Database Setup**
```javascript
// tests/setup/database.js
import { sequelize } from '../../src/database/connection.js';

export const setupTestDB = async () => {
    await sequelize.sync({ force: true });
    // Ejecutar seeders de test
};

export const cleanupTestDB = async () => {
    await sequelize.drop();
    await sequelize.close();
};
```

### **Mocking Database**
```javascript
// tests/mocks/database.js
export const mockUser = {
    id: '123e4567-e89b-12d3-a456-426614174000',
    email: 'test@example.com',
    firstName: 'Test',
    createdAt: new Date()
};

export const mockSequelize = {
    findOne: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    destroy: vi.fn()
};
```

## 📊 **Coverage Requirements**

### **Minimum Coverage**
```javascript
// vitest.config.js
export default {
    test: {
        coverage: {
            provider: 'v8',
            reporter: ['text', 'json', 'html'],
            thresholds: {
                global: {
                    branches: 80,
                    functions: 80,
                    lines: 80,
                    statements: 80
                }
            }
        }
    }
};
```

## 🚀 **Test Scripts**

### **Package.json Scripts**
```json
{
    "scripts": {
        "test": "vitest",
        "test:unit": "vitest tests/unit",
        "test:integration": "vitest tests/integration",
        "test:e2e": "vitest tests/e2e",
        "test:coverage": "vitest --coverage",
        "test:watch": "vitest --watch"
    }
}
```

## 🔒 **Security Testing**

### **Input Validation Tests**
```javascript
describe('Input Validation', () => {
    it('should prevent SQL injection', async () => {
        const maliciousInput = "'; DROP TABLE users; --";
        
        const response = await request(app)
            .post('/api/v1/users')
            .send({ name: maliciousInput });

        expect(response.status).toBe(422);
    });

    it('should sanitize XSS attempts', async () => {
        const xssInput = '<script>alert("xss")</script>';
        
        const response = await request(app)
            .post('/api/v1/users')
            .send({ bio: xssInput });

        expect(response.body.bio).not.toContain('<script>');
    });
});
```

Esta configuración garantiza calidad y confiabilidad del código backend.