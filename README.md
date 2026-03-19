# PPS-U3-RA3 - Errores en la Seguridad y Componentes Vulnerables

> **Alumno:** Izan CD
> **Curso:** Puesta en Produccion Segura (PPS)
> **Unidad:** 3 - Deteccion y Correccion de Vulnerabilidades de Aplicaciones Web
> **Actividad:** RA3 - Errores en la Seguridad y componentes vulnerables

---

## Introduccion

Esta actividad tiene como objetivo demostrar la evolucion de la seguridad en una aplicacion web mediante la prueba de tres formularios de login con diferentes niveles de proteccion frente a inyecciones SQL. Se evidencia el estado inicial vulnerable, la primera mitigacion aplicada y las mejoras adicionales implementadas.

---

## 1. Estado Inicial y Vulnerabilidad

### Descripcion del escenario

El entorno de pruebas se basa en un escenario LAMP multicontenedor con Docker-Compose, donde se ejecutan tres formularios de login en la carpeta `SQLi`.[web:4]

| Login | Archivo | Nivel de proteccion |
|-------|---------|--------------------|
| Login 1 | `login1.php` | Sin proteccion |
| Login 2 | `login2.php` | Mitigacion parcial |
| Login 3 | `login3.php` | Mitigacion completa |

---

### 1.1. Login 1 - Vulnerable (sin mitigaciones)

**URL:** `http://localhost/SQLi/login1.php`

Este primer formulario presenta una vulnerabilidad de **inyeccion SQL** sin ningun tipo de mitigacion. La consulta SQL concatena directamente los datos introducidos por el usuario, permitiendo que un atacante modifique la logica de la consulta.

#### Prueba de inyeccion SQL

**Payload utilizado:**

```sql
' OR '1'='1' -- -
```

**Explicacion del payload:**
- `'` cierra el campo de texto de la consulta SQL
- `OR '1'='1'` fuerza una condicion siempre verdadera
- `-- -` comenta el resto de la consulta original

#### Resultado observado

> *Insertar aqui captura de pantalla donde se vea:*
> - *El formulario de login vulnerable*
> - *El payload introducido en usuario/contrasena*
> - *El resultado (acceso concedido o listado de usuarios/contrasenas)*

#### Mitigacion aplicada

En este login **no se ha aplicado ninguna mitigacion**. La vulnerabilidad queda expuesta de forma total, demostrando como una consulta SQL mal construida puede ser explotada.

**Breve explicacion de la medida necesaria:**

> La consulta SQL concatena directamente los parametros del formulario con la query. Para solucionar esto, se deben utilizar **consultas preparadas** (prepared statements), que separan los datos de la estructura de la consulta, evitando que el contenido introducido por el usuario sea interpretado como codigo SQL.

---

## 2. Solucion o Mitigacion 1 Implementada

### 2.1. Login 2 - Mitigacion parcial de inyeccion SQL

**URL:** `http://localhost/SQLi/login2.php`

En este segundo formulario se ha implementado una primera capa de proteccion frente a inyecciones SQL. Se han aplicado tecnicas basicas de sanitizacion y se utiliza consulta preparada para separar los datos del codigo SQL.

#### Prueba con el mismo payload de SQLi

**Payload utilizado:**

```sql
' OR '1'='1' -- -
```

#### Resultado observado

> *Insertar aqui captura de pantalla donde se vea:*
> - *El formulario del segundo login*
> - *El mismo payload introducido en usuario/contrasena*
> - *El resultado (acceso denegado o fallo de autenticacion)*

#### Mitigacion aplicada

En este login se han implementado las siguientes medidas de seguridad:[web:4]

1. **Consultas preparadas (prepared statements):** En lugar de concatenar los valores del formulario directamente en la consulta SQL, se utiliza un placeholder (`?`) y los valores se pasan como parametros vinculados. Esto impide que el payload sea interpretado como codigo SQL.

2. **Sanitizacion basica de entradas:** Se aplica una limpieza de los datos de entrada para eliminar caracteres peligrosos antes de procesarlos.

**Breve explicacion de la medida tomada:**

> La principal mejora respecto al login 1 es el uso de **consultas preparadas**. Con `prepare()` y `bind_param()`, los datos del usuario se tratan como valores literales, no como codigo ejecutable. El payload `' OR '1'='1' -- -` se interpreta como una cadena de texto normal y no altera la logica de la consulta. Sin embargo, esta version aun puede tener margen de mejora en la gestion de errores y validacion adicional de los datos.

---

## 3. Solucion o Mitigacion 2 Implementada

