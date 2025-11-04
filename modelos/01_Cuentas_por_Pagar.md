# Modelo: AccountsPayable

## 📋 Descripción
El modelo **AccountsPayable** gestiona las operaciones relacionadas con las **cuentas por pagar a proveedores** dentro del sistema **Barkios**.  
Permite obtener registros, registrar pagos, anular abonos, actualizar estados de cuentas y generar estadísticas financieras sobre las deudas.

---

## 🧩 Estructura general
**Namespace:** `Barkios\models`  
**Hereda de:** `Barkios\core\Database`  
**Archivo:** `models/AccountsPayable.php`

---

## 🧠 Tablas involucradas
- `cuentas_pagar` — Registro principal de las cuentas pendientes por pagar.  
- `compras` — Contiene las compras asociadas a las cuentas.  
- `proveedores` — Información del proveedor relacionado.  
- `pagos_compras` — Pagos registrados por cada cuenta por pagar.

---

## 🧾 Campos principales
| Campo | Tipo | Descripción |
|--------|------|-------------|
| cuenta_pagar_id | INT | Identificador único de la cuenta. |
| compra_id | INT | ID de la compra asociada. |
| proveedor_rif | VARCHAR | RIF del proveedor asociado. |
| monto | DECIMAL | Monto total de la cuenta por pagar. |
| fecha_vencimiento | DATE | Fecha límite de pago. |
| estado | ENUM('pendiente', 'vencido', 'pagado') | Estado actual de la cuenta. |
| fec_actualizacion | TIMESTAMP | Última modificación registrada. |

---

## ⚙️ Métodos públicos

### 🔹 `getAll()`
Obtiene todas las cuentas por pagar con información de la compra, proveedor y pagos realizados.  
Actualiza automáticamente los estados a **vencido** o **pagado** según el saldo restante y la fecha de vencimiento.

**Retorna:**  
`array` — Lista de cuentas con los siguientes campos calculados:
- `total_pagado`: total abonado con pagos confirmados.  
- `saldo_pendiente`: monto restante por pagar.  
- `vencida`: bandera booleana (1 si está vencida, 0 si no).  
- `nombre_proveedor`, `nombre_contacto`, `factura_numero`, etc.

---

### 🔹 `getById($id)`
Obtiene una cuenta específica con los datos detallados de la compra y del proveedor.

**Parámetros:**
- `$id (int)` — ID de la cuenta a consultar.

**Retorna:**  
`array|null` — Información de la cuenta o `null` si no existe.

---

### 🔹 `getPagosByCuentaId($cuentaId)`
Obtiene todos los pagos asociados a una cuenta por pagar, excluyendo los anulados.

**Parámetros:**
- `$cuentaId (int)` — ID de la cuenta por pagar.

**Retorna:**  
`array` — Pagos registrados, ordenados por `fecha_pago DESC`.

---

### 🔹 `addPago($datos)`
Registra un nuevo pago o abono para una cuenta por pagar.  
Si el pago cubre el monto total, el estado de la cuenta cambia automáticamente a **pagado**.

**Parámetros:**
- `$datos (array)` con las claves:
  - `cuenta_pagar_id`
  - `compra_id`
  - `fecha_pago`
  - `monto`
  - `tipo_pago`
  - `moneda_pago`
  - `referencia_bancaria` *(opcional)*
  - `banco` *(opcional)*
  - `estado_pago` *(por defecto `CONFIRMADO`)*
  - `observaciones` *(opcional)*

**Retorna:**  
`int` — ID del pago registrado.

**Excepciones:**  
Lanza `Exception` si ocurre un error durante la transacción.

---

### 🔹 `updateEstado($cuentaId, $nuevoEstado)`
Actualiza el estado de una cuenta por pagar.

**Parámetros:**
- `$cuentaId (int)`
- `$nuevoEstado (string)` — Puede ser `pendiente`, `vencido` o `pagado`.

**Retorna:**  
`bool` — `true` si se actualizó correctamente.

---

### 🔹 `anularPago($pagoId)`
Anula un pago previamente registrado y actualiza el estado de la cuenta según corresponda.  
Si el saldo resultante vuelve a ser mayor que cero, la cuenta se marca como **pendiente** o **vencido** dependiendo de la fecha de vencimiento.

**Parámetros:**
- `$pagoId (int)` — ID del pago a anular.

**Retorna:**  
`bool` — `true` si se anula exitosamente.

**Excepciones:**  
Lanza `Exception` si el pago no existe o si falla la transacción.

---

### 🔹 `getEstadisticas()`
Obtiene métricas generales del módulo de cuentas por pagar.

**Retorna:**  
`array` con los siguientes campos:
| Campo | Descripción |
|--------|-------------|
| total_cuentas | Total de cuentas registradas. |
| deuda_original | Suma total de los montos iniciales. |
| deuda_pendiente | Suma total del saldo aún no pagado. |
| cuentas_vencidas | Número de cuentas en estado “vencido”. |
| por_vencer_7dias | Cuentas que vencerán en los próximos 7 días. |

---

## 🧮 Ejemplo de uso

```php
use Barkios\models\AccountsPayable;

$cuentas = new AccountsPayable();

// Obtener todas las cuentas
$listado = $cuentas->getAll();

// Registrar un pago
$pagoId = $cuentas->addPago([
    'cuenta_pagar_id' => 1,
    'compra_id' => 12,
    'fecha_pago' => '2025-11-04',
    'monto' => 150.00,
    'tipo_pago' => 'Transferencia',
    'moneda_pago' => 'VES',
]);

// Consultar estadísticas
$estadisticas = $cuentas->getEstadisticas();
