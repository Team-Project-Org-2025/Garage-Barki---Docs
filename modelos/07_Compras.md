# Modelo Purchase

## Descripción
Modelo para gestionar compras (`compras`), validaciones, inserción, actualización, eliminación lógica y estadísticas.

---

## Métodos

### 🔹 facturaExiste(string $numeroFactura, int|null $excludeId = null)
Verifica si existe una compra activa con el número de factura dado, opcionalmente excluyendo una compra por ID.

- **Parámetros**:
  - `$numeroFactura` (string): número de factura a buscar
  - `$excludeId` (int|null): ID de compra a excluir de la búsqueda (opcional)
- **Retorna**: bool (true si existe)
- **SQL**:
  ```sql
  SELECT COUNT(*) FROM compras 
  WHERE factura_numero = :factura_numero AND activo = 1
  [AND compra_id != :exclude_id]
```

### 🔹 codigoPrendaExiste(string $codigo)

Verifica si existe una prenda activa con el código dado (mayúsculas y trim).

- **Parámetros**:
  - `$codigo` (string): código prenda

**Retorna**: bool

**Consulta SQL**:
```sql
SELECT COUNT(*) FROM prendas 
WHERE codigo_prenda = :codigo AND activo = 1;
```

### 🔹 proveedorExiste(string $proveedorRif)

Verifica si existe un proveedor con el rif dado.

- **Parámetros**:
  - `$proveedorRif` (string): rif del proveedor

**Retorna**: bool

**Consulta SQL**:
```sql
SELECT COUNT(*) FROM proveedores WHERE proveedor_rif = :rif;
```

### 🔹 getMontoTotal(int $compraId)

Obtiene el monto total de una compra activa.

- **Parámetros**:
  - `$compraId` (int): ID de la compra

**Retorna**: float (0 si falla)

**Consulta SQL**:
```sql
SELECT monto_total FROM compras WHERE compra_id = :id AND activo = 1;
```

### 🔹 getAll()

Obtiene todas las compras activas con datos relacionados, incluyendo totales y estados de prendas y pagos.

- **Parámetros**: ninguno

**Retorna**: array de compras con detalles agregados

**Consulta SQL (resumida)**:
```sql
SELECT 
  c.*,
  p.nombre_empresa as nombre_proveedor,
  p.nombre_contacto,
  p.tipo_rif,
  COUNT(DISTINCT pr.prenda_id) as total_prendas,
  SUM(CASE WHEN pr.estado = 'DISPONIBLE' THEN 1 ELSE 0 END) as prendas_disponibles,
  SUM(CASE WHEN pr.estado = 'VENDIDA' THEN 1 ELSE 0 END) as prendas_vendidas,
  cp.cuenta_pagar_id,
  cp.estado as estado_pago,
  cp.fecha_vencimiento,
  COALESCE((SELECT SUM(pc.monto) FROM pagos_compras pc WHERE pc.cuenta_pagar_id = cp.cuenta_pagar_id AND pc.estado_pago = 'CONFIRMADO'), 0) as total_pagado,
  (c.monto_total - COALESCE(...)) as saldo_pendiente,
  CASE WHEN cp.fecha_vencimiento < CURDATE() AND cp.estado = 'pendiente' THEN 1 ELSE 0 END as vencida
FROM compras c
JOIN proveedores p ON c.proveedor_rif = p.proveedor_rif
LEFT JOIN prendas pr ON c.compra_id = pr.compra_id AND pr.activo = 1
LEFT JOIN cuentas_pagar cp ON c.compra_id = cp.compra_id
WHERE c.activo = 1
GROUP BY c.compra_id
ORDER BY c.fecha_compra DESC, c.compra_id DESC;
```

### 🔹 getById(int $id)

Obtiene una compra activa con detalles del proveedor, cuenta por pagar y pagos.

- **Parámetros**:
  - `$id` (int): ID de la compra

**Retorna**: array o null

