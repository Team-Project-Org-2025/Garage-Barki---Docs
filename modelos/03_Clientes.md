# 👥 Modelo: Clients

**Ubicación:** `app/models/Clients.php`  
**Namespace:** `Barkios\models`  
**Extiende de:** `Barkios\core\Database`

---

## 📘 Descripción general

El modelo **Clients** se encarga de gestionar todas las operaciones relacionadas con los **clientes** del sistema, incluyendo su registro, modificación, consulta, eliminación lógica y búsqueda avanzada.  

Su objetivo es mantener actualizada la base de datos de clientes activos, así como facilitar la gestión de membresías (por ejemplo: clientes *VIP*).

---

## 📂 Dependencias

- **PDO** → Conexión y ejecución de consultas SQL.  
- **Exception** → Manejo de errores.  
- **Barkios\core\Database** → Clase base que gestiona la conexión a la base de datos y la instancia `$this->db`.

---

## ⚙️ Métodos Principales

### 🔹 `getAll()`
Obtiene todos los clientes activos registrados en el sistema.

**Consulta:**  
```sql
SELECT * FROM clientes WHERE activo = 1 ORDER BY cliente_ced ASC
```

**Retorna:**
`array` — Lista de clientes activos.

**Notas:**

- Si ocurre un error, devuelve un arreglo vacío.

- Filtra solo los clientes con `activo = 1`.

### 🔹 `clientExists($cedula)`

Verifica si un cliente existe en la base de datos según su cédula.

**Parámetros:**

- `$cedula` (string) — Cédula o identificación del cliente.

**Retorna:**
`bool` — `true` si el cliente existe, `false` en caso contrario.

### 🔹 `getById($cedula)`

Obtiene los datos completos de un cliente específico.

**Parámetros:**

- `$cedula` (string) — Cédula del cliente.

**Retorna:**
`array|null` — Información del cliente o `null` si no se encuentra.

### 🔹 add($cedula, $nombre, $direccion, $telefono, $membresia)

Registra un nuevo cliente en la base de datos.

**Parámetros:**

| Parámetro	| Tipo	    | Descripción |
|-----------|-----------|-------------|
|`$cedula`	| string	| Cédula o ID único del cliente |
|`$nombre`	| string	| Nombre completo del cliente |
|`$direccion`	| string	| Dirección física o postal |
|`$telefono`	| string	| Número telefónico |
|`$membresia`	| string	| Tipo o nivel de cliente (ejemplo: `vip`, `regular`) |

**Proceso:**

1. Valida que el cliente no exista (`clientExists`).

2. Inserta un nuevo registro en la tabla `clientes`.

**Consulta SQL:**
```sql
INSERT INTO clientes (cliente_ced, nombre_cliente, direccion, telefono, tipo)
VALUES (:cliente_ced, :nombre_cliente, :direccion, :telefono, :tipo)
```

**Retorna:**
`bool` — `true` si se insertó correctamente.

**Lanza excepción:**
- `"Ya existe un cliente con esta cédula"` si el cliente ya está registrado.

### 🔹 update($cedula, $nombre, $direccion, $telefono, $membresia)

Actualiza los datos de un cliente existente.

**Parámetros:** iguales a los del método `add`.

**Consulta SQL:**
```sql
UPDATE clientes 
SET nombre_cliente = :nombre_cliente, 
    direccion = :direccion, 
    telefono = :telefono, 
    tipo = :tipo 
WHERE cliente_ced = :cliente_ced
```

**Retorna:**
`bool` — Indica si la actualización fue exitosa.

**Lanza excepción:**

- `"No existe un cliente con esta cédula"` si el cliente no está registrado.

### 🔹 delete($cedula)

Realiza una **eliminación lógica** del cliente (no borra físicamente el registro).

**Parámetros:**

- `$cedula` (string) — Cédula del cliente.

**Consulta SQL:**
```sql
UPDATE clientes SET activo = 0 WHERE cliente_ced = :cliente_ced
```

**Retorna:**
`bool` — `true` si se actualizó correctamente.

**Notas:**

- Este enfoque mantiene el historial de clientes pero los excluye de consultas activas.

### 🔹 searchVipClients($query)

Busca clientes **VIP activos** cuyos nombres coincidan con la cadena introducida.

**Parámetros:**

- `$query` (string) — Prefijo del nombre a buscar.

**Consulta SQL:**
```sql
SELECT cliente_ced, nombre_cliente, telefono, correo, tipo
FROM clientes 
WHERE tipo = 'vip' 
  AND activo = 1 
  AND nombre_cliente LIKE :query
ORDER BY nombre_cliente ASC
LIMIT 20
```

**Retorna:**
`array` — Lista de coincidencias.

**Notas:**

- Utiliza búsqueda parcial (`LIKE :query%`).

- Limita los resultados a 20 registros.

- Si ocurre un error, devuelve un arreglo vacío y registra el log.

### 🧱 Estructura SQL relacionada

**Tabla:** `clientes`

| Columna	| Tipo	| Descripción |
|---------|-------|-------------|
| cliente_ced	| VARCHAR	| Identificación única |
| nombre_cliente	| VARCHAR	| Nombre completo |
| direccion	| VARCHAR	| Dirección del cliente |
| telefono	| VARCHAR	| Teléfono de contacto |
| correo	| VARCHAR	| Correo electrónico (opcional) |
| tipo	| ENUM(vip, regular, etc.)	| Tipo de cliente |
| activo	| TINYINT	| Indica si el cliente está activo (1) o eliminado (0) |

### 🧠 Consideraciones técnicas

- Todas las operaciones son simples (sin transacciones), ya que las consultas afectan una única tabla.

- El modelo utiliza eliminación lógica, no física, preservando la integridad histórica.

- Las excepciones son lanzadas para control de errores y validadas en el flujo de controladores.

### 🧩 Relaciones con otros modelos

| Relación         | Modelo               | Descripción                                          |
| ---------------- | -------------------- | ---------------------------------------------------- |
| `ventas`         | `Sales`              | Cada cliente puede tener múltiples ventas asociadas. |
| `cuentas_cobrar` | `AccountsReceivable` | Si compra a crédito, se crean cuentas vinculadas.    |

### 🧾 Ejemplo de uso

```php

use Barkios\models\Clients;

$clientes = new Clients();

// Listar todos los clientes activos
$listado = $clientes->getAll();

// Agregar un nuevo cliente
$clientes->add('V12345678', 'Juan Pérez', 'Av. Principal #45', '0414-5550000', 'vip');

// Actualizar datos de cliente
$clientes->update('V12345678', 'Juan P. Pérez', 'Nueva dirección', '0414-5550001', 'regular');

// Eliminar cliente (inactivo)
$clientes->delete('V12345678');

// Buscar clientes VIP por nombre
$resultados = $clientes->searchVipClients('Ju');
```
