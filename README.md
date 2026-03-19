# PPS-U3-RA3 - Errores en la Seguridad y Componentes Vulnerables

En este documento se detallan los pasos realizados, comandos utilizados, payloads de inyeccion SQL, evidencias en forma de capturas de pantalla y explicaciones para completar la actividad de inyecciones SQL en tres formularios de login con diferentes niveles de proteccion.

---

**Alumno:** Izan CD

**Curso:** Puesta en Produccion Segura (PPS)

**Unidad:** 3 - Deteccion y Correccion de Vulnerabilidades de Aplicaciones Web

**Actividad:** RA3 - Errores en la Seguridad y componentes vulnerables

---
## 1. Preparacion del entorno

### 1.1. Arrancar entorno LAMP con Docker

Me sitúo en la carpeta del entorno de pruebas LAMP y restauro la configuracion original.

```
cd ~/ruta/a/mi/entorno/LAMP
sudo ./restaurarConfiguracionOriginal.sh
```

A continuación levanto el escenario multicontenedor.

```
docker-compose up -d
```

Compruebo que los contenedores están arriba:

```
docker ps
```

> Captura 1-T1: Contenedores del entorno LAMP funcionando correctamente.

### 1.2. Comprobacion de la base de datos SQLi

Acceso al contenedor de MySQL para verificar la base de datos.

```
docker exec -it lamp-mysql8 /bin/bash
```

Dentro del contenedor, me conecto a MySQL como root:

```
mysql -u root -p
```

Verifico que la base de datos SQLi y la tabla de usuarios existen:

```
SHOW DATABASES;
USE SQLi;
SELECT * FROM usuarios;
exit
```

Salgo del contenedor:

```
exit
```

> Captura 1-2-T1: Tabla `usuarios` en la BBDD `SQLi` mostrando los datos iniciales.

### 1.3. Verificacion de los archivos de login

Los tres formularios se encuentran en la carpeta `SQLi` del entorno LAMP:

| Login | Archivo | Nivel de proteccion |
|---|---|---|
| Login 1 | `login1.php` | Sin proteccion |
| Login 2 | `login2.php` | Mitigacion parcial |
| Login 3 | `login3.php` | Mitigacion completa |

Compruebo que los archivos existen:

```
ls -la ~/ruta/a/mi/entorno/LAMP/www/SQLi/
```

> Captura 1-3-T1: Listado de archivos en el directorio `SQLi`.
## 2. Login 1 - Formulario Vulnerable (sin mitigaciones)

### 2.1. Acceso al formulario

Abro el navegador y accedo a:

*   `http://localhost/SQLi/login1.php`

> Captura 2-1-T1: Formulario de login de `login1.php` mostrando los campos usuario y contraseña.

### 2.2. Prueba de autenticacion con credenciales correctas

Pruebo con las credenciales que conozco de la base de datos:

*   `username: admin`, `password: 1234`

> Captura 2-2-T1: Login exitoso en `login1.php` con credenciales reales.

### 2.3. Ataque de Inyeccion SQL

#### Payload utilizado

En el campo de usuario introduzco:

```
admin
```

Y en el campo de contraseña:

```
' OR '1'='1
```

#### Explicacion del payload

*   `'` → cierra el campo de texto de la consulta SQL original.
*   `OR '1'='1` → fuerza una condicion siempre verdadera en la consulta.
*   `--` (opcional) → comenta el resto de la consulta original para evitar errores de sintaxis.

La consulta final que ejecuta el servidor queda asi:

```sql
SELECT * FROM usuarios WHERE usuario = 'admin' AND contrasenya = '' OR '1'='1';
```

Como `'1'='1'` es siempre verdadero, el WHERE se cumple y devuelve todas las filas de la tabla.

> Captura 2-3-T1: Inyeccion SQL exitosa en `login1.php` mostrando acceso no autorizado.

### 2.4. Analisis del codigo vulnerable

El archivo `login1.php` construye la consulta concatenando directamente los valores del formulario:

```php
$query = "SELECT * FROM usuarios WHERE usuario = '" . $_REQUEST['username'] . 
         "' AND contrasenya = '" . $_REQUEST['password'] . "'";
$result = $conn->query($query);
```

Esto permite que cualquier entrada del usuario se interprete como parte de la consulta SQL.

### 2.5. Conclusion del Login 1

El formulario `login1.php` es completamente vulnerable a SQL Injection. Cualquier atacante puede:

*   Bypassear la autenticacion sin conocer credenciales reales.
*   Enumerar todos los usuarios de la base de datos.
*   Extraer informacion sensible mediante UNION-based SQL Injection.
## 3. Login 2 - Mitigacion Parcial de Inyeccion SQL

