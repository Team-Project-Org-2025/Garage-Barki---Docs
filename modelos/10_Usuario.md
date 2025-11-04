# Modelo User (Usuario)

## Descripción
Modelo para la gestión de usuarios: autenticación, CRUD (crear, leer, actualizar, eliminar), verificación de existencia y validación de contraseña fuerte.

---

## Métodos

### 🔹 __construct()
Constructor que llama al constructor de la clase base `Database`.

- **Parámetros**: ninguno
- **Retorna**: void

---

### 🔹 getLastInsertId(): ?int
Obtiene el último ID insertado en la base de datos.

- **Parámetros**: ninguno
- **Retorna**: int|null (ID o null en caso de error)

---

### 🔹 authenticate(string $email, string $password)
Autentica un usuario mediante email y contraseña.

- **Parámetros**:
  - `$email` (string): Email del usuario.
  - `$password` (string): Contraseña en texto plano.
- **Retorna**: array con datos del usuario (sin contraseña) o null si falla autenticación.
- **Detalles**:
  - Verifica la contraseña con `password_verify`.
  - Re-hashea la contraseña si el algoritmo cambió (`password_needs_rehash`).
- **Consulta SQL**:
```sql
SELECT id, email, password_hash, nombre FROM users WHERE email = :email;
```

### 🔹 private updatePasswordHash(int $userId, string $plainPassword): void

Actualiza el hash de la contraseña de un usuario.

- **Parámetros:**

- `$userId` (int): ID del usuario.

- `$plainPassword` (string): Contraseña en texto plano.

**Retorna**: void

**Consulta SQL**:
```sql
UPDATE users SET password_hash = :hash WHERE id = :id;
```

### 🔹 getAll()

Obtiene todos los usuarios registrados.

- **Parámetros**: ninguno

**Retorna**: array con usuarios (id, email, nombre)

**Consulta SQL**:
```sql
SELECT id, email, nombre FROM users ORDER BY id ASC;
```

### 🔹 userExists(int $id = null, string $email = null): bool

Verifica si un usuario existe por ID o por email.

- **Parámetros:**

- `$id` (int|null): ID del usuario (opcional).

- `$email` (string|null): Email del usuario (opcional).

**Retorna**: bool (true si existe)

**Consulta SQL**:
```sql
-- Si $id está definido:
SELECT COUNT(*) FROM users WHERE id = :id;

-- Si $email está definido:
SELECT COUNT(*) FROM users WHERE email = :email;
```

### 🔹 getById(int $id)

Obtiene un usuario por su ID.

- **Parámetros:**

- `$id` (int): ID del usuario.

**Retorna**: array con datos del usuario o null si no existe

**Consulta SQL**:
```sql
SELECT id, email, nombre FROM users WHERE id = :id;
```

### 🔹 add(string $nombre, string $email, string $password)

Agrega un nuevo usuario si no existe otro con el mismo email.

- **Parámetros:**

- `$nombre` (string)

- `$email` (string)

- `$password` (string) contraseña en texto plano

**Retorna**: bool (true si se insertó)

**Excepciones**: lanza Exception si ya existe un usuario con el email o si falla el hash de la contraseña.

**Consulta SQL**:
```sql
INSERT INTO users (nombre, email, password_hash)
VALUES (:nombre, :email, :password_hash);
```

### 🔹 update(int $id, string $nombre, string $email, string $password = null)

Actualiza los datos de un usuario. Si se pasa contraseña, también la actualiza.

- **Parámetros:**

- `$id` (int)

- `$nombre` (string)

- `$email` (string)

- `$password` (string|null) opcional, si se incluye se actualiza el hash

**Retorna**: bool (true si se actualizó)

**Excepciones**: lanza Exception si no existe el usuario o falla el hash.

**Consulta SQL**:
```sql
UPDATE users
SET nombre = :nombre,
    email = :email,
    [password_hash = :password_hash (si aplica)]
WHERE id = :id;
```

### 🔹 delete(int $id)

Elimina un usuario por su ID.

- **Parámetros:**

- `$id` (int)

**Retorna**: bool (true si se eliminó)

**Consulta SQL**:
```sql
DELETE FROM users WHERE id = :id;
```

### 🔹 isPasswordStrong(string $password): bool

Valida si la contraseña cumple con requisitos de seguridad:

- Al menos 8 caracteres.

- Al menos una letra minúscula.

- Al menos una letra mayúscula.

- Al menos un número.

- Al menos un símbolo especial (@$!%*?&._-)

- **Parámetros:**

- `$password` (string)

**Retorna**: bool (true si es fuerte)