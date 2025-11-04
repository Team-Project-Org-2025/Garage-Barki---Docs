# 📊 Modelo: Dashboard

**Ubicación:** `app/models/Dashboard.php`  
**Namespace:** `Barkios\models`  
**Extiende de:** `Barkios\core\Database`

---

## 📘 Descripción general

El modelo **Dashboard** centraliza la obtención de estadísticas, métricas financieras, movimientos e información agregada del sistema para alimentar el panel de control del e-commerce.  

Permite analizar tendencias de ventas, compras, cuentas por cobrar/pagar, inventario, productos y generar datos para gráficos y reportes temporales.

---

## 📂 Dependencias

- **PDO** → Manejo de consultas SQL parametrizadas.  
- **Exception** → Control y registro de errores.  
- **Barkios\core\Database** → Clase base que gestiona la conexión con la base de datos (`$this->db`).

---

## ⚙️ Métodos principales

### 🔹 `getStats($dateFrom, $dateTo)`
Obtiene todas las estadísticas generales del sistema para un rango de fechas determinado.

**Parámetros:**
| Nombre | Tipo | Descripción |
|--------|------|--------------|
| `$dateFrom` | string | Fecha de inicio del período (YYYY-MM-DD) |
| `$dateTo` | string | Fecha de fin del período (YYYY-MM-DD) |

**Incluye:**
- Ventas (`getVentasStats`)
- Compras (`getComprasStats`)
- Cuentas por Cobrar (`getCuentasCobrarStats`)
- Cuentas por Pagar (`getCuentasPagarStats`)
- Inventario (`getInventarioStats`)
- Productos (`getProductosStats`)

**Retorna:**  
`array` con todas las métricas agregadas.

---

### 🔹 `getVentasStats($dateFrom, $dateTo)`
Obtiene las estadísticas de **ventas** para el rango de fechas indicado.

**Consulta SQL:**
```sql
SELECT COUNT(*) as cantidad,
       COALESCE(SUM(monto_total), 0) as total,
       COALESCE(AVG(monto_total), 0) as promedio
FROM ventas
WHERE DATE(fecha) BETWEEN :from AND :to
  AND estado_venta != 'cancelada'
```

**Datos calculados:**

- `cantidad` → número de ventas.

- `total` → monto total de ventas.

- `promedio` → promedio por venta.

- `tendencia` → variación porcentual respecto al período anterior

### 🔹 getCuentasCobrarStats()

Obtiene métricas sobre **cuentas por cobrar** activas.

**Consulta SQL:**
```sql
SELECT COUNT(DISTINCT cc.cuenta_cobrar_id) as cantidad,
       COALESCE(SUM(v.saldo_pendiente), 0) as saldo_total,
       COUNT(DISTINCT CASE WHEN cc.estado = 'vencido' THEN cc.cuenta_cobrar_id END) as vencidas,
       COUNT(DISTINCT CASE 
            WHEN DATEDIFF(cc.vencimiento, NOW()) <= 3 
            AND cc.estado = 'pendiente' THEN cc.cuenta_cobrar_id END) as por_vencer
FROM cuentas_cobrar cc
INNER JOIN credito cr ON cc.credito_id = cr.credito_id
INNER JOIN ventas v ON cr.venta_id = v.venta_id
WHERE cc.estado IN ('pendiente', 'vencido')
  AND v.estado_venta != 'cancelada'
```

**Métricas:**

- `cantidad` → número total de cuentas.

- `saldo_total` → saldo total pendiente.

- `vencidas` y `por_vencer` dentro de 3 días.

### 🔹 getCuentasPagarStats()

Genera estadísticas sobre cuentas por pagar pendientes o vencidas.

**Consulta SQL principal:**
```sql
SELECT COUNT(DISTINCT cp.cuenta_pagar_id) as cantidad,
       COALESCE(SUM(
           cp.monto - COALESCE((
               SELECT SUM(pc.monto)
               FROM pagos_compras pc
               WHERE pc.cuenta_pagar_id = cp.cuenta_pagar_id
               AND pc.estado_pago = 'CONFIRMADO'
           ), 0)
       ), 0) as saldo_total,
       COUNT(DISTINCT CASE WHEN cp.estado = 'vencido' THEN cp.cuenta_pagar_id END) as vencidas,
       COUNT(DISTINCT CASE 
           WHEN DATEDIFF(cp.fecha_vencimiento, NOW()) <= 7 
           AND cp.estado = 'pendiente' THEN cp.cuenta_pagar_id END) as por_vencer
FROM cuentas_pagar cp
INNER JOIN compras c ON cp.compra_id = c.compra_id
WHERE cp.estado IN ('pendiente', 'vencido')
  AND c.activo = 1
```

**Métricas:**

- Total de cuentas.

- Monto total pendiente.

- Cuentas vencidas y próximas a vencer (≤7 días).

### 🔹 getInventarioStats($dateFrom, $dateTo)

Devuelve el total de **prendas vendidas y disponibles**.

**Consultas:**

