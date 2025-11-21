# AccessAI Backend - Timestamp Rules (OBLIGATORIO)

## 🚨 **REGLA CRÍTICA: SIEMPRE snake_case**

### ✅ **CORRECTO - OBLIGATORIO**
```javascript
// ✅ SIEMPRE usar snake_case en queries
order: [['created_at', 'DESC']]
order: [['updated_at', 'DESC']]

// ✅ SIEMPRE usar snake_case en where
where: { created_at: { [Op.gte]: startDate } }
where: { updated_at: { [Op.lt]: endDate } }

// ✅ SIEMPRE usar snake_case en updates
await Model.update({ updated_at: new Date() }, { where: { id } });
```

### ❌ **PROHIBIDO - NUNCA USAR**
```javascript
// ❌ NUNCA usar camelCase
order: [['createdAt', 'DESC']]   // MAL
order: [['updatedAt', 'DESC']]   // MAL

// ❌ NUNCA usar camelCase en where
where: { createdAt: { [Op.gte]: startDate } }  // MAL
where: { updatedAt: { [Op.lt]: endDate } }     // MAL

// ❌ NUNCA usar camelCase en updates
await Model.update({ updatedAt: new Date() }, { where: { id } }); // MAL
```

## 🔧 **Configuración Obligatoria de Modelos**

### ✅ **Configuración Estándar**
```javascript
const Model = sequelize.define('ModelName', {
    // campos del modelo
}, {
    tableName: 'table_name',
    underscored: true,           // ✅ OBLIGATORIO
    timestamps: true,            // ✅ OBLIGATORIO
    createdAt: 'created_at',     // ✅ OBLIGATORIO
    updatedAt: 'updated_at'      // ✅ OBLIGATORIO
});
```

## 📋 **Ejemplos Comunes**

### Queries de Listado
```javascript
// ✅ CORRECTO
const alerts = await Alert.findAll({
    order: [['created_at', 'DESC']]
});

// ✅ CORRECTO
const devices = await Device.findAll({
    order: [['updated_at', 'DESC']]
});
```

### Filtros por Fecha
```javascript
// ✅ CORRECTO
const recentAlerts = await Alert.findAll({
    where: {
        created_at: {
            [Op.gte]: new Date(Date.now() - 24 * 60 * 60 * 1000)
        }
    }
});
```

### Updates con Timestamp
```javascript
// ✅ CORRECTO
await Alert.update(
    { updated_at: new Date() },
    { where: { id: alertId } }
);
```

## 🚨 **Validación Obligatoria**

### Antes de Commit
1. **Buscar en código**: `createdAt` y `updatedAt`
2. **Reemplazar por**: `created_at` y `updated_at`
3. **Verificar queries**: Todos los `order` deben usar snake_case
4. **Verificar updates**: Todos los updates de timestamp deben usar snake_case

### Comando de Verificación
```bash
# Buscar usos incorrectos
grep -r "createdAt\|updatedAt" src/
grep -r "order.*createdAt\|order.*updatedAt" src/
```

## 📝 **Checklist Pre-Commit**

- [ ] ✅ Todos los `order` usan `created_at` / `updated_at`
- [ ] ✅ Todos los `where` usan `created_at` / `updated_at`  
- [ ] ✅ Todos los `update` usan `created_at` / `updated_at`
- [ ] ✅ Modelos configurados con `underscored: true`
- [ ] ✅ Modelos configurados con `createdAt: 'created_at'`
- [ ] ✅ Modelos configurados con `updatedAt: 'updated_at'`

Esta regla es **OBLIGATORIA** y **NO NEGOCIABLE** para mantener consistencia con la base de datos PostgreSQL que usa snake_case.