**Consulta SQL (resumida)**:
```sql
SELECT
  c.*,
  p.nombre_empresa as nombre_proveedor,
  p.nombre_contacto,
  p.direccion as direccion_proveedor,
  p.telefono as telefono_proveedor,
  p.correo as correo_proveedor,
  p.tipo_rif,
  cp.cuenta_pagar_id,
  cp.estado as estado_pago,
  cp.fecha_vencimiento,
  cp.observaciones as observaciones_pago,
  COALESCE((SELECT SUM(pc.monto) FROM pagos_compras pc WHERE pc.cuenta_pagar_id = cp.cuenta_pagar_id AND pc.estado_pago = 'CONFIRMADO'), 0) as total_pagado,
  (c.monto_total - COALESCE(...)) as saldo_pendiente
FROM compras c
JOIN proveedores p ON c.proveedor_rif = p.proveedor_rif
LEFT JOIN cuentas_pagar cp ON c.compra_id = cp.compra_id
WHERE c.compra_id = :id AND c.activo = 1;
```

### 🔹 getPrendasByCompraId(int $compraId)

Obtiene prendas activas asociadas a una compra con detalles y cálculo de margen y porcentaje de ganancia.

- **Parámetros**:
  - `$compraId` (int): ID de la compra

**Retorna**: array de prendas o array vacío

**Consulta SQL**:
```sql
SELECT
  pr.prenda_id,
  pr.codigo_prenda,
  pr.nombre,
  pr.categoria,
  pr.tipo,
  pr.precio_compra as precio_costo,
  pr.precio as precio_venta,
  pr.descripcion,
  pr.estado,
  pr.fecha_creacion,
  dc.detalle_compra_id,
  (pr.precio - pr.precio_compra) as margen_ganancia,
  ((pr.precio - pr.precio_compra) / pr.precio_compra * 100) as porcentaje_ganancia
FROM prendas pr
LEFT JOIN detalle_compra dc ON pr.codigo_prenda = dc.codigo_prenda AND dc.compra_id = :compra_id
WHERE pr.compra_id = :compra_id AND pr.activo = 1
ORDER BY pr.categoria, pr.nombre;
```

### 🔹 add(array $datos)

Agrega una nueva compra con sus prendas y crea cuenta por pagar.

- **Parámetros**:

- `$datos` (array): debe contener:

- `proveedor_rif` (string)

- `factura_numero` (string)

- `fecha_compra` (string fecha)

- `tracking` (string)

- `monto_total` (float)

- `observaciones` (string)

- `prendas` (array de prendas con: codigo_prenda, nombre, categoria, tipo, precio_venta, precio_costo, descripcion)

**Retorna**: int ID compra creada

**Excepciones**: Lanza Exception si falla alguna validación o inserción

**SQL inserción compra:**
```sql
INSERT INTO compras (proveedor_rif, factura_numero, fecha_compra, tracking, monto_total, observaciones, pdf_generado, activo)
VALUES (:proveedor_rif, :factura_numero, :fecha_compra, :tracking, :monto_total, :observaciones, 0, 1);
```

- **SQL inserción prendas** (cada una):
```sql
INSERT INTO prendas (codigo_prenda, compra_id, nombre, categoria, tipo, precio, precio_compra, descripcion, estado, activo)
VALUES (:codigo_prenda, :compra_id, :nombre, :categoria, :tipo, :precio, :precio_compra, :descripcion, 'DISPONIBLE', 1);
```

- **SQL inserción detalle_compra** (cada prenda):
```sql
INSERT INTO detalle_compra (compra_id, codigo_prenda, precio_compra)
VALUES (:compra_id, :codigo_prenda, :precio_compra);
```

- **SQL inserción cuenta por pagar:**
```sql
INSERT INTO cuentas_pagar (compra_id, proveedor_rif, monto, fecha, fecha_vencimiento, estado, observaciones)
VALUES (:compra_id, :proveedor_rif, :monto, :fecha, :fecha_vencimiento, 'pendiente', :observaciones);
```

