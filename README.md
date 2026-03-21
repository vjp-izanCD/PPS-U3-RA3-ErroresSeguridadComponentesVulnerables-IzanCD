# PPS-U3-RA3 – Errores de seguridad por componentes vulnerables

## 1. Iniciar entorno de pruebas

1. Situarte en la carpeta del entorno de pruebas LAMP (la que contiene los scripts y el `docker-compose.yml`).
2. Restaurar la configuración original del escenario:
   ```bash
   sudo ./restaurarConfiguracionOriginal.sh
   ```
3. Levantar el escenario multicontenedor:
   ```bash
   docker-compose up -d
   ```

En esta práctica **no** vamos a tocar principalmente archivos `.php` de la aplicación, sino los archivos de configuración del servidor:

- **Archivo de configuración global de Apache**: `/etc/apache2/apache2.conf`
  - Es el fichero principal de Apache.
  - No está en un volumen bind-mount, así que para modificarlo hay que entrar dentro del contenedor.
  - Asegúrate de tener solo un archivo de configuración válido; configuraciones duplicadas o incorrectas pueden impedir que el contenedor arranque.

- **Archivo de configuración de PHP**: `/usr/local/etc/php/php.ini`
  - En el escenario multicontenedor se expone por bind-mount en `./config/php/php.ini`, por lo que puedes editarlo directamente desde el anfitrión.

- **Configuración de sitios virtuales de Apache**: `/etc/apache2/sites-enabled/`
  - Tiene bind-mount con el directorio `./config/vhosts/` del anfitrión.
  - Podrás modificar o añadir configuraciones de sitios virtuales trabajando sobre `./config/vhosts/`.

En el último punto del enunciado original tienes una sección de "IMPORTANTE – Solución problemas que puedan surgir" donde se explican errores típicos al cambiar configuraciones; conviene leerla antes de tocar nada.

---

## 2. Apache en el escenario Docker

En este escenario utilizamos Docker Compose, así que para cambiar archivos de configuración:

- Si el archivo está en un volumen bind-mount (por ejemplo `./config`, `./logs`, `./config/vhosts`, `./config/php/php.ini`), lo editas desde tu máquina anfitriona.
- Si el archivo no está en bind-mount o prefieres cambiarlo dentro del contenedor, entra en el contenedor PHP/Apache:

```bash
docker exec -it lamp-php84 /bin/bash
```

El contenedor del servicio web se llama `lamp-php84`.
Si tu carpeta de proyecto no se llama `lamp`, el nombre del contenedor puede variar y tendrás que adaptarlo.

En una máquina Linux normal (sin docker-compose), Apache se instalaría con:

```bash
apt update
apt install apache2
```

Si no estás trabajando como `root`, añade `sudo` delante de los comandos.

---

## 3. Estructura de directorios de Apache

El directorio principal de configuración de Apache es `/etc/apache2`.
Dentro encontrarás, entre otros, esta estructura:

```text
/etc/apache2/
|-- apache2.conf
|       `-- ports.conf
|-- mods-enabled
|       |-- *.load
|       `-- *.conf
|-- conf-enabled
|       `-- *.conf
`-- sites-enabled
        `-- *.conf
`-- sites-available
        `-- *.conf
```

Puntos clave:

- `apache2.conf` es el archivo de configuración global.
- Los **módulos** añaden funcionalidades a Apache (por ejemplo, `ssl` para HTTPS).
  - `mods-available`: módulos disponibles.
  - `mods-enabled`: módulos habilitados (activos).
- Listar módulos cargados:
  ```bash
  apache2ctl -t -D DUMP_MODULES
  ```

**Gestión de módulos:**

- Habilitar un módulo:
  ```bash
  a2enmod nombre_modulo
  ```
  Ejemplo:
  ```bash
  a2enmod ssl
  ```

- Deshabilitar un módulo:
  ```bash
  a2dismod nombre_modulo
  ```

Internamente, `a2enmod` crea un enlace simbólico desde `mods-available` a `mods-enabled`; `a2dismod` elimina ese enlace.

**Gestión de sitios virtuales:**

- `sites-available`: configuraciones de sitios disponibles (pueden estar habilitados o no).
- `sites-enabled`: configuraciones de sitios que Apache tiene activos.
- Habilitar un sitio:
  ```bash
  a2ensite archivo.conf
  ```

---

## 4. Creación de sitio virtual `www.pps.edu`

Para crear un sitio virtual modificamos un archivo en `sites-available`.
En este caso, vamos a usar `default.conf` (o un fichero equivalente) con este contenido:

```conf
<VirtualHost *:80>

    ServerName www.pps.edu
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

