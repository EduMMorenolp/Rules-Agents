# Rules-Agents 🤖

Conjunto de reglas y mejores prácticas organizadas por lenguajes de programación para asistentes de IA como Amazon Q Developer.

## 📋 Descripción

Este repositorio contiene reglas estandarizadas y reutilizables organizadas por lenguajes de programación, diseñadas para ser utilizadas con asistentes de IA en el desarrollo de software.

## 🏗️ Estructura por Lenguajes

```
.amazonq/rules/
├── javascript/           # Node.js, Express, React
│   ├── api-design-generic.md
│   ├── architecture-generic.md
│   ├── testing-generic.md
│   └── ...
├── python/              # Django, Flask, FastAPI
│   ├── django-patterns.md
│   ├── flask-best-practices.md
│   └── ...
├── java/                # Spring Boot, Maven
│   ├── spring-patterns.md
│   ├── maven-structure.md
│   └── ...
├── csharp/              # .NET, ASP.NET Core
│   ├── dotnet-patterns.md
│   ├── entity-framework.md
│   └── ...
├── go/                  # Gin, Echo, Fiber
│   ├── go-patterns.md
│   ├── gin-structure.md
│   └── ...
├── php/                 # Laravel, Symfony
│   ├── laravel-patterns.md
│   ├── php-standards.md
│   └── ...
└── typescript/          # Angular, NestJS
    ├── nestjs-patterns.md
    ├── angular-structure.md
    └── ...
```

## 🎯 Lenguajes Soportados

### **JavaScript/Node.js** 🟨
- **Frameworks**: Express.js, Fastify, Koa
- **Frontend**: React, Vue, Vanilla JS
- **Testing**: Jest, Vitest, Cypress
- **Herramientas**: ESLint, Prettier, Webpack

### **Python** 🐍
- **Frameworks**: Django, Flask, FastAPI
- **Testing**: pytest, unittest
- **Herramientas**: Black, flake8, mypy
- **Bases de datos**: SQLAlchemy, Django ORM

### **Java** ☕
- **Frameworks**: Spring Boot, Quarkus
- **Testing**: JUnit, TestNG, Mockito
- **Herramientas**: Maven, Gradle
- **Bases de datos**: JPA, Hibernate

### **C#** 🔷
- **Frameworks**: ASP.NET Core, .NET
- **Testing**: xUnit, NUnit, MSTest
- **Herramientas**: NuGet, Entity Framework
- **Patrones**: CQRS, Repository

### **Go** 🐹
- **Frameworks**: Gin, Echo, Fiber
- **Testing**: testing package, Testify
- **Herramientas**: go mod, gofmt
- **Bases de datos**: GORM, sqlx

### **PHP** 🐘
- **Frameworks**: Laravel, Symfony, CodeIgniter
- **Testing**: PHPUnit, Pest
- **Herramientas**: Composer, PHP-CS-Fixer
- **Bases de datos**: Eloquent, Doctrine

### **TypeScript** 🔷
- **Frameworks**: NestJS, Angular, Next.js
- **Testing**: Jest, Vitest
- **Herramientas**: TSC, ESLint
- **Patrones**: Decorators, Dependency Injection

## 🚀 Uso por Lenguaje

### **Para JavaScript/Node.js:**
```bash
# Copiar reglas específicas
cp .amazonq/rules/javascript/* tu-proyecto/.amazonq/rules/
```

### **Para Python:**
```bash
# Copiar reglas específicas
cp .amazonq/rules/python/* tu-proyecto/.amazonq/rules/
```

### **Para cualquier lenguaje:**
1. **Navega** a la carpeta del lenguaje
2. **Copia** las reglas relevantes
3. **Personaliza** según tu proyecto
4. **Configura** tu asistente de IA

## 📋 Reglas Comunes por Lenguaje

### **Todas incluyen:**
- ✅ **Arquitectura** - Patrones y estructura
- ✅ **API Design** - RESTful, GraphQL
- ✅ **Testing** - Unit, Integration, E2E
- ✅ **Security** - Autenticación, autorización
- ✅ **Performance** - Optimización, caching
- ✅ **Monitoring** - Logs, métricas, alertas
- ✅ **Database** - ORM, migraciones, queries
- ✅ **Deployment** - CI/CD, containerización

## 🔧 Frameworks Específicos

### **JavaScript Ecosystem:**
- **Express.js** - API REST tradicional
- **Fastify** - High performance APIs
- **NestJS** - Enterprise applications
- **React** - Frontend applications

### **Python Ecosystem:**
- **Django** - Full-stack framework
- **Flask** - Microframeworks
- **FastAPI** - Modern async APIs
- **SQLAlchemy** - Database toolkit

### **Java Ecosystem:**
- **Spring Boot** - Enterprise applications
- **Quarkus** - Cloud-native apps
- **Maven/Gradle** - Build tools
- **JPA/Hibernate** - ORM solutions

## 📖 Beneficios por Lenguaje

### **Específico y Relevante**
- Reglas adaptadas a cada ecosistema
- Patrones nativos del lenguaje
- Herramientas específicas

### **Mejores Prácticas**
- Convenciones de la comunidad
- Performance optimizations
- Security patterns específicos

### **Ecosistema Completo**
- Frameworks populares
- Testing tools
- Deployment strategies

## 🤝 Contribución

### **Agregar nuevo lenguaje:**
1. Crear carpeta en `.amazonq/rules/[lenguaje]/`
2. Seguir estructura estándar
3. Incluir reglas básicas
4. Actualizar README

### **Mejorar lenguaje existente:**
1. Fork el repositorio
2. Editar reglas específicas
3. Actualizar CHANGELOG
4. Enviar Pull Request

## 📄 Licencia

MIT License - Libre para uso comercial y personal.

---

**Reglas optimizadas por lenguaje para desarrollo con asistentes de IA.**