### 🔹 update(int $id, array $datos)

Actualiza compra y cuenta por pagar.

- **Parámetros**:

- `$id` (int): ID compra

- `$datos` (array) con campos similares a add

**Retorna**: bool

**Excepciones**: Lanza Exception si número de factura ya existe en otra compra o proveedor no existe

- **SQL actualización compra:**
```sql
UPDATE compras
SET proveedor_rif = :proveedor_rif,
    factura_numero = :factura_numero,
    fecha_compra = :fecha_compra,
    tracking = :tracking,
    monto_total = :monto_total,
    observaciones = :observaciones,
    fec_actualizacion = CURRENT_TIMESTAMP
WHERE compra_id = :id;
```

- **SQL actualización cuenta por pagar:**
```sql
UPDATE cuentas_pagar
SET proveedor_rif = :proveedor_rif,
    monto = :monto,
    fec_actualizacion = CURRENT_TIMESTAMP
WHERE compra_id = :compra_id;
```

### 🔹 addPrendasToCompra(int $compraId, array $prendas)

Agrega prendas nuevas a una compra existente, actualizando montos.

- **Parámetros:**

- `$compraId` (int)

- `$prendas` (array de prendas con estructura similar a add)

**Retorna**: int cantidad de prendas agregadas

**Excepciones**: Lanza Exception si hay códigos duplicados

**SQL actualización monto compra:**
```sql
UPDATE compras 
SET monto_total = :monto_total, fec_actualizacion = CURRENT_TIMESTAMP
WHERE compra_id = :id;
```

- **SQL actualización monto cuentas por pagar:**
```sql
UPDATE cuentas_pagar
SET monto = :monto, fec_actualizacion = CURRENT_TIMESTAMP
WHERE compra_id = :compra_id;
```

### 🔹 delete(int $id)

Eliminación lógica de compra, prendas y cuenta por pagar si no hay prendas vendidas ni pagos confirmados.

**Parámetros:**

- `$id` (int)

**Retorna**: bool

**Excepciones:** Lanza Exception si hay prendas vendidas o pagos confirmados

- **SQL conteo prendas vendidas:**
```sql
SELECT COUNT(*) as vendidas FROM prendas WHERE compra_id = :id AND estado = 'VENDIDA' AND activo = 1;
```

- **SQL conteo pagos confirmados:**
```sql
SELECT COUNT(*) as total_pagos
FROM pagos_compras pc
JOIN cuentas_pagar cp ON pc.cuenta_pagar_id = cp.cuenta_pagar_id
WHERE cp.compra_id = :id AND pc.estado_pago = 'CONFIRMADO';
```

- **SQL actualización compra:**
```sql
UPDATE compras SET activo = 0, fec_actualizacion = CURRENT_TIMESTAMP WHERE compra_id = :id;
```

- **SQL actualización cuentas por pagar:**
```sql
UPDATE cuentas_pagar SET estado = 'cancelado', fec_actualizacion = CURRENT_TIMESTAMP WHERE compra_id = :id;
```

- **SQL actualización prendas:**
```sql
UPDATE prendas SET activo = 0, estado = 'ELIMINADA' WHERE compra_id = :id;
```

🔹 getPagosByCompraId(int $compraId)

Obtiene pagos no anulados asociados a una compra, ordenados por fecha desc.

- **Parámetros:**

- `$compraId` (int)

**Retorna**: array de pagos o vacío

**Consulta SQL:**
```sql
SELECT pc.*, cp.cuenta_pagar_id
FROM pagos_compras pc
JOIN cuentas_pagar cp ON pc.cuenta_pagar_id = cp.cuenta_pagar_id
WHERE cp.compra_id = :compra_id
AND pc.estado_pago != 'ANULADO'
ORDER BY pc.fecha_pago DESC;
```

### 🔹 markPdfGenerated(int $id)

Marca compra indicando que el PDF fue generado.

- **Parámetros:**

- `$id` (int)

