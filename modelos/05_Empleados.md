# 👨‍💼 Modelo: Employees

**Ubicación:** `app/models/Employees.php`  
**Namespace:** `Barkios\models`  
**Extiende de:** `Barkios\core\Database`

---

## 📘 Descripción general

El modelo **Employees** gestiona toda la información relacionada con los empleados del sistema.  
Permite realizar operaciones **CRUD** (crear, leer, actualizar, eliminar lógicamente) sobre los registros de empleados activos, asegurando la integridad de los datos mediante validaciones previas y manejo de excepciones.

---

## 📂 Dependencias

- **PDO** → Para la ejecución de consultas SQL seguras y parametrizadas.  
- **Exception** → Control de errores y validaciones lógicas.  
- **Barkios\core\Database** → Clase base que maneja la conexión a la base de datos (`$this->db`).

---

## ⚙️ Métodos principales

### 🔹 `getAll()`
Obtiene todos los empleados activos del sistema ordenados por nombre.

**Consulta SQL:**
```sql
SELECT empleado_ced, nombre, telefono, cargo, fecha_ingreso
FROM empleados
WHERE activo = 1
ORDER BY nombre ASC
```

**Retorna:**
array con todos los registros activos de la tabla empleados.

**Manejo de errores:**
Si ocurre una excepción, se registra en el log del sistema (`error_log("Error en getAll: ...")`).

### 🔹 employeeExists($cedula)

Verifica si un empleado con una cédula específica existe en la base de datos.

**Parámetros:**

| Nombre    | Tipo   | Descripción                               |
| --------- | ------ | ----------------------------------------- |
| `$cedula` | string | Número de cédula del empleado a verificar |

```sql
SELECT COUNT(*) FROM empleados WHERE empleado_ced = :empleado_ced
```

**Retorna:**
`true` si existe, `false` en caso contrario.

🔹 getById($cedula)

Obtiene la información de un empleado activo por su cédula.

**Consulta SQL:**

```sql
SELECT empleado_ced, nombre, telefono, cargo, fecha_ingreso
FROM empleados
WHERE empleado_ced = :empleado_ced AND activo = 1
```

**Retorna:**

- `array` con los datos del empleado si existe.

- `null` si no se encuentra activo o no existe.

### 🔹 add($cedula, $nombre, $telefono, $cargo = 'Empleado')

Agrega un nuevo empleado al sistema con validación previa para evitar duplicados.

**Parámetros:**

| Nombre      | Tipo   | Descripción                               |
| ----------- | ------ | ----------------------------------------- |
| `$cedula`   | string | Cédula del empleado                       |
| `$nombre`   | string | Nombre completo                           |
| `$telefono` | string | Número de teléfono                        |
| `$cargo`    | string | Cargo asignado (por defecto `'Empleado'`) |

**Lógica interna:**

- Verifica si la cédula ya existe (employeeExists).

- Genera la fecha de ingreso (date('Y-m-d')).

- Inserta el nuevo registro.

**Consulta SQL:**
```sql
INSERT INTO empleados (empleado_ced, nombre, telefono, cargo, fecha_ingreso, activo)
VALUES (:empleado_ced, :nombre, :telefono, :cargo, :fecha_ingreso, 1)
```

**Retorna:**
- `true` si se inserta correctamente.
- Lanza excepción si la cédula ya existe.

### 🔹 update($cedula, $nombre, $telefono, $cargo = 'Empleado')

Actualiza los datos de un empleado existente.

**Parámetros:**

| Nombre      | Tipo   | Descripción                            |
| ----------- | ------ | -------------------------------------- |
| `$cedula`   | string | Cédula del empleado a modificar        |
| `$nombre`   | string | Nuevo nombre                           |
| `$telefono` | string | Nuevo número telefónico                |
| `$cargo`    | string | Nuevo cargo (por defecto `'Empleado'`) |

**Consulta SQL:**
```sql
UPDATE empleados
SET nombre = :nombre,
    telefono = :telefono,
    cargo = :cargo
WHERE empleado_ced = :empleado_ced
```

**Validación previa:**
Si el empleado no existe, lanza una excepción:

`"No existe un empleado con esta cédula"`

**Retorna:**
- `true` si la actualización fue exitosa.

### 🔹 delete($cedula)

Elimina lógicamente a un empleado estableciendo su campo `activo` en 0.

**Consulta SQL:**
```sql
UPDATE empleados
SET activo = 0
WHERE empleado_ced = :empleado_ced
```

**Parámetros:**

| Nombre    | Tipo   | Descripción                    |
| --------- | ------ | ------------------------------ |
| `$cedula` | string | Cédula del empleado a eliminar |

**Retorna:**
- `true` si se desactiva correctamente.

**Nota:**
No elimina físicamente el registro, solo lo marca como inactivo.

### 🔹 searchEmployees($query)

Realiza una búsqueda dinámica de empleados activos según coincidencias parciales en el nombre.

**Consulta SQL:**
```sql 
SELECT empleado_ced, nombre, telefono, cargo
FROM empleados
WHERE activo = 1
  AND nombre LIKE :query
ORDER BY nombre ASC
LIMIT 20
```

**Parámetros:**
| Nombre   | Tipo   | Descripción                         |
| -------- | ------ | ----------------------------------- |
| `$query` | string | Texto parcial a buscar en el nombre |

**Retorna:**
- `array` con los primeros 20 empleados que coincidan con la búsqueda.

**Manejo de errores:**
- Registra los errores en el log con el prefijo "Error en searchEmployees:".

### 🧱 Tablas SQL involucradas
| Tabla       | Campos principales                                                       | Descripción                                       |
| ----------- | ------------------------------------------------------------------------ | ------------------------------------------------- |
| `empleados` | `empleado_ced`, `nombre`, `telefono`, `cargo`, `fecha_ingreso`, `activo` | Contiene la información completa de cada empleado |

### 🧩 Relaciones con otros modelos
Aunque este modelo no tiene relaciones directas en el código actual, puede vincularse en el futuro con módulos como:

- Ventas: para asociar vendedores.

- Usuarios: si los empleados se gestionan como cuentas del sistema.

### 🧠 Consideraciones técnicas

- Todos los accesos a base de datos se hacen mediante **PDO** con consultas **parametrizadas**.

- Se implementa **borrado lógico** mediante el campo `activo`.

- Validación preventiva de duplicados mediante `employeeExists`.

- Manejo centralizado de errores con `error_log`.

- El modelo cumple con el principio de **Single Responsibility**, manejando solo lógica de empleados

### 🧾 Ejemplo de uso
```php
use Barkios\models\Employees;

$employees = new Employees();

// Obtener todos los empleados activos
$lista = $employees->getAll();

// Agregar un nuevo empleado
$employees->add('30399111', 'Carlos Ramírez', '04141234567', 'Vendedor');

// Actualizar datos
$employees->update('30399111', 'Carlos R. Ramírez', '04145556677', 'Supervisor');

// Buscar empleados
$resultado = $employees->searchEmployees('Carlos');

// Eliminar lógicamente
$employees->delete('30399111');
```