1. Prendas vendidas:
```sql
SELECT COUNT(DISTINCT p.codigo_prenda) as vendidas
FROM prendas p
INNER JOIN detalle_venta dv ON p.codigo_prenda = dv.codigo_prenda
INNER JOIN ventas v ON dv.venta_id = v.venta_id
WHERE DATE(v.fecha) BETWEEN :from AND :to
  AND v.estado_venta != 'cancelada'
```

2. Prendas disponibles:
```sql
SELECT COUNT(*) as disponibles
FROM prendas
WHERE estado = 'DISPONIBLE' AND activo = 1
```

**Retorna:**
`array` con las claves `vendidas` y `disponibles`.

### 🔹 getProductosStats()

Obtiene estadísticas de **productos** totales en el sistema.

**Consultas:**

1. Conteo general:
```sql
SELECT COUNT(*) as total,
       COUNT(CASE WHEN estado = 'DISPONIBLE' THEN 1 END) as disponibles,
       COUNT(CASE WHEN estado = 'VENDIDA' THEN 1 END) as vendidas
FROM prendas
WHERE activo = 1
```

2. Valor total de inventario disponible:
```sql
SELECT COALESCE(SUM(precio), 0) as valor_total
FROM prendas
WHERE estado = 'DISPONIBLE' AND activo = 1
```

**Retorna:**
Totales, productos disponibles, vendidas y valor monetario total.

### 🔹 getChartTimeline($dateFrom, $dateTo, $filter)

Genera los datos para **gráficos de línea temporal** (ventas y compras).

| Nombre      | Tipo   | Descripción                                |
| ----------- | ------ | ------------------------------------------ |
| `$dateFrom` | string | Fecha inicial                              |
| `$dateTo`   | string | Fecha final                                |
| `$filter`   | string | Tipo de agrupación: `day`, `month`, `year` |

```php
[
  'labels' => [...],
  'ventas' => [...],
  'compras' => [...]
]
```

**Agrupa resultados** según el filtro:

- `year` → agrupado por mes (%Y-%m)

- `month` → agrupado por día

- `default` → agrupado por fecha exacta

### 🔹 formatPeriodLabel($periodo, $filter)

Convierte un valor de período (2025-11 o 2025-11-04) en una **etiqueta legible** para el gráfico.

| Filtro  | Formato de salida |
| ------- | ----------------- |
| `year`  | `"Nov 2025"`      |
| `month` | `"04 Nov"`        |
| `day`   | `"04/11"`         |

### 🔹 getTransactions($dateFrom, $dateTo, $limit = 50)

Obtiene las **transacciones más recientes** (ventas y compras).

**Consulta Ventas:**
```sql
SELECT v.fecha, 'VENTA' as tipo, v.referencia,
       c.nombre_cliente as cliente_proveedor,
       v.monto_total as monto, v.estado_venta as estado
FROM ventas v
INNER JOIN clientes c ON v.cliente_ced = c.cliente_ced
WHERE DATE(v.fecha) BETWEEN :from AND :to
ORDER BY v.fecha DESC
LIMIT :limit
```

**Consulta Compras:**
```sql
SELECT c.fecha_compra as fecha, 'COMPRA' as tipo,
       c.factura_numero as referencia,
       p.nombre_empresa as cliente_proveedor,
       c.monto_total as monto, 'completada' as estado
FROM compras c
INNER JOIN proveedores p ON c.proveedor_rif = p.proveedor_rif
WHERE DATE(c.fecha_compra) BETWEEN :from AND :to
  AND c.activo = 1
ORDER BY c.fecha_compra DESC
LIMIT :limit
```

**Resultado:**
Array unificado de transacciones (ventas + compras) ordenadas por fecha descendente.

### 🧱 Tablas SQL involucradas

- `ventas`

- `compras`

- `cuentas_cobrar`

- `cuentas_pagar`

- `prendas`

- `detalle_venta`

- `clientes`

- `proveedores`

### 🧩 Relaciones con otros modelos

| Relación    | Modelo      | Descripción                        |
| ----------- | ----------- | ---------------------------------- |
| `Sales`     | Ventas      | Se utiliza para métricas de ventas |
| `Purchases` | Compras     | Cálculo de compras y pagos         |
| `Clients`   | Clientes    | Identificación de compradores      |
| `Suppliers` | Proveedores | Relación con compras               |
| `Inventory` | Prendas     | Análisis de stock disponible       |

### 🧠 Consideraciones técnicas

- Todas las consultas son **parametrizadas** para evitar inyecciones SQL.

- Maneja períodos dinámicos con cálculo automático de tendencias.

- Incluye comparaciones con períodos anteriores para medir desempeño.

- Los errores se registran en el log mediante `error_log()`. 

- Los métodos privados (`getVentasStats`, `getComprasStats`, etc.) son invocados internamente desde `getStats()`.

### 📊 Ejemplo de uso
```php
use Barkios\models\Dashboard;

$dashboard = new Dashboard();

// Estadísticas generales
$stats = $dashboard->getStats('2025-01-01', '2025-01-31');

// Datos de gráfico
$chartData = $dashboard->getChartTimeline('2025-01-01', '2025-01-31', 'month');

// Transacciones recientes
$transactions = $dashboard->getTransactions('2025-01-01', '2025-01-31', 20);
```