### 3.1. Acceso al formulario

Abro el navegador y accedo a:

*   `http://localhost/SQLi/login2.php`

> Captura 3-1-T1: Formulario de login de `login2.php` mostrando los campos usuario y contraseña.

### 3.2. Prueba con el mismo payload de SQLi

Pruebo el mismo payload que funciono en el login 1:

*   `username: admin`
*   `password: ' OR '1'='1`

> Captura 3-2-T1: Intento de SQL Injection en `login2.php` (resultado: ataque fallido o mensaje generico).

### 3.3. Mitigacion aplicada en Login 2

El archivo `login2.php` implementa **consultas preparadas (prepared statements)**:

```php
$query = "SELECT * FROM usuarios WHERE usuario = ? AND contrasenya = ?";
$stmt = $conn->prepare($query);
$stmt->bind_param("ss", $username, $password);
$stmt->execute();
$result = $stmt->get_result();
```

**Explicacion de la mejora:**

*   Se utilizan placeholders (`?`) en lugar de concatenar valores directamente.
*   `bind_param()` vincula los valores de las variables como parametros, no como codigo SQL.
*   El motor de base de datos trata los datos del usuario como valores literales, por lo que el payload `' OR '1'='1` se interpreta como una simple cadena de texto y no altera la logica de la consulta.

### 3.4. Pruebas adicionales

Pruebo con las credenciales correctas para verificar que el login funcional sigue trabajando:

*   `username: admin`, `password: 1234`

> Captura 3-4-T1: Login exitoso en `login2.php` con credenciales reales.

### 3.5. Conclusion del Login 2

El uso de consultas preparadas mitiga eficazmente la inyeccion SQL basica. Sin embargo, este login podria mejorarse con:

*   Validacion estricta de entradas (longitud maxima, caracteres permitidos).
*   Mensajes de error genericos que no revelen informacion del backend.
*   Hashing de contraseñas con `password_hash()` en lugar de texto plano.
*   Limitacion de intentos para evitar ataques de fuerza bruta.
## 4. Login 3 - Mitigacion Completa y Hardening Adicional

### 4.1. Acceso al formulario

Abro el navegador y accedo a:

*   `http://localhost/SQLi/login3.php`

> Captura 4-1-T1: Formulario de login de `login3.php` mostrando los campos usuario y contraseña.

### 4.2. Prueba con payload de SQLi

Pruebo nuevamente el payload de inyeccion SQL:

*   `username: admin`
*   `password: ' OR '1'='1`

> Captura 4-2-T1: Intento de SQL Injection en `login3.php` (resultado: ataque bloqueado totalmente).

### 4.3. Prueba con payload de XSS

Pruebo un payload adicional para verificar proteccion contra XSS:

*   `username: <script>alert('XSS')</script>`
*   `password: password123`

> Captura 4-3-T1: Intento de XSS en `login3.php` (resultado: codigo sanitized, alerta no se ejecuta).

### 4.4. Mitigaciones aplicadas en Login 3

El archivo `login3.php` implementa un enfoque de **defensa en profundidad**:

#### 1. Consultas preparadas (base)
```php
$query = "SELECT contrasenya FROM usuarios WHERE usuario = ?";
$stmt = $conn->prepare($query);
$stmt->bind_param("s", $username);
```

#### 2. Validacion estricta de entradas
```php
$username = trim($_REQUEST['username']);
$password = trim($_REQUEST['password']);

if (strlen($username) < 3 || strlen($username) > 50) {
    echo "Formato de usuario invalido.";
    exit;
}

if (!preg_match('/^[a-zA-Z0-9_]+$/', $username)) {
    echo "Caracteres no permitidos en el nombre de usuario.";
    exit;
}
```

#### 3. Hashing de contraseñas
```php
$hashed_password = $row['contrasenya'];
if (password_verify($password, $hashed_password)) {
    // Login exitoso
} else {
    // Login fallido
}
```

#### 4. Gestion segura de errores
```php
error_reporting(0);
ini_set('display_errors', 0);

// Mensaje generico sin exponer detalles
$message = "Usuario o contraseña incorrectos.";
```

#### 5. Proteccion contra XSS
```php
$username = htmlspecialchars($username, ENT_QUOTES, 'UTF-8');
$username = strip_tags($username);
```

#### 6. Limitacion de intentos (opcional)
```php
// Comprobar intentos fallidos
if ($failed_attempts >= 3 && $time_since_last_attempt < 900) {
    echo "Cuenta temporalmente bloqueada. Intente de nuevo en " . 
         ($remaining_seconds / 60) . " minutos.";
    exit;
}
```

