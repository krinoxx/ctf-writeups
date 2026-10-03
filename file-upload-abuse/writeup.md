# File Upload Vulnerability Scenarios – Multi-Bypass — file_upload_vulnerability_scenarios
**Dificultad:** Media | **SO:** Linux (contenedor Docker, PHP/Apache) | **Fecha:** 26/09/2026 | **Autor:** krinoxx | **Plataforma:** [file_upload_vulnerability_scenarios (moeinfatehi)](https://github.com/moeinfatehi/file_upload_vulnerability_scenarios)

## Índice
- [Reconocimiento](#reconocimiento)
- [Enumeración web](#enumeración-web)
- [Acceso inicial](#acceso-inicial)
  - [Escenario 1 — Sin validación](#escenario-1--sin-validación)
  - [Escenario 3 — Validación solo client-side](#escenario-3--validación-solo-client-side)
  - [Escenario 10 — Blacklist de extensión (.php5)](#escenario-10--blacklist-de-extensión-php5)
  - [Escenario 11 — Blacklist de extensión (.pht)](#escenario-11--blacklist-de-extensión-pht)
  - [Escenario 12 — Subida de .htaccess](#escenario-12--subida-de-htaccess)
  - [Escenario 16 — Manipulación de MAX_FILE_SIZE](#escenario-16--manipulación-de-max_file_size)
  - [Escenario 17 — MAX_FILE_SIZE + short-tag](#escenario-17--max_file_size--short-tag)
  - [Escenario 21 — Bypass de Content-Type](#escenario-21--bypass-de-content-type)
  - [Escenario 23 — Magic bytes GIF](#escenario-23--magic-bytes-gif)
  - [Escenarios 31/33/35/41 — Subida ciega + descubrimiento](#escenarios-31333541--subida-ciega--descubrimiento)
  - [Escenario 51 — Doble extensión + magic bytes JPEG](#escenario-51--doble-extensión--magic-bytes-jpeg)
  - [Escenario 56 — Directorio controlado por parámetro](#escenario-56--directorio-controlado-por-parámetro)
  - [Escenario 58 — .htaccess + directorio controlado](#escenario-58--htaccess--directorio-controlado)
- [Escalada de privilegios](#escalada-de-privilegios)
- [Lecciones aprendidas](#lecciones-aprendidas)
  - [Desde el punto de vista del atacante](#desde-el-punto-de-vista-del-atacante)
  - [Desde el punto de vista del defensor](#desde-el-punto-de-vista-del-defensor)
- [Capturas](#capturas)

## Reconocimiento

El laboratorio consiste en una aplicación PHP dockerizada que expone múltiples formularios de subida de archivos, cada uno con una restricción distinta (extensión, tipo MIME, tamaño máximo, magic bytes, etc.), todos sirviendo por Apache en `localhost:9001`. Cada escenario está aislado bajo su propia ruta (`/upload1/`, `/upload3/`, `/upload10/`...), lo que permite practicar de forma incremental distintas técnicas de bypass sobre el mismo entorno.

Despliegue del laboratorio:

**Captura 1 — Clonación del repositorio**
![Captura 1](capturas/01.png)

**Captura 2 — Despliegue con docker-compose**
![Captura 2](capturas/02.png)

Herramientas usadas durante todo el ejercicio: **Burp Suite** (Repeater principalmente), **Firefox** con DevTools, **nvim** para preparar payloads, `xxd`/`file` para verificar magic bytes, `md5sum`/`sha1sum` para calcular hashes de nombres de archivo, `gobuster` para enumeración de directorios, y `curl` para automatizar peticiones de RCE.

## Enumeración web

En el escenario **41**, la respuesta de subida no revelaba la ruta del archivo subido ("File uploaded successfully" sin más detalle). Tras probar sin éxito a adivinar el nombre por hash del nombre del fichero, se lanzó `gobuster` contra la raíz del escenario:

```
gobuster dir -u http://localhost:9001/upload41/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

**Captura 36 — Lanzamiento de gobuster contra upload41**
![Captura 36](capturas/36.png)

**Captura 37 — Resultado: directorios `images/` y `static/` descubiertos**
![Captura 37](capturas/37.png)

El archivo subido resultó estar en `images/cmd.php`.

## Acceso inicial

A continuación se detalla cada técnica de bypass usada, escenario por escenario. El payload PHP base usado como shell en la mayoría de escenarios fue:

```php
<?php
    system($_GET['cmd']);
?>
```
(en los primeros escenarios se usó una variante fija `system("whoami")` para una comprobación rápida).

### Escenario 1 — Sin validación
El formulario indicaba "You can just upload [jpeg,gif] files", pero no aplicaba ninguna comprobación real en el servidor. Se subió `cmd.php` directamente vía Burp Repeater y se ejecutó desde `uploads/cmd.php`, obteniendo `www-data`.

**Captura 3 — Formulario del escenario 1**
![Captura 3](capturas/03.png)

**Captura 4 — Subida de cmd.php aceptada sin restricción**
![Captura 4](capturas/04.png)

**Captura 5 — RCE confirmado (whoami → www-data)**
![Captura 5](capturas/05.png)

### Escenario 3 — Validación solo client-side
Misma restricción aparente, pero la validación (`onsubmit="return Validate(this);"`) se ejecutaba únicamente en JavaScript del lado del cliente. Interceptando la petición en Burp Repeater y reenviándola directamente al servidor, se saltó el JS y se subió `cmd.php` sin problema.

**Captura 6 — Inspección del atributo onsubmit="return Validate(this)"**
![Captura 6](capturas/06.png)

**Captura 7 — Petición reenviada por Burp saltando la validación JS**
![Captura 7](capturas/07.png)

**Captura 8 — RCE confirmado**
![Captura 8](capturas/08.png)

### Escenario 10 — Blacklist de extensión (.php5)
El servidor bloqueaba explícitamente `.php` ("uploading php files is forbidden"). Se probó con la extensión alternativa `.php5` (reconocida como intérprete PHP por Apache en configuraciones antiguas), consiguiendo ejecución en `uploads/cmd.php5`.

**Captura 9 — Bloqueo de la extensión .php**
![Captura 9](capturas/09.png)

**Captura 10 — Bypass subiendo con extensión .php5**
![Captura 10](capturas/10.png)

**Captura 11 — RCE confirmado con cmd.php5**
![Captura 11](capturas/11.png)

### Escenario 11 — Blacklist de extensión (.pht)
Misma blacklist que el anterior. En este caso el bypass funcionó con la extensión `.pht`, otra variante reconocida por el handler de PHP de Apache.

**Captura 12 — Bypass subiendo con extensión .pht**
![Captura 12](capturas/12.png)

**Captura 13 — RCE confirmado con cmd.pht**
![Captura 13](capturas/13.png)

### Escenario 12 — Subida de .htaccess
Aquí se combinaron dos pasos: primero se subió un archivo `.htaccess` con la directiva:
```
AddType application/x-httpd-php .test
```
Esto le indica a Apache que trate cualquier archivo con esa extensión como PHP ejecutable. Después se subió el payload con dicha extensión (bypasseando así cualquier blacklist de extensiones "clásicas"), logrando RCE.

**Captura 14 — Subida del .htaccess con la directiva AddType**
![Captura 14](capturas/14.png)

**Captura 15 — Subida del payload con la extensión personalizada**
![Captura 15](capturas/15.png)

**Captura 16 — RCE confirmado**
![Captura 16](capturas/16.png)

### Escenario 16 — Manipulación de MAX_FILE_SIZE
El formulario incluía un campo oculto `MAX_FILE_SIZE` en el propio multipart, usado por PHP para rechazar el archivo si excede ese tamaño antes de aplicar cualquier otra validación. Se interceptó la petición y se incrementó el valor de dicho campo (de 30 a 80), permitiendo que el archivo pasara la primera comprobación y se subiera sin más restricciones.

**Captura 17 — Rechazo inicial por tamaño de archivo**
![Captura 17](capturas/17.png)

**Captura 18 — Modificación del campo oculto MAX_FILE_SIZE en Burp**
![Captura 18](capturas/18.png)

**Captura 19 — RCE confirmado tras el bypass**
![Captura 19](capturas/19.png)

### Escenario 17 — MAX_FILE_SIZE + short-tag
Restricción similar ("Extra security"), resuelta también ajustando `MAX_FILE_SIZE`. Adicionalmente, se sustituyó el payload por una short-tag con backticks para ejecución de comandos:
```php
<?=`$_GET[0]`?>
```
Ejecutado luego como `cmd.php?0=id`.

**Captura 20 — Payload con short-tag y backticks en Burp**
![Captura 20](capturas/20.png)

**Captura 21 — RCE confirmado**
![Captura 21](capturas/21.png)

### Escenario 21 — Bypass de Content-Type
El servidor validaba el header `Content-Type` de la parte multipart, rechazando `application/x-php`. Cambiando dicho header a `image/jpg` (sin tocar la extensión `.php` ni el contenido del payload), la subida fue aceptada y el archivo siguió siendo interpretado como PHP.

**Captura 22 — Rechazo por Content-Type application/x-php**
![Captura 22](capturas/22.png)

**Captura 23 — Bypass cambiando Content-Type a image/jpg**
![Captura 23](capturas/23.png)

**Captura 24 — RCE confirmado**
![Captura 24](capturas/24.png)

### Escenario 23 — Magic bytes GIF
Restricción "Upload gif file" que comprobaba los primeros bytes del archivo. Se antepuso la cabecera mágica de un GIF (`GIF8;`) al contenido PHP, y se ajustó `Content-Type: image/gif`.

**Captura 25 — Preparación del payload con cabecera mágica GIF8;**
![Captura 25](capturas/25.png)

**Captura 26 — Verificación local con xxd y file (detectado como GIF image data)**
![Captura 26](capturas/26.png)

**Captura 27 — Subida aceptada con Content-Type image/gif**
![Captura 27](capturas/27.png)

**Captura 28 — RCE confirmado**
![Captura 28](capturas/28.png)

### Escenarios 31/33/35/41 — Subida ciega + descubrimiento
En estos escenarios la respuesta tras subir el archivo no indicaba la ruta ("File uploaded successfully", sin más). Se probaron dos estrategias:

**Captura 29 — Subida "ciega": sin ruta revelada en la respuesta**
![Captura 29](capturas/29.png)

**Predicción de nombre por hash** (escenarios 31, 33 y 35): en varios casos el servidor renombraba el archivo usando el hash MD5 o SHA1 del nombre original.

**Captura 30 — Cálculo del hash MD5 del nombre original (upload31)**
![Captura 30](capturas/30.png)

**Captura 31 — Acceso al archivo por hash y RCE confirmado (upload31)**
![Captura 31](capturas/31.png)

**Captura 32 — Cálculo de hash MD5 incluyendo la extensión (upload33)**
![Captura 32](capturas/32.png)

**Captura 33 — Acceso al archivo por hash y RCE confirmado (upload33)**
![Captura 33](capturas/33.png)

**Captura 34 — Cálculo de hash SHA1 del nombre (upload35)**
![Captura 34](capturas/34.png)

**Captura 35 — Acceso al archivo por hash y RCE confirmado (upload35)**
![Captura 35](capturas/35.png)

**Enumeración de directorios** (escenario 41): cuando la predicción por hash no funcionó, se recurrió a `gobuster` (ver sección de Enumeración web), encontrando el archivo en `images/cmd.php`.

**Captura 38 — RCE confirmado tras localizar el archivo en /images/**
![Captura 38](capturas/38.png)

En todos los casos, la ejecución vía `?cmd=id` devolvió `uid=33(www-data)`.

### Escenario 51 — Doble extensión + magic bytes JPEG
Restricción "You can only upload jpg files". Se combinó doble extensión (`cmd.jpg.php`) junto con la cabecera mágica de JPEG (`JPEG;`) antepuesta al payload. El archivo se subió como `uploads/cmd.jpg.php` y se ejecutó correctamente.

**Captura 39 — Rechazo inicial (solo se permiten archivos jpg)**
![Captura 39](capturas/39.png)

**Captura 40 — Bypass con doble extensión cmd.jpg.php y cabecera JPEG;**
![Captura 40](capturas/40.png)

**Captura 41 — RCE confirmado**
![Captura 41](capturas/41.png)

### Escenario 56 — Directorio controlado por parámetro
Este formulario incluía un campo adicional de texto ("Your name — Keep it secret"), evidenciando (vía un warning de PHP `Undefined variable $target_file`) una mala gestión de la ruta de destino en el código fuente. Tras la subida a `testing/cmd.php`, se automatizó la explotación con `curl`.

**Captura 42 — Subida usando el campo de nombre del formulario**
![Captura 42](capturas/42.png)

**Captura 43 — Verificación con curl sin parámetro cmd (error fatal en system())**
![Captura 43](capturas/43.png)

**Captura 44 — RCE confirmado: id y volcado de /etc/passwd vía curl**
![Captura 44](capturas/44.png)

**Captura 45 — Obtención de la IP interna del contenedor (hostname -I)**
![Captura 45](capturas/45.png)

### Escenario 58 — .htaccess + directorio controlado
Combinación de las dos técnicas anteriores: el parámetro de texto (`username=cmd`) definía parte de la ruta de subida, y se aprovechó para subir primero un `.htaccess` con `AddType application/x-httpd-php .pwned`, y después el payload con extensión `.pwned`, quedando accesible en `cmd/cmd.pwned` y ejecutándose como PHP.

**Captura 46 — Preparación del .htaccess con la extensión personalizada .pwned**
![Captura 46](capturas/46.png)

**Captura 47 — Subida del .htaccess vía el parámetro de nombre**
![Captura 47](capturas/47.png)

**Captura 48 — Subida del payload con extensión .pwned**
![Captura 48](capturas/48.png)

**Captura 49 — RCE final confirmado**
![Captura 49](capturas/49.png)

## Escalada de privilegios

No fue necesaria escalada de privilegios: el objetivo de todos los escenarios es únicamente conseguir ejecución de comandos arbitrarios a través del bypass del filtro de subida de archivos. En todos los casos la ejecución quedó confinada al usuario `www-data` (uid=33), propio del servidor web.

## Lecciones aprendidas

### Desde el punto de vista del atacante
- Nunca confiar en un único punto de rechazo: probar sistemáticamente extensión, Content-Type, contenido (magic bytes) y campos ocultos del formulario (como `MAX_FILE_SIZE`) por separado.
- Cuando la respuesta no revela la ruta de subida, no dar la vulnerabilidad por "no explotable": predecir el nombre (hashes de nombres comunes) o enumerar directorios con herramientas como `gobuster` suele ser suficiente.
- Las extensiones ejecutables por Apache no se limitan a `.php`: variantes como `.php5`, `.pht`, `.phtml` o extensiones completamente arbitrarias vía `.htaccess` son vectores igual de válidos.
- Un `.htaccess` subido por el propio atacante puede redefinir por completo qué extensiones ejecuta el servidor como código, siendo en sí mismo una vulnerabilidad crítica si se permite su subida.

### Desde el punto de vista del defensor
- La validación de subida de archivos debe hacerse siempre en el servidor, nunca solo en el cliente (JavaScript), y debe basarse en una whitelist estricta de extensiones y tipos MIME, no en una blacklist.
- Comprobar únicamente la extensión o el `Content-Type` declarado por el cliente es insuficiente; ambos son controlables por el atacante. Se recomienda verificar el contenido real del archivo (magic bytes reales, no solo los primeros bytes) y, preferiblemente, reescribir/normalizar la imagen tras la subida.
- Los archivos subidos por usuarios nunca deberían almacenarse en un directorio donde el servidor web pueda ejecutarlos como código; lo ideal es servirlos desde un directorio sin permisos de ejecución o desde un dominio/almacenamiento separado.
- Debe prohibirse explícitamente la subida de archivos `.htaccess` (o cualquier fichero de configuración del servidor), y desactivar `AllowOverride` cuando no sea estrictamente necesario.
- Cualquier parámetro de usuario que influya en la ruta de destino de un archivo (como el campo "nombre" del escenario 56/58) debe ser saneado o ignorado; permitir "path/naming injection" en subidas es tan peligroso como la propia subida de código ejecutable.

## Capturas

Todas las capturas están numeradas según el orden en que aparecen en el writeup y se encuentran en la carpeta `capturas/` junto a este documento.

---
*Writeup educativo en entorno controlado.*