Significado de las directivas más importantes:

- `ServerName`: nombre del host virtual (en este caso `www.pps.edu`).
- `ServerAdmin`: correo del administrador del sitio.
- `DocumentRoot`: carpeta donde se sirven los archivos del sitio (`/var/www/html`).
- `ErrorLog` y `CustomLog`: rutas de los logs, usando la variable `${APACHE_LOG_DIR}` (en el contenedor es `/var/log/apache2`).

En tu pila LAMP los ficheros de vhosts están bind-mount en `./config/vhosts/`, así que normalmente basta con editar ahí y recargar Apache, sin usar `a2ensite`.
Si lo hicieras "a la antigua" desde dentro del contenedor:

```bash
docker exec -it lamp-php84 /bin/bash
nano /etc/apache2/sites-available/default.conf
a2ensite /etc/apache2/sites-available/default.conf
service apache2 reload
```

### Permisos y propietarios del DocumentRoot

Apache atiende las peticiones con el usuario y grupo `www-data`.
Para que tenga acceso al contenido del sitio en `/var/www/html`, puedes ejecutar:

```bash
chown -R www-data:www-data /var/www/html/*
chmod -R 755 /var/www/html/*
```

**Importante**: estos cambios se aplican también al bind-mount `./www` en tu anfitrión, por lo que después tendrás que editar los archivos desde el contenedor (no desde el host).

---

## 5. Resolución local de nombres: `/etc/hosts`

Tu navegador resuelve direcciones como `www.google.com` consultando servidores DNS que asocian el nombre con su IP.
Para los sitios virtuales locales que no están en DNS públicos, editas el archivo `/etc/hosts` en tu máquina anfitriona para asociar los nombres con IPs locales.

En el archivo `/etc/hosts` añades:

```
127.0.0.1 pps.edu www.pps.edu
```

Como trabajamos con Docker y usamos redirección de puertos (el contenedor se expone en `http://localhost:80`), la IP `127.0.0.1` (bucle local) es perfecta.

Si quisieras acceder desde otros equipos de la red local o consultar la IP real del contenedor:

```bash
docker inspect lamp-php84 | grep IPAddress
```

Una vez añadida la línea en `/etc/hosts`, puedes acceder a:

```
http://www.pps.edu/
```

---

## 6. Creación de servidor virtual `www.hacker.edu`

Vamos a crear un segundo sitio virtual para alojar contenido de prueba (archivos "maliciosos" para simular vulnerabilidades).

Pasos:

1. Crear la estructura de directorios en el contenedor:
   ```bash
   docker exec -it lamp-php84 /bin/bash
   mkdir -p /var/www/hacker
   ```

2. Crear un archivo `index.html` básico en `/var/www/hacker/index.html`:
   ```html
   <html>
   <head><title>Hacker Site</title></head>
   <body>
   <h1>Welcome to Hacker Education Site</h1>
   <p>Este es un sitio de prueba para simulación de vulnerabilidades.</p>
   </body>
   </html>
   ```

3. Establecer permisos y propietarios:
   ```bash
   chown -R www-data:www-data /var/www/hacker
   chmod -R 755 /var/www/hacker
   ```

4. Crear el archivo de configuración del sitio virtual en `./config/vhosts/hacker.conf`:
   ```conf
   <VirtualHost *:80>
       ServerName www.hacker.edu
       ServerAdmin webmaster@localhost
       DocumentRoot /var/www/hacker

       ErrorLog ${APACHE_LOG_DIR}/error.log
       CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
   ```

5. Recargar Apache:
   ```bash
   service apache2 reload
   ```

