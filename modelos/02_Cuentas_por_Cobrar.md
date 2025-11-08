# 🧾 Modelo: AccountsReceivable

**Ubicación:** `app/models/AccountsReceivable.php`  
**Namespace:** `Barkios\models`  
**Extiende de:** `Barkios\core\Database`

---

## 📘 Descripción general

El modelo **AccountsReceivable** centraliza toda la gestión de las **cuentas por cobrar** del sistema.  
Su función principal es administrar créditos, registrar pagos, controlar vencimientos y sincronizar automáticamente los estados de las ventas asociadas.

Implementa control transaccional, validaciones de negocio y métodos reutilizables para optimizar el manejo de datos relacionados con los créditos y pagos.

---

## 📂 Dependencias

- **PDO** → Ejecución y manipulación de consultas SQL.
    
- **Exception** → Captura y manejo de errores en las operaciones.
    
- **Barkios\core\Database** → Clase base que gestiona la conexión y los métodos de base de datos.
    

---

## ⚙️ Métodos principales

### 🔹 `getAll()`

Obtiene todas las **cuentas por cobrar activas**, excluyendo las eliminadas o anuladas.

**Retorna:**  
`array` — Lista de cuentas con información combinada del cliente, venta y crédito.

**Incluye:**

- Días restantes para el vencimiento (`DATEDIFF`).
    
- Estado visual (`Pagado`, `Vencido`, `Por vencer`, `Vigente`).
    
- Orden automático por fecha de vencimiento y prioridad de cuenta.
    

---

### 🔹 `getById(int $id)`

Obtiene toda la información de una cuenta específica.

**Parámetros:**

- `$id` _(int)_ — ID único de la cuenta por cobrar.
    

**Incluye:**

- Datos de venta, cliente, crédito y empleado.
    
- Pagos asociados (`getPaymentsByAccount`).
    
- Total pagado acumulado automáticamente.
    

**Retorna:**  
`array|null` — Datos completos de la cuenta o `null` si no existe.

---

### 🔹 `getByClient(string $cedula)`

Obtiene todas las cuentas por cobrar vinculadas a un cliente determinado.

**Parámetros:**

- `$cedula` _(string)_ — Cédula o identificación del cliente.
    

**Retorna:**  
`array` — Listado de cuentas (pendientes, vencidas o pagadas).

---

### 🔹 `getPaymentsByAccount(int $id)`

Devuelve todos los **pagos confirmados** asociados a una cuenta específica.

**Parámetros:**

- `$id` _(int)_ — ID de la cuenta por cobrar.
    

**Retorna:**  
`array` — Lista de pagos confirmados ordenados por fecha (descendente).

---

### 🔹 `registerPayment(array $data)`

Registra un **nuevo pago** sobre una cuenta pendiente y actualiza los saldos correspondientes.

**Parámetros esperados:**

|Clave|Descripción|
|---|---|
|`cuenta_cobrar_id`|ID de la cuenta a la que se aplica el pago|
|`monto`|Monto a registrar|
|`tipo_pago`|Tipo de pago (ej. `EFECTIVO`, `TRANSFERENCIA`)|
|`referencia_bancaria`|Código de referencia bancaria|
|`banco`|Nombre del banco (opcional)|
|`observaciones`|Comentarios o notas adicionales|

**Proceso:**

1. Valida que la cuenta exista y no esté pagada o vencida.
    
2. Verifica que el monto sea válido y no supere el saldo pendiente.
    
3. Inserta el pago con estado `CONFIRMADO`.
    
4. Actualiza el saldo y el estado de la venta asociada.
    
5. Marca la cuenta como **pagada** si el saldo llega a cero.
    

**Transacción SQL:** ✅ Controlada mediante `beginTransaction`, `commit` y `rollBack`.

**Retorna:**

```php
[
  'success' => bool,
  'message' => string,
  'nuevo_saldo' => float|null
]
```

---

