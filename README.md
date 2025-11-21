# Rules-Agents 🤖

Conjunto de reglas y mejores prácticas genéricas para desarrollo backend con Amazon Q Developer y otros asistentes de IA.

## 📋 Descripción

Este repositorio contiene reglas estandarizadas y reutilizables para proyectos backend Node.js/Express, diseñadas para ser utilizadas con asistentes de IA como Amazon Q Developer.

## 🏗️ Estructura

```
.amazonq/rules/backend/
├── api-design-generic.md      # Diseño de APIs RESTful
├── architecture-generic.md    # Patrones de arquitectura
├── changelog-generic.md       # Gestión de changelog
├── database-generic.md        # Patrones de base de datos
├── testing-generic.md         # Estrategias de testing
├── security-generic.md        # Seguridad y protección
├── performance-generic.md     # Optimización de rendimiento
├── monitoring-generic.md      # Observabilidad y monitoreo
└── idioma-generic.md         # Reglas de idioma
```

## 🎯 Características

### **Reglas de API Design**
- Códigos de estado HTTP estándar
- Formatos de respuesta consistentes
- Paginación y filtrado
- Versionado de APIs

### **Arquitectura Backend**
- Patrón Controller-Service
- ES Modules obligatorio
- Manejo de errores centralizado
- Timestamps en snake_case

### **Base de Datos**
- Patrones Sequelize optimizados
- Migraciones numeradas secuencialmente
- Soft delete por defecto
- Índices para performance

### **Testing**
- Pirámide de testing (70% unit, 20% integration, 10% e2e)
- Coverage mínimo 80%
- Security testing integrado

### **Seguridad**
- Headers de seguridad obligatorios
- Sanitización automática de inputs
- Prevención de SQL injection
- Encriptación de datos sensibles

### **Performance**
- Optimización de queries con Promise.all
- Caching con Redis
- Memory management
- Stream processing

### **Monitoreo**
- Logging estructurado
- Métricas en tiempo real
- Sistema de alertas
- Dashboard de observabilidad

## 🚀 Uso

1. **Copia las reglas** a tu proyecto en `.amazonq/rules/`
2. **Personaliza** según las necesidades específicas
3. **Configura** tu asistente de IA para usar estas reglas
4. **Desarrolla** siguiendo los patrones establecidos

## 📝 Idioma

- **Código**: Inglés (estándar internacional)
- **Comentarios**: Español
- **Documentación**: Español
- **Mensajes de error API**: Inglés
- **Validaciones UI**: Español

## 🔧 Tecnologías Soportadas

- **Backend**: Node.js, Express.js
- **Base de Datos**: PostgreSQL, Sequelize ORM
- **Testing**: Vitest, Jest, Supertest
- **Caching**: Redis
- **Logging**: Winston
- **Seguridad**: Helmet, bcrypt, JWT

## 📖 Mejores Prácticas Incluidas

- ✅ Desarrollo ágil con force sync
- ✅ Seeders obligatorios en desarrollo
- ✅ Changelog automático
- ✅ Response formatters centralizados
- ✅ Validación de entrada robusta
- ✅ Rate limiting configurado
- ✅ Health checks implementados

## 🤝 Contribución

1. Fork el repositorio
2. Crea una rama para tu feature
3. Actualiza el CHANGELOG.md
4. Envía un Pull Request

## 📄 Licencia

MIT License - Libre para uso comercial y personal.

---

**Desarrollado para optimizar el trabajo con asistentes de IA en proyectos backend enterprise-ready.**