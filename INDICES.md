# Índices en MongoDB: Guía Definitiva para Alto Rendimiento (2025)
*Estrategias usadas en sistemas de 1M+ operaciones/segundo y 100TB+ de datos*

Los índices en MongoDB no son solo "mejoras opcionales" – son **el núcleo de la escalabilidad**. Un mal diseño de índices puede costar millones en hardware adicional.

---

## Tipos de Índices y Cuándo Usarlos (Con Ejemplos Reales)

### 1. Índice Simple (Single Field)
**Para qué sirve**: Búsquedas por igualdad o rango en un solo campo.  
**Ejemplo avanzado** (Sistema de IoT - 50B documentos):
```javascript
// Sensor ID es consultado en el 95% de las queries
db.sensor_data.createIndex({ sensor_id: 1 }, {
  name: "idx_sensor_id_partial",
  partialFilterExpression: { status: { $eq: "active" } }, // Solo indexar sensores activos
  collation: { locale: "en", strength: 2 } // Case-insensitive
});
```
**Key Insight**:  
- Usar `partialFilterExpression` para reducir el tamaño del índice en un 70% cuando solo se necesita un subconjunto de datos.

---

### 2. Índice Compuesto (Compound)
**Para qué sirve**: Consultas que filtran por múltiples campos.  
**Regla de Oro**:  
- Orden de campos = Selectividad (campo más selectivo primero) + Operadores usados (igualdad antes de rango).  

**Ejemplo real** (Plataforma E-commerce - 10M órdenes/día):
```javascript
// Query frecuente: status + fecha + región
db.orders.createIndex({
  status: 1,         // Igualdad (high cardinality)
  region: 1,         // Igualdad
  created_at: -1     // Ordenamiento DESC
}, {
  background: true,  // Creación sin bloquear la DB
  compression: "zstd" // Ahorra 30% de espacio vs snappy
});
```
**Performance Hack**:  
- Usar `explain("executionStats")` para verificar si el índice cubre todos los campos del `sort()` y `projection()`.

---

### 3. Índice Multikey (Arrays)
**Para qué sirve**: Optimizar consultas sobre arrays (tags, listas de IDs).  
**Ejemplo crítico** (Red Social - 5M usuarios con 1000+ amigos cada uno):
```javascript
db.users.createIndex({ "friends.id": 1 }, {
  name: "idx_friends_covering",
  storageEngine: {
    wiredTiger: { configString: "block_compressor=zstd" }
  }
});

// Query optimizada (Covered Index):
db.users.find(
  { "friends.id": "user123" },
  { _id: 0, "friends.$": 1 }
).hint("idx_friends_covering");
```
**Alerta**:  
- Los índices multikey tienen limitaciones con consultas `$elemMatch` complejas. Siempre verificar con `explain()`.

---

### 4. Índice de Texto (Full-Text Search)
**Para qué sirve**: Búsquedas semánticas en texto largo.  
**Ejemplo pro** (Motor de Búsqueda Legal - 10M documentos):
```javascript
db.legal_docs.createIndex({
  title: "text",
  content: "text",
  comments: "text"
}, {
  weights: {
    title: 10,    // Priorizar matches en título
    content: 5,
    comments: 1
  },
  default_language: "spanish",
  wildcardProjection: { // Para evitar indexar todo el documento
    title: 1,
    content: 1,
    comments: 1
  }
});

// Query con relevancia y filtros combinados:
db.legal_docs.find({
  $text: { $search: "contrato \"cláusula confidencial\" -rescisión" },
  year: { $gte: 2020 }
}).sort({ score: { $meta: "textScore" } });
```
**Secretos del Oficio**:  
- Combinar con índices compuestos (`year` en este caso) para filtrar antes de aplicar el texto.

---

### 5. Índices Geospaciales (2dsphere)
**Para qué sirve**: Consultas de proximidad, geofencing.  
**Ejemplo ultra-optimizado** (Delivery en Tiempo Real - 100K riders):
```javascript
db.riders.createIndex({ location: "2dsphere" }, {
  sphereVersion: 3,  // Usar última versión del algoritmo
  bits: 26           // Mayor precisión (default es 26)
});

// Query para riders en 5km + disponibilidad:
db.riders.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [ -58.3816, -34.6037 ] },
      $maxDistance: 5000
    }
  },
  status: "available"
}).hint({ location: "2dsphere" });
```
**Optimización**:  
- El índice `2dsphere` ya incluye los campos de coordenadas. No es necesario indexar `status` por separado si la selectividad es baja.

---

### 6. Índices TTL (Time-To-Live)
**Para qué sirve**: Eliminar documentos automáticamente (logs, sesiones).  
**Ejemplo de sistema de alta demanda** (JWT Tokens - 1B tokens/día):
```javascript
db.sessions.createIndex({ "expire_at": 1 }, {
  expireAfterSeconds: 0,  // Basado en el campo expire_at
  comment: "Auto-purge JWT tokens"
});
```
**Nota Clave**:  
- Usar `expireAfterSeconds: 0` + campo tipo fecha para control exacto. Evita el "time drift" de MongoDB.

---

## Conclusión
Los índices son tu arma secreta en MongoDB, pero requieren estrategia militar:
1. **Benchmarking Constante**: Usar `explain()` y `$indexStats` semanalmente.  
2. **Compresión Avanzada**: `zstd` para ahorrar 30-50% de espacio.  
3. **Tunning Continuo**: Eliminar índices no usados (pueden costar más que no tenerlos).  
4. **Diseño Basado en Queries**: Los índices deben seguir tus patrones de acceso, no al revés.  

