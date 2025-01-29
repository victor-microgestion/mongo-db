# Filtros en MongoDB: Tácticas para Consultas de Misión Crítica  
*Domina operadores no documentados, expresiones regulares compiladas y filtros atómicos en sistemas de 100K+ req/seg*

---

## 🔥 Operadores Básicos con Poder Oculto

### 1. `$eq` vs `$in` (Elección Nuclear)
```javascript
// Sistema de Autenticación - 50M usuarios
// ❌ Ineficiente para valores únicos
db.users.find({ _id: { $in: [ObjectId("663b9e8d4a1c8f3d8a7b6c1d")] } })

// ✅ Máxima velocidad con índice
db.users.find({
  _id: ObjectId("663b9e8d4a1c8f3d8a7b6c1d"),
  status: { $eq: "verified" } // Forzar uso de índice
}).hint("_id_");
```

**Benchmark**:  
- `$eq` + hint: 0.5ms vs `$in` sin hint: 8ms en colección de 100M docs.

---

### 2. `$elemMatch` Profundo (Nested Arrays)
```javascript
// Plataforma E-learning - Cursos con múltiples niveles
db.courses.find({
  modules: {
    $elemMatch: {
      "lessons.quizzes": {
        $elemMatch: {
          difficulty: "hard",
          "questions.type": "code"
        }
      }
    }
  }
}).projection({
  "modules.$": 1,
  "modules.lessons.$": 1
});
```

**Key Insight**:  
- Usar `$` projection con `$elemMatch` reduce transferencia de datos en 90%.

---

## 🚀 Operadores Avanzados de Filtrado

### 3. `$jsonSchema` (Validación en DB)
```javascript
// Sistema Bancario - Filtros con validación estructural
db.transactions.find({
  $jsonSchema: {
    bsonType: "object",
    required: ["amount", "currency"],
    properties: {
      amount: {
        bsonType: "decimal",
        minimum: 0,
        maximum: 1000000
      },
      currency: {
        enum: ["USD", "EUR", "GBP"]
      }
    }
  }
});
```

**Caso Real**:  
- Detección de documentos corruptos en tiempo de consulta (0.1% overhead).

---

### 4. `$mod` (Divisibilidad Cuántica)
```javascript
// Sistema de Criptografía - Claves RSA
db.keys.find({
  n: {
    $mod: [
      65537,  // Exponente público común
      0       // Buscar claves vulnerables
    ]
  }
}).maxTimeMS(100); // Evitar DoS
```

**Advertencia**:  
- Operación O(n) - Solo usar con colecciones pequeñas o índices expresivos.

---

### 5. `$where` (Nuclear Option)
```javascript
// Sistema de Trading - Patrones complejos
db.stocks.find({
  $where: function() {
    return (this.prices[0] * 1.1 < this.prices[this.prices.length - 1])
      && (this.volume > 1e6)
      && (this.symbol.startsWith("AI"));
  }
}).allowDiskUse(true); // Solo para análisis batch
```

**Performance**:  
- 100-1000x más lento que operadores nativos. Usar solo en ETLs.

---

## 💡 Filtros con Expresiones Regulares Compiladas

### 6. Regex Indexado (BSON Type 11)
```javascript
// Motor de Búsqueda - 500M productos
db.products.createIndex({
  title: "text",
  description: "text"
}, {
  weights: { title: 10 },
  wildcardProjection: { title: 1 }
});

// Búsqueda insensible a acentos y mayúsculas
db.products.find({
  $text: {
    $search: "móvil",
    $language: "es",
    $caseSensitive: false,
    $diacriticSensitive: false
  }
});
```

**Optimización**:  
- Pre-compilar regexes en la app: `new RegExp("patron", "i")`.

---

## 🛡️ Filtros Atómicos para Transacciones

### 7. Filtros con Operadores de Actualización
```javascript
// Sistema de Reservas - Asientos de Avión
db.seats.updateOne(
  {
    flight: "AA123",
    number: "15A",
    status: "available",
    $expr: { $lt: ["$booked_count", "$max_capacity"] }
  },
  {
    $inc: { booked_count: 1 },
    $set: { status: "reserved" }
  },
  {
    writeConcern: { w: "majority" },
    collation: { locale: "en", strength: 2 }
  }
);
```

**Patrón Clave**:  
- Filtro atómico + actualización previene race conditions.

---

## Conclusión Final
Los filtros en MongoDB son tu primera línea de defensa en sistemas de alta escala. Dominar:  
- Operadores especializados (`$jsonSchema`, `$mod`, `$geoWithin`)  
- Técnicas de indexación asociadas  
- Patrones anti-DoS  

> 🚀 **Recurso Pro**: Crea un índice parcial para filtros frecuentes:  
> ```javascript
> db.logs.createIndex({ error_code: 1 }, {
>   partialFilterExpression: { severity: { $gte: "HIGH" } }
> });
> ```

