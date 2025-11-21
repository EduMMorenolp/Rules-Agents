# AccessAI Backend - Seeders Rules

## 🌱 **REGLA OBLIGATORIA: Seeders Seguros y Esenciales**

### 🚨 **CRITICAL: Con force: true, seeders son OBLIGATORIOS**
```javascript
// Con force: true en desarrollo, las tablas se recrean en cada inicio
// Los seeders son la Única forma de tener datos de prueba consistentes
```

### ✅ **SIEMPRE eliminar solo registros específicos**
```javascript
// ✅ CORRECTO - Eliminar solo registros creados por el seeder
async down(queryInterface, Sequelize) {
    await queryInterface.bulkDelete('table_name', {
        id: [
            'specific-uuid-1',
            'specific-uuid-2',
            'specific-uuid-3'
        ]
    }, {});
}

// ❌ PROHIBIDO - NUNCA eliminar todos los registros
async down(queryInterface, Sequelize) {
    await queryInterface.bulkDelete('table_name', null, {}); // MAL
    await queryInterface.bulkDelete('table_name', {}, {}); // MAL
}
```

### 📋 **Estructura Obligatoria de Seeders**
```javascript
// seeders/YYYYMMDDHHMMSS-demo-[entity].cjs
module.exports = {
    async up(queryInterface, Sequelize) {
        await queryInterface.bulkInsert('table_name', [
            {
                id: 'uuid-1', // ID específico
                // ... otros campos
                created_at: new Date(),
                updated_at: new Date()
            },
            {
                id: 'uuid-2', // ID específico
                // ... otros campos
                created_at: new Date(),
                updated_at: new Date()
            }
        ], {});
    },

    async down(queryInterface, Sequelize) {
        // ✅ OBLIGATORIO: Eliminar solo por IDs específicos
        await queryInterface.bulkDelete('table_name', {
            id: ['uuid-1', 'uuid-2']
        }, {});
    }
};
```

## 🤖 **REGLA AUTOMÁTICA: Pregunta de Seeders**

### **Cuándo Preguntar Automáticamente**
- Cuando se implemente un **nuevo modelo/tabla**
- Cuando se implemente un **nuevo endpoint CRUD**
- Cuando se implemente una **nueva funcionalidad** que requiera datos de prueba
- Cuando se modifique la **estructura de datos**

### **Mensaje Automático Obligatorio**
```
🤖 "He implementado [funcionalidad/modelo/endpoint]: [nombre]"
🤖 "¿Quieres que implemente seeders de demostración para esta funcionalidad? (s/n)"

- Si "s" o "S" o "si" o "yes": Implementar seeders
- Si "n" o "N" o "no": Mostrar recordatorio
- Si no responde: Preguntar nuevamente
```

### **Recordatorio si Rechaza**
```
⚠️  RECORDATORIO: Seeders pendientes para [funcionalidad]
📝 Los seeders son importantes para demostración y desarrollo
🚀 Usa: npm run db:seed para poblar datos de prueba
```

## 📊 **Datos de Demostración Estándar**

### **Usuarios de Prueba**
- **Administradores**: 1-2 admin del sistema
- **Oficiales de Seguridad**: 2-3 security officers
- **Empleados**: 3-5 empleados regulares
- **Visitantes**: 2-3 visitantes temporales

### **Datos del Sistema**
- **Zonas**: Diferentes áreas (oficinas, almacén, producción)
- **Dispositivos**: Cámaras y sensores por zona
- **Alertas**: Histórico de alertas resueltas y pendientes
- **Logs de Acceso**: Registros de entrada/salida

### **Configuraciones**
- **Reglas de Acceso**: Horarios y permisos por zona
- **Embeddings Faciales**: Datos de reconocimiento
- **Modelos IA**: Configuración de modelos activos

## 🔢 **Numeración de Seeders**

### **Patrón Obligatorio**
```bash
# Verificar seeders existentes
ls src/database/seeders/ | sort

# Formato: YYYYMMDDHHMMSS-demo-[entity].cjs
20250814100001-demo-roles.cjs
20250814100002-demo-users.cjs
20250814100003-demo-zones.cjs
20250814100004-demo-devices.cjs
20250814100005-demo-alerts.cjs
20250814100006-demo-access-logs.cjs
20250814100007-demo-face-embeddings.cjs
20250814100008-demo-temp-visitors.cjs
20250814100009-demo-access-rules.cjs
20250814100010-demo-ai-models.cjs
```

### **Orden de Dependencias**
1. **Roles** (base del sistema)
2. **Users** (depende de roles)
3. **Zones** (independiente)
4. **Devices** (depende de zones)
5. **Alerts** (depende de devices y users)
6. **Access Logs** (depende de users y devices)
7. **Face Embeddings** (depende de users)
8. **Temp Visitors** (independiente)
9. **Access Rules** (depende de zones)
10. **AI Models** (independiente)

## 🚀 **Comandos de Seeders**

### **Scripts Disponibles**
```json
{
  "db:seed": "sequelize-cli db:seed:all",
  "db:seed:undo": "sequelize-cli db:seed:undo:all",
  "db:seed:specific": "sequelize-cli db:seed --seed",
  "db:seed:undo:specific": "sequelize-cli db:seed:undo --seed"
}
```

### **Uso Seguro**
```bash
# Poblar todos los seeders
npm run db:seed

# Rollback seguro (solo elimina datos de seeders)
npm run db:seed:undo

# Seeder específico
npm run db:seed:specific 20250814100001-demo-roles.cjs

# Rollback específico
npm run db:seed:undo:specific 20250814100001-demo-roles.cjs
```

## 📝 **Validación Pre-Commit**

### **Checklist Obligatorio**
- [ ] ✅ Seeder tiene IDs específicos en `up`
- [ ] ✅ Seeder elimina solo IDs específicos en `down`
- [ ] ✅ No usa `null` o `{}` en `bulkDelete`
- [ ] ✅ Respeta orden de dependencias
- [ ] ✅ Datos realistas y variados
- [ ] ✅ Timestamps correctos

### **Comando de Verificación**
```bash
# Buscar seeders peligrosos
grep -r "bulkDelete.*null\|bulkDelete.*{}" src/database/seeders/
```

## 🔄 **Workflow con Force Sync**

### **Desarrollo Diario**
```bash
# 1. Iniciar servidor (tablas se recrean automáticamente)
npm run dev

# 2. Poblar datos de prueba (OBLIGATORIO)
npm run db:seed

# 3. Continuar desarrollo con datos consistentes
```

### **Modificación de Modelos**
```bash
# 1. Modificar modelo en src/database/models/
# 2. Reiniciar servidor (cambios se aplican automáticamente)
# 3. Ejecutar seeders (datos de prueba actualizados)
npm run db:seed
```

### **Ventajas del Sistema**
- ✅ **Sin migraciones** en desarrollo
- ✅ **Cambios instantáneos** de estructura
- ✅ **Datos consistentes** con seeders
- ✅ **Sin conflictos** entre desarrolladores
- ✅ **Base limpia** en cada inicio

Esta regla garantiza desarrollo eficiente con datos seguros y consistentes para AccessAI.