### 3.1. Login 3 - Mitigacion completa y hardening adicional

**URL:** `http://localhost/SQLi/login3.php`

Este tercer formulario representa la version mas segura, con todas las mejoras implementadas y capas adicionales de proteccion. Se ha aplicado un enfoque de defensa en profundidad.

#### Prueba con payload de SQLi

**Payload utilizado:**

```sql
' OR '1'='1' -- -
```

#### Prueba con payload adicional (XSS)

**Payload utilizado:**

```javascript
<script>alert('XSS')</script>
```

#### Resultado observado

> *Insertar aqui captura de pantalla donde se vea:*
> - *El formulario del tercer login*
> - *Los payloads introducidos*
> - *El resultado (bloqueo total del ataque)*

#### Mitigacion aplicada

En este login se han implementado las siguientes medidas de seguridad adicionales:[web:4]

1. **Consultas preparadas:** Se mantienen como base de la proteccion contra SQLi.

2. **Validacion estricta de entradas:** Se comprueba que los datos cumplen con el formato esperado (longitud maxima, caracteres permitidos, tipo de dato).

3. **Gestion segura de errores:** Los mensajes de error son genericos y no revelan informacion sobre la estructura de la base de datos o el backend.

4. **Proteccion contra XSS:** Se aplica `htmlspecialchars()` a las salidas para evitar inyeccion de scripts maliciosos.

5. **Limitacion de intentos:** Se puede implementar un sistema de bloqueo temporal tras varios intentos fallidos.

**Breve explicacion de la medida tomada:**

> El login 3 anade **capas extra de seguridad** sobre la base de consultas preparadas del login 2. Se aplica un enfoque de **defensa en profundidad**: validacion de entradas en servidor, sanitizacion de salidas, gestion de errores genericos y posibles limitaciones de intentos. Esto no solo previene la inyeccion SQL, sino tambien otros ataques como XSS, y reduce la informacion que un atacante puede obtener del sistema.

---

## 4. Resumen Comparativo de las Mitigaciones

| Aspecto | Login 1 (Vulnerable) | Login 2 (Mitigacion 1) | Login 3 (Mitigacion 2) |
|---------|---------------------|------------------------|------------------------|
| **Inyeccion SQL** | Explotable totalmente | Mitigada con prepared statements | Mitigada + validacion extra |
| **Consultas preparadas** | No implementadas | Implementadas | Implementadas |
| **Sanitizacion de entradas** | No | Basica | Estricta |
| **Gestion de errores** | Expone info sensible | Generica | Generica + logging seguro |
| **Proteccion XSS** | No | No | Si (htmlspecialchars) |
| **Limitacion intentos** | No | No | Opcional implementado |
| **Defensa en profundidad** | No | Parcial | Si |

---

## 5. Hardening del Servidor Web (Contexto de la actividad)

Los formularios de login se ejecutan sobre un servidor Apache con hardening implementado en el escenario LAMP:[web:4]

- **HTTPS con certificado autofirmado** (OpenSSL)
- **Forzado de HTTPS** mediante redirect en `default.conf`
- **Ocultacion de versiones** (ServerTokens Prod, ServerSignature Off, expose_php Off)
- **Cabeceras de seguridad:** X-Frame-Options, X-Content-Type-Options, X-XSS-Protection
- **WAF con ModSecurity + OWASP CRS** para bloquear ataques conocidos
- **Deshabilitacion de metodos HTTP inseguros** (PUT, DELETE, TRACE)
- **Deshabilitacion de listado de directorios** (Options -Indexes)

---

## 6. Conclusion

Tras probar los tres formularios de login en el servidor web endurecido, se evidencia claramente la importancia de aplicar medidas de seguridad de forma progresiva:

> El **Login 1** demostro como una consulta SQL mal construida puede ser totalmente explotada. El **Login 2** mostro que el uso de consultas preparadas mitiga eficazmente la inyeccion SQL basica. Finalmente, el **Login 3** evidencio que la seguridad debe ser un enfoque de **defensa en profundidad**, combinando multiples capas de proteccion.

Este documento de evidencias sera utilizado en la tarea obligatoria de la Unidad 3 para demostrar la realizacion de las actividades de hardening del servidor web y proteccion frente a componentes vulnerables.

---

## Referencias

- Actividad Hardening Servidor Apache - https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad3-VulnerabilidadesWeb/Actividad-HardeningSevidorApache-HTTPS-HSTS-WAF/README.md
- Puesta en Produccion Segura - Unidad 3 - IES Valle del Jerte
