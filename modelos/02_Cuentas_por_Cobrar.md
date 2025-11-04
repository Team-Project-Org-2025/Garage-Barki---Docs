# 🧾 Modelo: AccountsReceivable

**Ubicación:** `app/models/AccountsReceivable.php`  
**Namespace:** `Barkios\models`  
**Extiende de:** `Barkios\core\Database`

---

## 📘 Descripción general

El modelo **AccountsReceivable** gestiona toda la lógica relacionada con las **cuentas por cobrar** del sistema.  
Su objetivo es controlar los créditos otorgados a clientes, registrar pagos, manejar vencimientos y actualizar los estados de las ventas relacionadas.

Incluye consultas avanzadas, validaciones de negocio y control de transacciones SQL para garantizar la integridad de los datos.

---

## 📂 Dependencias

- **PDO** → Manejo de base de datos.
- **Exception** → Manejo de errores.
- **Barkios\core\Database** → Clase base de conexión y gestión de consultas.

---

## ⚙️ Métodos Principales

### 🔹 `getAll()`
Obtiene todas las cuentas por cobrar activas (no eliminadas ni anuladas).

**Retorna:**  
`array` — Lista de cuentas con datos de cliente, venta y crédito asociados.

**Características:**
- Calcula días restantes (`DATEDIFF`).
- Determina el estado visual: `Pagado`, `Vencido`, `Por vencer`, `Vigente`.
- Ordena por fecha de vencimiento.

---

### 🔹 `getById($cuentaId)`
Obtiene los datos completos de una cuenta específica.

**Parámetros:**
- `$cuentaId` *(int)* — ID de la cuenta por cobrar.

**Incluye:**
- Detalles de venta, cliente, empleado y crédito.
- Pagos asociados (`getPaymentsByAccount`).
- Total pagado acumulado.

**Retorna:**  
`array|null` — Datos de la cuenta o `null` si no existe.

---

### 🔹 `getByClient($clienteCed)`
Devuelve todas las cuentas por cobrar pertenecientes a un cliente específico.

**Parámetros:**
- `$clienteCed` *(string)* — Cédula o identificación del cliente.

**Retorna:**  
`array` — Listado de cuentas pendientes, vencidas o pagadas.

---

### 🔹 `getPaymentsByAccount($cuentaId)`
Obtiene todos los **pagos confirmados** asociados a una cuenta.

**Parámetros:**
- `$cuentaId` *(int)* — ID de la cuenta por cobrar.

**Retorna:**  
`array` — Pagos confirmados ordenados por fecha.

---

### 🔹 `registerPayment($data)`
Registra un nuevo pago sobre una cuenta pendiente.

**Parámetros esperados:**
- `cuenta_cobrar_id`
- `monto`
- `tipo_pago`
- `referencia_bancaria`
- `banco`
- `observaciones`

**Proceso:**
1. Valida existencia de la cuenta.
2. Verifica que no esté pagada o vencida.
3. Inserta el pago confirmado.
4. Actualiza el saldo pendiente y estado de venta.
5. Marca la cuenta como “pagada” si el saldo llega a 0.

**Transacción SQL:** ✅ (usa `beginTransaction`, `commit`, `rollback`)

**Retorna:**
```php
[
  'success' => bool,
  'message' => string,
  'pago_id' => int|null,
  'nuevo_saldo' => float|null
]
```

### 🔹 updateDueDate($cuentaId, $nuevaFecha)

Actualiza la fecha de vencimiento de una cuenta.

**Reglas:**

- La nueva fecha debe ser futura.

- Solo permite actualizar cuentas pendientes o vencidas.

**Efectos secundarios:**

- Si estaba “vencida”, cambia su estado a “pendiente”.

### 🔹 processExpiredAccounts()

Procesa automáticamente las cuentas vencidas y actualiza estados globales.

**Acciones:**

1. Cambia el estado de las cuentas a vencido.

2. Cancela ventas asociadas con estado pendiente.

3. Libera prendas relacionadas (p.estado = 'DISPONIBLE').

4. Devuelve cantidad total de cuentas afectadas.

### 🔹 delete($cuentaId)

Elimina (lógicamente) una cuenta por cobrar.

**Reglas:**

- No puede eliminarse una cuenta pagada.

- Marca la cuenta como eliminado.

- Cancela la venta relacionada.

- Libera los artículos vendidos.

**Retorna:**
```php
[
  'success' => bool,
  'message' => string
]
```

### 🔹 getStats()

Obtiene estadísticas generales del módulo de cuentas por cobrar.

**Datos devueltos:**

- Total de cuentas

- Cantidad pendientes, pagadas y vencidas

- Saldo total pendiente

- Monto por vencer (en los próximos 3 días)

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

### 🧠 Consideraciones técnicas

- Todas las operaciones críticas están protegidas por transacciones.

- Los estados posibles de una cuenta:

- `pendiente`

- `pagado`

- `vencido`

- `eliminado`

- Los estados de venta relacionados se sincronizan automáticamente.

### Relaciones con otros modelos

| Relación	| Tabla / Modelo	| Descripción |
| --- | --- | --- |
| `ventas`	| `Sale`	| Cada cuenta pertenece a una venta a crédito.
| `credito`	| `Credit`	| Relación directa entre cuenta y crédito.
| `clientes`	| `Client`	| Cliente deudor asociado.
| `pagos`	| `Payment`	| Historial de pagos confirmados.
| `prendas`	| `Product`	| Productos vendidos que pueden liberarse si la cuenta se elimina o vence.

### 🧾 Ejemplo de uso

```php 
use Barkios\models\AccountsReceivable;

$cuentas = new AccountsReceivable();

// Listar todas las cuentas activas
$listado = $cuentas->getAll();

// Registrar un pago
$pago = $cuentas->registerPayment([
    'cuenta_cobrar_id' => 12,
    'monto' => 150.00,
    'tipo_pago' => 'TRANSFERENCIA',
    'banco' => 'Banco Nacional',
    'referencia_bancaria' => 'TRX12345'
]);
```