### 4.5. Pruebas finales

*   Login con credenciales correctas → Acceso exitoso.
*   Login con credenciales incorrectas → Mensaje generico de error.
*   Payload SQLi → Bloqueado completamente.
*   Payload XSS → Sanitizado, no se ejecuta.
*   Múltiples intentos fallidos → Posible bloqueo temporal.

> Captura 4-5-T1: Login exitoso en `login3.php` con credenciales reales y validacion funcionando.

### 4.6. Conclusion del Login 3

El formulario `login3.php` representa la version mas segura con múltiples capas de proteccion combinadas. El enfoque de defensa en profundidad asegura que incluso si una capa de seguridad falla, las demas continuan protegiendo la aplicacion.
## 5. Hardening del Servidor Web (Contexto de la actividad)

Los formularios de login se ejecutan sobre un servidor Apache con hardening implementado en el escenario LAMP:

*   **HTTPS con certificado autofirmado** → Configuracion mediante OpenSSL.
*   **Forzado de HTTPS** → Redireccionamiento 301 en `default.conf`.
*   **Ocultacion de versiones** → `ServerTokens Prod`, `ServerSignature Off`, `expose_php Off`.
*   **Cabeceras de seguridad** → `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`.
*   **WAF con ModSecurity + OWASP CRS** → Bloqueo automatico de ataques conocidos como SQL Injection.
*   **Deshabilitacion de metodos HTTP inseguros** → PUT, DELETE, TRACE deshabilitados.
*   **Deshabilitacion de listado de directorios** → `Options -Indexes`.

> Captura 5-1-T1: Cabeceras de seguridad HTTP del servidor Apache verificadas con las herramientas del navegador.

## 6. Resumen Comparativo de Mitigaciones

| Aspecto | Login 1 (Vulnerable) | Login 2 (Mitigacion 1) | Login 3 (Mitigacion 2) |
|---|---|---|---|
| **Inyeccion SQL** | Explotable totalmente | Mitigada con prepared statements | Mitigada + validacion extra |
| **Consultas preparadas** | No implementadas | Implementadas | Implementadas |
| **Sanitizacion de entradas** | No | Basica | Estricta |
| **Gestion de errores** | Expone informacion sensible | Generica | Generica + logging seguro |
| **Proteccion XSS** | No | No | Si (`htmlspecialchars`) |
| **Hashing de contraseñas** | Texto plano | Texto plano | Hasheadas (`password_hash`) |
| **Limitacion de intentos** | No | No | Opcional implementado |
| **Defensa en profundidad** | No | Parcial | Si |

## 7. Conclusiones

Tras probar los tres formularios de login en el servidor web endurecido, se evidencia claramente la importancia de aplicar medidas de seguridad de forma progresiva:

1.  **Login 1** demostro como una consulta SQL mal construida puede ser totalmente explotada. Un simple payload como `' OR '1'='1` es suficiente para bypassar la autenticacion.
2.  **Login 2** mostro que el uso de consultas preparadas mitiga eficazmente la inyeccion SQL basica, al tratar los datos del usuario como valores literales y no como codigo ejecutable.
3.  **Login 3** evidencio que la seguridad debe ser un enfoque de **defensa en profundidad**, combinando multiples capas de proteccion (validacion de entradas, hashing, sanitizacion de salidas, gestion de errores genericos, y limitacion de intentos).

El hardening del servidor Apache complementa estas medidas a nivel de aplicacion, proporcionando proteccion adicional mediante el WAF, HTTPS obligatorio y ocultacion de informacion sensible del servidor.

## 8. Guardar cambios y apagar el entorno

Una vez completada la actividad, guardo la configuracion y restauro el entorno.

```
sudo ./guardarConfiguraciones.sh SQLi
sudo ./restaurarConfiguracionOriginal.sh
```

Los archivos de evidencias quedan en la carpeta del entorno LAMP bajo `./www/SQLi/`.

Si no voy a seguir trabajando con el entorno, detengo los contenedores:

```
docker-compose down
```

## Referencias

*   Actividad Hardening Servidor Apache - [github.com/jmmedinac03vjp/PuestaProduccionSegura](https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad3-VulnerabilidadesWeb/Actividad-HardeningSevidorApache-HTTPS-HSTS-WAF/README.md)
*   Puesta en Produccion Segura - Unidad 3 - IES Valle del Jerte
*   OWASP SQL Injection - [owasp.org/www-community/attacks/SQL_Injection](https://owasp.org/www-community/attacks/SQL_Injection)
*   OWASP Prepared Statements - [cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
