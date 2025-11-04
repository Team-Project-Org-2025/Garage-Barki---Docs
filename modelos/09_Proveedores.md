# Modelo Supplier (Proveedor)

## Descripción
Modelo para gestionar proveedores: obtener lista, validar existencia, agregar, actualizar, eliminar (lógica) y búsqueda rápida.

---

## Métodos

### 🔹 getAll()
Obtiene todos los proveedores activos.

- **Parámetros**: ninguno
- **Retorna**: array con proveedores activos
- **Consulta SQL**:
```sql
SELECT * FROM proveedores WHERE activo = 1;
```

### 🔹 supplierExists(string $proveedor_rif)

Verifica si existe un proveedor con el RIF dado.

- **Parámetros:**

- `$proveedor_rif` (string)

- **Retorna**: bool (true si existe)

**Consulta SQL:**
```sql
SELECT COUNT(*) FROM proveedores WHERE proveedor_rif = :proveedor_rif;
```

### 🔹 getById(string $proveedor_rif)

Obtiene un proveedor por su RIF.

- **Parámetros:**

- `$proveedor_rif` (string)

- **Retorna**: array con datos del proveedor o null si no existe

**Consulta SQL:**
```sql
SELECT * FROM proveedores WHERE proveedor_rif = :proveedor_rif;
```

### 🔹 add(string $proveedor_rif, string $nombre_contacto, string $nombre_empresa, string $direccion, string $tipo_rif)

Agrega un nuevo proveedor si no existe el RIF.

- **Parámetros:**

- `$proveedor_rif` (string)

- `$nombre_contacto` (string)

- `$nombre_empresa` (string)

- `$direccion` (string)

- `$tipo_rif` (string)

- **Retorna**: bool (true si se insertó)

- **Excepciones**: lanza Exception si ya existe el proveedor con ese RIF

**Consulta SQL:**
```sql
INSERT INTO proveedores (proveedor_rif, nombre_empresa, nombre_contacto, direccion, tipo_rif)
VALUES (:proveedor_rif, :nombre_empresa, :nombre_contacto, :direccion, :tipo_rif);
```

### 🔹 update(string $proveedor_rif, string $nombre_contacto, string $nombre_empresa, string $direccion, string $tipo_rif)

Actualiza un proveedor existente.

- **Parámetros:**

- `$proveedor_rif` (string)

- `$nombre_contacto` (string)

- `$nombre_empresa` (string)

- `$direccion` (string)

- `$tipo_rif` (string)

- **Retorna**: bool (true si se actualizó)

- **Excepciones**: lanza Exception si no existe el proveedor con ese RIF

**Consulta SQL:**
```sql
UPDATE proveedores
SET nombre_contacto = :nombre_contacto,
    nombre_empresa = :nombre_empresa,
    direccion = :direccion,
    tipo_rif = :tipo_rif
WHERE proveedor_rif = :proveedor_rif;
```

### 🔹 delete(string $proveedor_rif)

Eliminación lógica del proveedor (activo = 0).

- **Parámetros:**

- `$proveedor_rif` (string)

- **Retorna**: bool (true si se eliminó)

**Consulta SQL:**
```sql
UPDATE proveedores SET activo = 0 WHERE proveedor_rif = :proveedor_rif;
```

### 🔹 search(string $term)

Busca proveedores activos cuyo nombre empresa, nombre contacto o RIF coincidan parcialmente con el término.

- **Parámetros:**

- `$term` (string)

- **Retorna**: array con máximo 10 resultados ordenados por nombre_empresa asc

**Consulta SQL:**
```sql
SELECT 
  proveedor_rif AS rif,
  nombre_empresa,
  nombre_contacto
FROM proveedores
WHERE activo = 1
  AND (
    nombre_empresa LIKE :term
    OR nombre_contacto LIKE :term
    OR proveedor_rif LIKE :term
  )
ORDER BY nombre_empresa ASC
LIMIT 10;
```