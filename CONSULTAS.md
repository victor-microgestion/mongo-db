# Métodos de MongoDB: Técnicas para Sistemas Críticos (2025)
*Dominando find(), update(), aggregate() y métodos no documentados usados en Fortune 500*

---

## 🔥 Método `find()`: Más Allá del Básico

### Técnica 1: Proyecciones de Alto Rendimiento
```javascript
// Sistema de Analytics - 100M documentos
db.logs.find(
  { user_id: "u123", timestamp: { $gte: ISODate("2025-01-01") } },
  {
    _id: 0,
    session_data: 1,
    "metrics.$": 1 // Projection positional para subdocumentos
  }
).maxTimeMS(50) // Timeout agresivo para SLA estrictos
  .readConcern("majority") // Consistencia fuerte
  .hint("idx_user_timestamp") // Forzar índice específico
```

**Key Insight**:  
- Usar `{_id: 0}` mejora rendimiento en 15-20% al evitar deserializar ObjectId.

---

### Técnica 2: Cursors de Batched Iteration
```javascript
// Exportación masiva - 1B documentos
const cursor = db.products.find({ category: "electronics" })
  .batchSize(1000) // Optimizar transferencia de red
  .noCursorTimeout(); // Evitar desconexiones en largas operaciones

let count = 0;
while (cursor.hasNext()) {
  const doc = cursor.next();
  // Procesamiento streamming
  if (++count % 10000 === 0) {
    print(`Procesados ${count} docs - Memoria: ${Math.round(process.memoryUsage().heapUsed / 1024 / 1024)}MB`);
  }
}
```

**Alerta**: Siempre cerrar cursors con `cursor.close()` para evitar memory leaks.

---

## 🚀 Método `insertMany()`: Inserción Ultra-Rápida

### Técnica de Bulk Unordered + Compression
```javascript
// IoT Platform - 500k docs/segundo
const ops = [];
for (let i = 0; i < 100000; i++) {
  ops.push({
    insertOne: {
      document: {
        sensor_id: i,
        value: Math.random() * 100,
        timestamp: new Date()
      }
    }
  });
}

db.sensordata.bulkWrite(ops, {
  ordered: false, // Paraleliza inserts
  writeConcern: { w: 0 }, // Fire-and-forget (riesgo de pérdida)
  bypassDocumentValidation: true // Si los docs ya están validados
});
```

**Estadísticas Clave**:  
- `ordered:false` → 300% más rápido que inserts individuales.
- `w:0` → 0.2ms vs 5ms de latency (pero peligroso).

---

## 💎 Método `aggregate()`: Pipeline de Alta Eficiencia

### Pipeline de 10M docs/seg (Sistema de Fraude)
```javascript
db.transactions.aggregate([
  { $match: {
      amount: { $gt: 10000 },
      currency: "USD"
  }},
  { $lookup: {
      from: "users",
      localField: "user_id",
      foreignField: "_id",
      as: "user",
      pipeline: [ // Sub-pipeline optimizado
        { $project: { _id: 1, risk_score: 1 } }
      ]
  }},
  { $unwind: "$user" },
  { $set: {
      risk_weight: { $multiply: ["$amount", "$user.risk_score"] }
  }},
  { $merge: {
      into: "risk_reports",
      whenMatched: "replace",
      whenNotMatched: "insert"
  }}
], {
  allowDiskUse: false, // Forzar trabajo en RAM
  maxTimeMS: 5000,
  comment: "Fraud detection pipeline v2.3"
});
```

**Optimizaciones**:  
- `allowDiskUse: false` → Fallar rápido si el dataset no cabe en memoria.
- Sub-pipelines en `$lookup` reducen datos transferidos en 90%.

---

## 🛡️ Método `update()`: Tácticas de Escritura Masiva

### Técnica Delta Updates + Causal Consistency
```javascript
// Sistema de Inventario - Actualización atómica
db.inventory.updateOne(
  {
    sku: "X203",
    warehouse: "NYC",
    quantity: { $gte: 1 }
  },
  {
    $inc: { quantity: -1 },
    $push: {
      logs: {
        $each: [ { ts: new Date(), action: "sale" } ],
        $slice: -1000 // Mantener solo últimos 1000 logs
      }
    }
  },
  {
    writeConcern: { w: "majority", j: true }, // Journal confirmado
    collation: { locale: "en", numericOrdering: true } // Orden numérico correcto
  }
);
```

**Patrón Clave**:  
- Usar operadores de actualización (`$inc`, `$push` con `$slice`) para evitar race conditions.

---

## 🔍 Métodos "Secretos" del Motor de Storage

### 1. `db.collection.watch()` (Change Streams)
```javascript
// Sistema de Realtime Analytics (CDC)
const pipeline = [{
  $match: {
    operationType: { $in: ["insert", "update"] },
    "fullDocument.category": "premium"
  }
}];

const changeStream = db.orders.watch(pipeline, {
  fullDocument: "updateLookup", // Obtener documento completo
  batchSize: 100,
  maxAwaitTimeMS: 500
});

changeStream.on("change", (change) => {
  kafka.produce("orders-topic", JSON.stringify(change.fullDocument));
});
```

**Uso en Producción**:  
- Base para sistemas CQRS y Event Sourcing de baja latencia.

---

> 💡 **Recurso Final**: Ejecuta `db.currentOp()` en producción para encontrar cuellos de botella en tiempo real.