6. Añadir el dominio a `/etc/hosts` de tu anfitrón:
   ```
   127.0.0.1 hacker hacker.edu www.hacker.edu
   ```

7. Acceder desde el navegador:
   ```
   http://www.hacker.edu/
   ```

---

## 7. Habilitar HTTPS con SSL/TLS en Apache

Para proteger las comunicaciones, habilitamos HTTPS en nuestro servidor.

### Paso 1: Generar certificado SSL autofirmado

Entra en el contenedor:

```bash
docker exec -it lamp-php84 /bin/bash
```

Crea el directorio de certificados y genera uno autofirmado válido por 365 días:

```bash
mkdir -p /etc/apache2/ssl
cd /etc/apache2/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout localhost.key -out localhost.crt
```

**Parámetros del comando:**
- `-x509`: crea un certificado autofirmado en lugar de una CSR.
- `-nodes`: omite cifrar la clave privada (sin contraseña).
- `-newkey rsa:2048`: genera clave RSA de 2048 bits.
- `-keyout`: archivo de la clave privada.
- `-out`: archivo del certificado.
- `-days 365`: validez del certificado.

Durante la ejecución se te pedirá información (país, organización, nombre común, etc.). Puedes presionar Enter para dejar los valores por defecto, pero en **Common Name (CN)** pon el nombre de tu servidor:

```
Common Name (e.g. server FQDN or YOUR name): pps.edu
```

### Paso 2: Configurar Apache para usar HTTPS

Modifica el archivo de configuración de tu sitio virtual. En tu pila LAMP, este archivo está en `./config/vhosts/default.conf` (bind-mount).

Edita `./config/vhosts/default.conf` en tu anfitrón:

```conf
<VirtualHost *:80>
    ServerName www.pps.edu
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName www.pps.edu

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/localhost.crt
    SSLCertificateKeyFile /etc/apache2/ssl/localhost.key

    DocumentRoot /var/www/html
</VirtualHost>
```

### Paso 3: Habilitar módulo SSL y recargar Apache

Desde el contenedor o anfitrón:

```bash
docker exec lamp-php84 /bin/bash -c "a2enmod ssl; service apache2 reload"
```

### Paso 4: Acceder por HTTPS

Ahora puedes acceder a:

```
https://www.pps.edu/
```

**Nota**: Como el certificado es autofirmado, el navegador avisará de que no es de confianza. Haz clic en "Avanzado" o "Continuar igualmente" para acceder al sitio.

---

## 8. Forzar HTTPS en Apache

Para forzar que todas las peticiones HTTP se redireccionen automáticamente a HTTPS, tienes dos opciones:

### Opción A: Redirect directo

Modifica `./config/vhosts/default.conf`:

```conf
<VirtualHost *:80>
    ServerName pps.edu
    ServerAlias www.pps.edu

    Redirect permanent / https://pps.edu/
</VirtualHost>

<VirtualHost *:443>
    ServerName pps.edu
    ServerAlias www.pps.edu
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/localhost.crt
    SSLCertificateKeyFile /etc/apache2/ssl/localhost.key
    SSLCertificateChainFile /etc/apache2/ssl/localhost.crt
</VirtualHost>
```

Recarga Apache:

```bash
docker exec lamp-php84 /bin/bash -c "service apache2 restart"
```

### Opción B: RewriteEngine (más flexible)

Modifica `./config/vhosts/default.conf`:

```conf
<VirtualHost *:80>
    ServerName www.pps.edu
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<VirtualHost *:443>
    ServerName www.pps.edu

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/localhost.crt
    SSLCertificateKeyFile /etc/apache2/ssl/localhost.key

    DocumentRoot /var/www/html
</VirtualHost>
```

Asegúrate de que el módulo `mod_rewrite` está habilitado:

```bash
docker exec lamp-php84 /bin/bash -c "a2enmod rewrite; service apache2 restart"
```

---

## 9. Alternativa: Forzar HTTPS con `.htaccess`

Si prefieres no modificar la configuración global de Apache, puedes usar un archivo `.htaccess` en la raiz del proyecto (`./www/.htaccess`):

