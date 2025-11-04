# 🛍️ Modelo: Product

**Ubicación:** `app/models/Product.php`  
**Namespace:** `Barkios\models`  
**Extiende de:** `Barkios\core\Database`

---

## 📘 Descripción general

El modelo **Product** se encarga de gestionar todas las operaciones relacionadas con los **productos (prendas)** del sistema.  
Este modelo controla el ciclo de vida de cada producto, desde su creación y actualización hasta su eliminación lógica o física, incluyendo el manejo de imágenes, estados (`DISPONIBLE`, `VENDIDA`, `ELIMINADA`), y filtrado por categorías.

---

## 📂 Dependencias

- **PDO** → Para ejecutar consultas seguras y parametrizadas.  
- **Exception** → Para el manejo controlado de errores y validaciones.  
- **Barkios\core\Database** → Clase base que establece la conexión a la base de datos (`$this->db`).

---

## ⚙️ Métodos principales

### 🔹 `__construct()`
Constructor que inicializa la conexión con la base de datos heredada de la clase `Database`.

---

### 🔹 `getAll()`
Obtiene todas las prendas activas del sistema, ya sean **disponibles** o **vendidas**.

**Consulta SQL:**
```sql
SELECT * FROM prendas 
WHERE activo = 1 AND estado IN ('DISPONIBLE', 'VENDIDA')
ORDER BY codigo_prenda ASC
```

**Retorna:**
- `array` — Lista de productos activos.

**Manejo de errores:**
- Registra en log si ocurre una excepción (`Error al obtener productos`).

### 🔹 getDisponibles()

Obtiene todos los productos disponibles (no vendidos ni eliminados).

**Consulta SQL:**

```sql
SELECT * FROM prendas
WHERE activo = 1 AND estado = 'DISPONIBLE'
ORDER BY codigo_prenda ASC
```

**Retorna:**
`array` con los productos disponibles.

### 🔹 getById(int $id)

Obtiene los datos de una prenda específica.

**Parámetros:**

- `$id` (int) — Identificador (prenda_id).

**Consulta SQL:**
```sql
SELECT * FROM prendas WHERE prenda_id = :id
```

**Retorna:**

- `array` con los datos del producto, o

- `null` si no existe.

### 🔹 productExists(int $id)

Verifica si existe un producto con el prenda_id dado.

**Parámetros:**

- `$id` (int) — ID del producto.

**Consulta SQL:**
```sql
SELECT COUNT(*) FROM prendas WHERE prenda_id = :id
```

**Retorna:**
- `true` si existe, `false` en caso contrario.

### 🔹 add(...)

Agrega un nuevo producto con todos sus datos y opcionalmente una imagen.

**Parámetros:**

| Nombre           | Tipo          | Descripción                               |
| ---------------- | ------------- | ----------------------------------------- |
| `$id`            | int           | ID o código único                         |
| `$nombre`        | string        | Nombre del producto                       |
| `$tipo`          | string        | Tipo de prenda (ej. “camisa”, “pantalón”) |
| `$categoria`     | string        | Categoría asociada                        |
| `$precio`        | float         | Precio de venta                           |
| `$imagen`        | string | null | Ruta o nombre del archivo de imagen       |
| `$descripcion`   | string | null | Descripción del producto                  |
| `$precio_compra` | float | null  | Precio de adquisición                     |

**Proceso:**

1. Valida que el `prenda_id` y `codigo_prenda` no existan.

2. Inserta el registro con estado 'DISPONIBLE' y `activo = 1`.

**Consulta SQL:**
```sql
INSERT INTO prendas (codigo_prenda, nombre, tipo, categoria, precio, precio_compra, imagen, descripcion, activo, estado)
VALUES (:codigo_prenda, :nombre, :tipo, :categoria, :precio, :precio_compra, :imagen, :descripcion, 1, 'DISPONIBLE')
```

**Retorna:**
`true` si la inserción fue exitosa.

**Lanza excepción:**

- Si el producto ya existe por ID o código.

### 🔹 update(...)

Actualiza los datos de un producto existente.
Permite decidir si se actualiza o no la imagen del producto.

