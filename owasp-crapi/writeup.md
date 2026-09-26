# crAPI — Abuso de APIs

**Dificultad:** Media | **SO:** Linux (contenedores Docker) | **Fecha:** 25/09/2026 | **Autor:** krinoxx | **Plataforma:** [github.com/OWASP/crAPI](https://github.com/OWASP/crAPI)

## Índice
- [Reconocimiento](#reconocimiento)
- [Enumeración web](#enumeración-web)
- [Acceso inicial](#acceso-inicial)
- [Escalada de privilegios](#escalada-de-privilegios)
- [Lecciones aprendidas](#lecciones-aprendidas)
  - [Desde el punto de vista del atacante](#desde-el-punto-de-vista-del-atacante)
  - [Desde el punto de vista del defensor](#desde-el-punto-de-vista-del-defensor)
- [Capturas](#capturas)

## Reconocimiento

Se clona el repositorio oficial de OWASP crAPI (`git clone https://github.com/OWASP/crAPI`) y se despliega el entorno completo mediante Docker Compose (`docker compose -f docker-compose.yml --compatibility up -d`), levantando los microservicios que componen la aplicación: `crapi-identity`, `crapi-community`, `crapi-workshop`, `crapi-chatbot`, `crapi-web`, `mongodb`, `postgresdb`, `mailhog` y `chromadb`.

La aplicación queda expuesta en `http://localhost:8888`, con MailHog (servidor SMTP de pruebas) accesible en el puerto 8025 para capturar todos los correos que envía la app (OTPs, datos de vehículos, etc.), un detalle clave para varias de las vulnerabilidades explotadas más adelante.

Se configura Postman como cliente principal para interactuar con la API, replicando cada acción realizada desde el frontend para poder manipular las peticiones directamente.

## Enumeración web

Navegando por la SPA de crAPI (Dashboard, Shop, Community) y observando el tráfico con las DevTools del navegador se identifican los siguientes endpoints principales, repartidos entre distintos microservicios:

- `POST /identity/api/auth/signup` y `/login` — registro y autenticación
- `GET /identity/api/v2/user/dashboard` — datos del usuario autenticado
- `GET /identity/api/v2/vehicle/{carId}/location` — ubicación del vehículo asociado a un `carId`
- `POST /identity/api/auth/forgot-password` y `POST /identity/api/auth/v2|v3/check-otp` — flujo de recuperación de contraseña por OTP
- `GET /workshop/api/shop/products` y `POST /workshop/api/shop/products` — catálogo de productos de la tienda
- `POST /workshop/api/shop/orders` — creación de pedidos
- `POST /workshop/api/shop/orders/return_order` — devolución de pedidos
- `POST /community/api/v2/coupon/validate-coupon` — validación de cupones de descuento
- `GET /community/api/v2/community/posts/recent` y `GET /community/api/v2/community/posts/{post_id}` — publicaciones de otros usuarios

Cada endpoint se guarda como request en una colección de Postman ("crAPI"), usando una variable de colección `accessToken` (con el JWT obtenido en el login) referenciada como `{{accessToken}}` en la pestaña Authorization (Bearer Token) del resto de peticiones.

## Acceso inicial

Se realiza el signup de un usuario normal (Mario Perez / krxx@krxx.com) y el login correspondiente. La respuesta del login devuelve un JWT (`type: Bearer`) que se captura tanto desde las DevTools como desde Postman y se guarda como variable de colección para autenticar automáticamente el resto de peticiones.

Con el token válido se confirma el acceso legítimo al dashboard (`GET /identity/api/v2/user/dashboard`), que expone los datos del propio usuario: `id`, `name`, `email`, `number`, `available_credit` (100.0 inicial) y `role: ROLE_USER`. Se realiza también una compra legítima en la tienda para observar el comportamiento normal del flujo de pedidos (`credit` bajando de 100.0 a 90.0) y se prueba la devolución de un pedido ya devuelto, confirmando que ese endpoint sí valida correctamente el estado (`400 Bad Request — "This order is already returned!"`).

## Escalada de privilegios

A partir de aquí se abusa de varios fallos de diseño de la API (más que "escalada de privilegios" clásica, son fallos de autorización y de lógica de negocio típicos de la OWASP API Security Top 10):

**1. Fuerza bruta de OTP — endpoint legacy sin rate-limiting (Broken Authentication / API versioning)**
En el flujo de "Forgot Password" se solicita un OTP y se prueba manualmente un valor inventado (`1234`), que la API rechaza con un `500 Internal Server Error` y el mensaje "Invalid OTP! Please try again..". Se lanza `ffuf` contra `v3/check-otp` con una wordlist de 4 dígitos (0000-9999), pero tras un número limitado de peticiones el endpoint deja de responder correctamente: `v3` sí implementa algún tipo de rate-limiting/bloqueo, que corta el fuzzing y evita seguir probando valores.

Sin embargo, la propia API sigue exponiendo una versión anterior del mismo endpoint (`v2/check-otp`) que, al parecer, nunca recibió el mismo parche de protección. Repitiendo exactamente el mismo ataque contra `v2/check-otp` (matcher `-mc 200`) el fuzzing corre sin ningún bloqueo y se localiza el código correcto en segundos. MailHog confirma el OTP real enviado por correo (`1249`), que efectivamente devuelve `200 OTP verified` contra la `v2`, permitiendo resetear la contraseña de cualquier cuenta conociendo solo el email.

Esto es un fallo clásico de gestión de versiones de API: se parchea la vulnerabilidad en la versión "actual" del endpoint, pero se olvida retirar o parchear igualmente la versión anterior, que sigue activa y accesible sin ningún control adicional.

**2. Mass Assignment / Business Logic Flaw — precios negativos**
Se hace fuzzing de métodos HTTP contra `/workshop/api/shop/products` (`ffuf -X FUZZ`), confirmando que `POST` está permitido (401 sin auth, pero no bloqueado a nivel de método). Con el token válido, un `POST` sin campos revela los campos requeridos (`name`, `price`, `image_url`) vía el mensaje de error 400. Enviando un producto con `price: -10000` la API lo acepta sin validar que el precio sea positivo, devolviendo `200 OK` y creando el producto "Hacked" con `id: 3`.

**3. Explotación de la lógica de negocio para inflar el saldo**
El producto de precio negativo aparece listado en la tienda junto a Wheel y Seat. Replicando la compra directamente contra `/workshop/api/shop/orders` con `{"product_id":3,"quantity":100}`, la resta de un precio negativo multiplicado por 100 unidades dispara el `credit` del usuario a `1010090.0`.

**4. NoSQL Injection en la validación de cupones**
El endpoint `/community/api/v2/coupon/validate-coupon` devuelve "Invalid Coupon Code" para un cupón inventado (`123`) y un `500 Internal Server Error` al replicarlo tal cual en Postman. Inyectando un operador de MongoDB en el JSON (`{"coupon_code":{"$ne":"123"}}`) la consulta interna interpreta la condición como "distinto de 123" y devuelve el primer cupón válido de la base de datos (`TRAC075`, `amount: 75`), que se aplica correctamente ("Coupon applied"), sumando más saldo.

**5. Broken Object Level Authorization (BOLA/IDOR) — ubicación de vehículos ajenos**
El correo de bienvenida ("Welcome to crAPI") capturado en MailHog incluye el VIN y el PIN del vehículo asociado a la cuenta, usados para verificarlo en `/verify-vehicle`. Una vez vinculado, el dashboard consulta `GET /identity/api/v2/vehicle/{carId}/location`, exponiendo la ubicación GPS, nombre y email del propietario.
Navegando por los posts de la comunidad se descubre que la respuesta de cada post expone datos adicionales del autor, incluyendo su `vehicleId` (en este caso, del usuario "Adam", `adam007@example.com`). Reutilizando ese `vehicleId` ajeno contra el mismo endpoint de localización (`GET /identity/api/v2/vehicle/{vehicleId_de_Adam}/location`) sin ningún control de propiedad, la API devuelve `200 OK` con la ubicación exacta, nombre y email de Adam — un fallo clásico de BOLA, ya que el endpoint no verifica que el `carId`/`vehicleId` solicitado pertenezca al usuario autenticado.

## Lecciones aprendidas

### Desde el punto de vista del atacante
- Los endpoints de autenticación secundarios (recuperación de contraseña, verificación de OTP) suelen tener menos controles que el login principal, y son un objetivo prioritario para fuerza bruta si el OTP es corto y no hay rate-limiting.
- Cuando un endpoint versionado (`v3`) bloquea el ataque, merece la pena probar las versiones anteriores (`v2`, `v1`) del mismo endpoint: es muy común que un parche de seguridad se aplique solo a la versión "actual" y se olvide la versión legacy, que sigue expuesta y sin protección.
- Los IDs referenciados en un recurso (posts, comentarios, perfiles públicos) casi siempre filtran identificadores de otros recursos (`vehicleId`, `userId`) reutilizables contra otros endpoints — conviene mapear todos los IDs que aparecen en cualquier respuesta, no solo los de la petición actual.
- Los campos numéricos de negocio (precios, cantidades, importes) rara vez se validan por rango en aplicaciones de prueba/demo: probar siempre valores negativos, cero y extremos.
- Operadores de bases de datos NoSQL (`$ne`, `$gt`, `$regex`, etc.) inyectados en campos JSON son un vector de bypass de validación muy efectivo cuando la API no sanea la entrada antes de construir la query.

### Desde el punto de vista del defensor
- Implementar rate-limiting y bloqueo temporal de cuenta tras varios intentos fallidos de OTP, además de aumentar su longitud/entropía y limitar su ventana de validez.
- Validar en el backend (nunca solo en el frontend) los rangos permitidos de campos de negocio como precios y cantidades, rechazando valores negativos o fuera de un rango razonable.
- Sanear y tipar estrictamente los inputs antes de pasarlos a consultas NoSQL, evitando interpretar objetos JSON arbitrarios como operadores de query (usar esquemas de validación como whitelisting de tipos).
- Aplicar controles de autorización a nivel de objeto (BOLA) en todo endpoint que reciba un identificador de recurso, verificando siempre que el recurso solicitado pertenece al usuario autenticado, y evitar exponer identificadores internos (`vehicleId`, `carId`) en respuestas públicas como posts de comunidad.

*Writeup educativo en entorno controlado.*

## Capturas

### 01. Clonado del repositorio OWASP crAPI (`git clone`)
![](capturas/01.png)

### 02. Despliegue de todos los contenedores con `docker compose up -d`
![](capturas/02.png)

### 03. Formulario de Signup (Mario Perez / krxx@krxx.com)
![](capturas/03.png)

### 04. Login con DevTools abierto — Network mostrando el JWT en la respuesta
![](capturas/04.png)

### 05. Postman — POST `/identity/api/auth/login`, token recibido
![](capturas/05.png)

### 06. Postman — variable de colección `accessToken` guardada con el JWT
![](capturas/06.png)

### 07. Postman — Authorization Bearer Token `{{accessToken}}` a nivel de colección
![](capturas/07.png)

### 08. Postman — GET Dashboard, `available_credit: 100.0`
![](capturas/08.png)

### 09. Navegador Shop — compra legítima, Network mostrando `credit: 90.0`
![](capturas/09.png)

### 10. Navegador Past Orders — Return de un pedido, QR code de UPS
![](capturas/10.png)

### 11. Postman "Return" — repetir la devolución → 400 "Order already returned"
![](capturas/11.png)

### 12. Navegador Forgot Password (paso 2) — OTP inventado `1234` → 500
![](capturas/12.png)

### 13. Postman "Check OTP" — POST `v3/check-otp` con `otp:"1234"` → 500
![](capturas/13.png)

### 14. Terminal — `ffuf` fuzzing OTP de 4 dígitos contra `v3/check-otp` (interrumpido)
![](capturas/14.png)

### 15. Postman "Check OTP" — POST `v3/check-otp` con `otp:"0000"` → 500 "ERROR.."
![](capturas/15.png)

### 16. Postman "Check OTP" — POST `v2/check-otp` con `otp:"0000"` → 500 "Invalid OTP..."
![](capturas/16.png)

### 17. Terminal — `ffuf` fuzzing OTP contra `v2/check-otp` (matcher 200) → encuentra `1249`
![](capturas/17.png)

### 18. MailHog — email "crAPI OTP" con el código real `1249`
![](capturas/18.png)

### 19. Postman — respuesta `200 OK`, `"message": "OTP verified"`
![](capturas/19.png)

### 20. Terminal — `ffuf` fuzzing de métodos HTTP contra `/shop/products` — OPTIONS 200, resto 401
![](capturas/20.png)

### 21. Postman "Available Products" — POST sin campos → 400 pidiendo `name`, `price`, `image_url`
![](capturas/21.png)

### 22. Postman "Available Products" — POST con `price: "-10000"` → 200 OK, producto "Hacked" (`id: 3`)
![](capturas/22.png)

### 23. Navegador Shop — producto "Hacked" ($-10000.00) junto a Wheel y Seat, balance $90
![](capturas/23.png)

### 24. Postman "Orders" — POST `product_id: 3, quantity: 100` → 200 OK, `credit: 1010090.0`
![](capturas/24.png)

### 25. Navegador Shop — modal "Enter Coupon Code" con "123" → "Invalid Coupon Code"
![](capturas/25.png)

### 26. Postman "Cupon" — NoSQL injection `{"$ne":"123"}` → 200 OK, cupón válido `TRAC075` (75)
![](capturas/26.png)

### 27. Popup "Success — Coupon applied"
![](capturas/27.png)

### 28. MailHog — email "Welcome to crAPI" con VIN y Pincode del vehículo propio
![](capturas/28.png)

### 29. Navegador Verify Vehicle Details — formulario con PIN y VIN rellenados
![](capturas/29.png)

### 30. Navegador Dashboard — ubicación del propio vehículo en el mapa, Network GET `/vehicle/{carId}/location`
![](capturas/30.png)

### 31. Navegador Community — post de otro usuario (Adam) exponiendo su `vehicleId` y email
![](capturas/31.png)

### 32. Postman "Location" — GET `/vehicle/{vehicleId_de_Adam}/location` → 200 OK, BOLA confirmado
![](capturas/32.png)