### 🔹 `updateDueDate(int $id, string $nuevaFecha)`

Actualiza la **fecha de vencimiento** de una cuenta.

**Parámetros:**

- `$id` _(int)_ — ID de la cuenta por cobrar.
    
- `$nuevaFecha` _(string)_ — Nueva fecha en formato `YYYY-MM-DD`.
    

**Reglas:**

- La fecha debe ser **posterior a la actual**.
    
- Solo puede modificarse si la cuenta está `pendiente` o `vencida`.
    
- Si estaba vencida, su estado pasa automáticamente a `pendiente`.
    

**Retorna:**

```php
[
  'success' => bool,
  'message' => string
]
```

---

### 🔹 `processExpiredAccounts()`

Ejecuta un proceso automatizado que **detecta y actualiza las cuentas vencidas**.

**Acciones realizadas:**

1. Marca como `vencido` todas las cuentas cuya fecha de vencimiento ha pasado.
    
2. Cancela las ventas asociadas con estado `pendiente`.
    
3. Libera las prendas asociadas a las ventas canceladas.
    
4. Devuelve la cantidad total de cuentas procesadas.
    

**Transacción SQL:** ✅

**Retorna:**

```php
[
  'success' => bool,
  'message' => string,
  'affected' => int
]
```

---

### 🔹 `getStats()`

Obtiene estadísticas globales del módulo de cuentas por cobrar.

**Datos devueltos:**

|Campo|Descripción|
|---|---|
|`total_cuentas`|Total general de cuentas registradas|
|`pendientes`|Cantidad de cuentas pendientes|
|`pagadas`|Cantidad de cuentas pagadas|
|`vencidas`|Cantidad de cuentas vencidas|
|`saldo_total`|Suma total de los saldos pendientes|
|`por_vencer`|Monto total de cuentas que vencen en ≤ 3 días|

**Ejemplo de retorno:**

```php
[
  "total_cuentas" => 120,
  "pendientes" => 45,
  "pagadas" => 65,
  "vencidas" => 10,
  "saldo_total" => 54321.75,
  "por_vencer" => 3200.00
]
```

---

## 🧠 Consideraciones técnicas

- Todos los métodos críticos se ejecutan dentro de **transacciones atómicas**.
    
- Se implementaron métodos genéricos (`run()` y `execute()`) para reducir duplicación.
    
- El modelo aplica los estados lógicos:
    
    - `pendiente`
        
    - `pagado`
        
    - `vencido`
        
    - `eliminado`
        
- Los estados de **ventas** se sincronizan automáticamente con los de la cuenta.
    
- Compatible con **PHP 8.1+** y buenas prácticas de manejo de excepciones.
    

---

## 🔗 Relaciones con otros modelos

|Relación|Tabla / Modelo|Descripción|
|---|---|---|
|`ventas`|`Sale`|Cada cuenta pertenece a una venta a crédito.|
|`credito`|`Credit`|Relación directa entre cuenta y crédito.|
|`clientes`|`Client`|Cliente deudor asociado a la cuenta.|
|`pagos`|`Payment`|Historial de pagos confirmados.|
|`prendas`|`Product`|Productos liberados si la cuenta vence o se cancela.|

---

## 🧾 Ejemplo de uso

```php
use Barkios\models\AccountsReceivable;

$cuentas = new AccountsReceivable();

// Listar todas las cuentas activas
$listado = $cuentas->getAll();

// Consultar una cuenta específica
$detalle = $cuentas->getById(10);

// Registrar un nuevo pago
$resultado = $cuentas->registerPayment([
    'cuenta_cobrar_id' => 12,
    'monto' => 150.00,
    'tipo_pago' => 'TRANSFERENCIA',
    'banco' => 'Banco Nacional',
    'referencia_bancaria' => 'TRX12345',
    'observaciones' => 'Pago parcial'
]);

// Actualizar fecha de vencimiento
$cuentas->updateDueDate(12, '2025-12-01');
```