```apache
RewriteEngine On

# Si no está usando HTTPS
RewriteCond %{HTTPS} !=on
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

**Requisito**: En tu `./config/vhosts/default.conf` debes permitir que se lean archivos `.htaccess`:

```conf
<VirtualHost *:80>
    ServerName www.pps.edu

    <Directory /var/www/html>
        AllowOverride All
    </Directory>

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName www.pps.edu

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/localhost.crt
    SSLCertificateKeyFile /etc/apache2/ssl/localhost.key

    DocumentRoot /var/www/html
</VirtualHost>
```

Asegúrate de que `mod_rewrite` está habilitado:

```bash
docker exec lamp-php84 /bin/bash -c "a2enmod rewrite; service apache2 restart"
```


---

## 10. Notas importantes

### Cambios en la configuración de Apache

- El archivo principal `/etc/apache2/apache2.conf` **no** está en bind-mount. Si lo modificas dentro del contenedor, no se puede editar desde tu anfitrón.
- Los archivos de `/etc/apache2/sites-enabled/` (vhosts) **sí** están en bind-mount en `./config/vhosts/`, por lo que puedes editarlos desde tu máquina.
- El archivo `/usr/local/etc/php/php.ini` está en bind-mount en `./config/php/php.ini`.

### Errores frecuentes

- **Contenedor no arranca**: Verifica que los archivos `.conf` sean válidos. Una sintaxis incorrecta rompe Apache.
- **"Acceso denegado"**: Asegúrate de que los permisos de `/var/www` son correctos (`chown -R www-data:www-data`, `chmod -R 755`).
- **Certificado no reconocido**: Es normal si usas certificados autofirmados. El navegador avisará, pero puedes continuar.
- **Redirección no funciona**: Comprueba que los módulos `ssl` y `rewrite` estén habilitados (`a2enmod ssl rewrite`).

### Comandos útiles

```bash
# Entrar en el contenedor
docker exec -it lamp-php84 /bin/bash

# Ver estado de Apache
service apache2 status

# Probar configuración de Apache
apache2ctl -t

# Ver módulos cargados
apache2ctl -M

# Ver logs de error
tail -f /var/log/apache2/error.log

# Recargar configuración
service apache2 reload

# Reiniciar Apache
service apache2 restart
```

---

## 11. Resumen de pasos rápido

1. **Iniciar escenario**: `sudo ./restaurarConfiguracionOriginal.sh` → `docker-compose up -d`
2. **Crear sitio `pps.edu`**: Añadir configuración en `./config/vhosts/default.conf`
3. **Resolver DNS**: Añadir `127.0.0.1 www.pps.edu` en `/etc/hosts`
4. **Crear sitio `hacker.edu`**: Crear directorio `/var/www/hacker`, configuración en `./config/vhosts/hacker.conf`
5. **Habilitar SSL**: Generar certificado con `openssl`, modificar vhosts para puertos 80 y 443
6. **Forzar HTTPS**: Usar `Redirect`, `RewriteEngine` o `.htaccess`
7. **Verificar**: Acceder a `http://www.pps.edu/` (redirige a HTTPS) y `http://www.hacker.edu/`

---

## 12. Leyendo el archivo de ejemplo

En el repositorio original encontrarás archivos de ejemplo comentados en GitHub:

- [`default.conf`](https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad3-VulnerabilidadesWeb/Actividad-HardeningSevidorApache-HTTPS-HSTS-WAF/files/000-default.conf): Configuración básica
- [`apache2.conf`](https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad3-VulnerabilidadesWeb/Actividad-HardeningSevidorApache-HTTPS-HSTS-WAF/files/apache2.conf.minimo): Configuración global mínima
- [`php.ini`](https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad3-VulnerabilidadesWeb/Actividad-HardeningSevidorApache-HTTPS-HSTS-WAF/files/php.ini): Configuración de PHP
- [`.htaccess`](https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad3-VulnerabilidadesWeb/Actividad-HardeningSevidorApache-HTTPS-HSTS-WAF/files/htaccess): Redirección con `.htaccess`

---

**¿Pregunta?** Consulta el archivo README original o revisita la documentación de Apache en [httpd.apache.org](https://httpd.apache.org).