**Parámetros principales:**

| Nombre           | Tipo          | Descripción                    |
| ---------------- | ------------- | ------------------------------ |
| `$id`            | int           | ID del producto                |
| `$nombre`        | string        | Nuevo nombre                   |
| `$tipo`          | string        | Tipo de producto               |
| `$categoria`     | string        | Categoría                      |
| `$precio`        | float         | Precio actualizado             |
| `$imagen`        | string | null | Nueva imagen (opcional)        |
| `$descripcion`   | string | null | Nueva descripción              |
| `$updateImage`   | bool          | Si `true`, actualiza la imagen |
| `$precio_compra` | float | null  | Nuevo precio de compra         |

**Validaciones:**

- No permite editar productos vendidos o eliminados.

**Consulta SQL (según `$updateImage`):**
```sql
UPDATE prendas SET 
    nombre = :nombre,
    tipo = :tipo,
    categoria = :categoria,
    precio = :precio,
    precio_compra = :precio_compra,
    [imagen = :imagen,]
    descripcion = :descripcion,
    fec_actualizacion = NOW()
WHERE prenda_id = :prenda_id
```

**Retorna:**
`true` si se actualizó correctamente.

### 🔹 marcarVendida($id)

Cambia el estado de una prenda a 'VENDIDA'.
```sql
UPDATE prendas SET estado = 'VENDIDA' WHERE prenda_id = :prenda_id
```

### 🔹 liberarPrenda(int $id)

Cambia el estado de una prenda a DISPONIBLE nuevamente.

**Consulta SQL:**
```sql
UPDATE prendas 
SET estado = 'DISPONIBLE' 
WHERE prenda_id = :prenda_id;
```

### 🔹 delete(int $id)

Elimina lógicamente una prenda (sin borrarla de la base de datos).

**Consulta SQL:**
```sql
UPDATE prendas 
SET activo = 0, estado = 'ELIMINADA' 
WHERE prenda_id = :prenda_id;
```

### 🔹 deletePhysically(int $id)

Elimina definitivamente una prenda de la base de datos.

**Consulta SQL:**
```sql
DELETE FROM prendas 
WHERE prenda_id = :id;
```

### 🔹 getImagePath(int $id)

Obtiene la ruta de imagen de una prenda.

**Consulta SQL:**
```sql
SELECT imagen FROM prendas 
WHERE prenda_id = :id;
```

**Retorna:**
- `string|null` — Ruta de imagen o null.

### 🔹 updateImage(int $id, string $imagePath)

Actualiza solo la imagen del producto.

**Consulta SQL:**
```sql
UPDATE prendas 
SET imagen = :imagen, fec_actualizacion = NOW() 
WHERE prenda_id = :id;
```

### 🔹 removeImage(int $id)

Elimina la referencia de imagen (deja `NULL` en el campo).

**Consulta SQL:**
```sql
UPDATE prendas 
SET imagen = NULL, fec_actualizacion = NOW() 
WHERE prenda_id = :id;
```

### 🔹 getLatest(int $limit = 8)

Obtiene las prendas más recientes según `fecha_creacion`.

**Consulta SQL:**
```sql
SELECT * FROM prendas
WHERE activo = 1 
  AND estado = 'DISPONIBLE'
ORDER BY fecha_creacion DESC
LIMIT :limit;
```

### 🔹 getByCategoria(string $categoria, ?int $limit = null)

Obtiene productos filtrados por categoría.

**Consulta SQL:**
```sql
SELECT * FROM prendas
WHERE activo = 1 
  AND estado = 'DISPONIBLE'
  AND categoria = :categoria
ORDER BY fecha_creacion DESC
LIMIT :limit;
```

### 🧠 Ejemplo de uso
```php
use Barkios\models\Product;

$product = new Product();

// Agregar producto
$product->add(101, 'Camisa Azul', 'Camisa', 'Hombres', 25.50, 'uploads/101.jpg', 'Camisa casual', 15.00);

// Consultar productos disponibles
$productos = $product->getDisponibles();

// Marcar como vendida
$product->marcarVendida(101);
```