**Retorna**: bool

**Consulta SQL:**
```sql
UPDATE compras SET pdf_generado = 1, fec_actualizacion = CURRENT_TIMESTAMP WHERE compra_id = :id;
```

### 🔹 getEstadisticas()

Obtiene estadísticas agregadas sobre compras, proveedores, prendas y cuentas pendientes.

- **Parámetros:** ninguno

**Retorna**: array con estadísticas

**Consulta SQL (resumida):**
```sql
SELECT 
  COUNT(DISTINCT c.compra_id) as total_compras,
  SUM(c.monto_total) as monto_total_compras,
  COUNT(DISTINCT c.proveedor_rif) as total_proveedores,
  (SELECT COUNT(*) FROM prendas WHERE activo = 1 AND estado = 'DISPONIBLE') as prendas_disponibles,
  (SELECT COUNT(*) FROM prendas WHERE activo = 1 AND estado = 'VENDIDA') as prendas_vendidas,
  (SELECT SUM(precio_compra) FROM prendas WHERE activo = 1 AND estado = 'DISPONIBLE') as valor_inventario,
  COALESCE((
    SELECT SUM(c2.monto_total - COALESCE(
      (SELECT SUM(pc.monto) FROM pagos_compras pc
       JOIN cuentas_pagar cp2 ON pc.cuenta_pagar_id = cp2.cuenta_pagar_id
       WHERE cp2.compra_id = c2.compra_id AND pc.estado_pago = 'CONFIRMADO'), 0))
    FROM compras c2
    JOIN cuentas_pagar cp ON c2.compra_id = cp.compra_id
    WHERE c2.activo = 1 AND cp.estado = 'pendiente'
  ), 0) as saldo_pendiente_total,
  (SELECT COUNT(*) FROM cuentas_pagar WHERE estado = 'pendiente' AND fecha_vencimiento < CURDATE()) as cuentas_vencidas
FROM compras c
WHERE c.activo = 1;
```

### 🔹 canEdit(int $compraId)

Verifica si la compra puede ser editada (ninguna prenda vendida).

- **Parámetros:**

- `$compraId` (int)

**Retorna**: bool

**Consulta SQL:**
```sql
SELECT COUNT(*) as vendidas FROM prendas WHERE compra_id = :id AND estado = 'VENDIDA' AND activo = 1;
```

## Métodos privados

### 🔹 insertarPrenda(int $compraId, array $prenda)

Inserta una prenda en la compra y detalle_compra.

- **Parámetros:**

- `$compraId` (int)

- `$prenda` (array): debe incluir codigo_prenda, nombre, categoria, tipo, precio_venta, precio_costo, descripcion

**Retorna**: float precio de costo de la prenda

- **Consulta SQL inserción prenda:**
```sql
INSERT INTO prendas (codigo_prenda, compra_id, nombre, categoria, tipo, precio, precio_compra, descripcion, estado, activo)
VALUES (:codigo_prenda, :compra_id, :nombre, :categoria, :tipo, :precio, :precio_compra, :descripcion, 'DISPONIBLE', 1);
```

- **Consulta SQL inserción detalle_compra:**
```sql
INSERT INTO detalle_compra (compra_id, codigo_prenda, precio_compra)
VALUES (:compra_id, :codigo_prenda, :precio_compra);
```

### 🔹 crearCuentaPorPagar(int $compraId, array $datos)

Crea registro en cuentas_pagar para la compra.

- **Parámetros:**

- `$compraId` (int)

- `$datos` (array): debe incluir proveedor_rif, monto_total, fecha_compra, factura_numero, opcional fecha_vencimiento

**Retorna**: void

**Consulta SQL:**
```sql
INSERT INTO cuentas_pagar (compra_id, proveedor_rif, monto, fecha, fecha_vencimiento, estado, observaciones)
VALUES (:compra_id, :proveedor_rif, :monto, :fecha, :fecha_vencimiento, 'pendiente', :observaciones);
```