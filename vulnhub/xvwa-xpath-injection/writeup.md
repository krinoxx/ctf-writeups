# XVWA — XPATH Injection — VulnHub
**Dificultad:** Fácil | **SO:** Ubuntu 14.04.5 LTS | **Fecha:** 13/10/2026 | **Autor:** krinoxx | **Plataforma:** [Xtreme Vulnerable Web Application (XVWA): 1 (VulnHub)](https://www.vulnhub.com/entry/xtreme-vulnerable-web-application-xvwa-1,209/)

## Índice
- [Reconocimiento](#reconocimiento)
- [Enumeración web](#enumeración-web)
- [Explotación — XPath Injection](#explotación--xpath-injection)
- [Lecciones aprendidas](#lecciones-aprendidas)
  - [Desde el punto de vista del atacante](#desde-el-punto-de-vista-del-atacante)
  - [Desde el punto de vista del defensor](#desde-el-punto-de-vista-del-defensor)

---

## Reconocimiento

XVWA (Xtreme Vulnerable Web Application) es una ISO de VulnHub que despliega una máquina Ubuntu con una aplicación web intencionadamente vulnerable (PHP/MySQL), pensada para practicar distintas clases de vulnerabilidades web. En este caso el foco es el módulo de **XPATH Injection**.

### 1. Descarga de la ISO desde VulnHub
![01 - Descarga ISO VulnHub](capturas/01_descarga_iso_vulnhub.png)

### 2. Creación de la máquina virtual en VMware Workstation
Se crea la VM con red en modo Bridged para que quede en el mismo segmento que la máquina atacante.
![02 - Creación VM VMware](capturas/02_creacion_vm_vmware.png)

### 3. Arranque de la VM y obtención de IP
Al arrancar, Ubuntu 14.04.5 LTS asigna por DHCP la IP `192.168.1.59`.
![03 - Arranque VM e IP](capturas/03_arranque_vm_ip.png)

### 4. Descubrimiento del host con arp-scan
![04 - arp-scan](capturas/04_arp-scan.png)

### 5. Verificación de conectividad con ping
![05 - ping](capturas/05_ping.png)

---

## Enumeración web

### 6. Acceso al módulo XPATH Injection
Dentro de XVWA, en el menú **Attacks → XPATH Injection**, encontramos un formulario "Search Your Coffee" que busca por `ID` en un backend XML.
![06 - Acceso módulo XPATH](capturas/06_acceso_modulo_xpath.png)

### 7. Configuración del proxy (FoxyProxy + Burp Suite)
Se configura FoxyProxy en el navegador para enrutar el tráfico a través de Burp Suite y poder interceptar y manipular las peticiones.
![07 - FoxyProxy + Burp](capturas/07_foxyproxy_burp.png)

### 8. Petición baseline en Burp
Se captura una petición normal (`search=1`) para tener una referencia del tamaño de respuesta (longitud de `r.text`) con la que comparar más adelante.
![08 - Petición baseline](capturas/08_peticion_baseline_burp.png)

---

## Explotación — XPath Injection

### Confirmación de la vulnerabilidad

#### 9. Inyección de comilla simple
Al enviar `search=1'`, la aplicación responde "Item Not Found", lo que indica que la comilla rompe la consulta XPath subyacente — primer indicio de inyección.
![09 - Comilla simple](capturas/09_comilla_simple.png)

#### 10. Confirmación boolean-based
Con `search=1' or '1'='1`, la condición siempre verdadera devuelve todos los registros del XML, confirmando que podemos alterar la lógica de la consulta.
![10 - Boolean-based](capturas/10_boolean_or_1=1.png)

#### 11. Descartando sintaxis de SQL Injection
Se prueba `1' order by 100-- -`, típico de SQLi, pero XPath no soporta esta cláusula: la respuesta sigue siendo "Item Not Found".
![11 - order by descartado](capturas/11_order_by_descartado.png)

#### 12. Descartando inyección basada en tiempo
De igual forma, `1' and sleep(5)-- -` no tiene efecto, ya que XPath no dispone de una función `sleep()` nativa como SQL.
![12 - sleep descartado](capturas/12_sleep_descartado.png)

### Extracción manual de la estructura XML

Confirmado que estamos ante XPath (no SQL), se usan funciones propias de XPath 1.0 (`name()`, `substring()`, `string-length()`, `count()`) para ir reconstruyendo, carácter a carácter y nodo a nodo, la estructura del documento XML sin tener acceso directo al archivo.

#### 13. Primer carácter del nombre del nodo raíz
`1' and substring(name(/*[1]),1,1)='C` confirma que el nombre del nodo raíz empieza por "C" (Coffees).
![13 - substring nombre raíz](capturas/13_substring_nombre_raiz.png)

#### 14. Longitud del nombre del nodo raíz
`1' and string-length(name(/*[1]))='7` confirma que el nombre tiene 7 caracteres.
![14 - string-length nombre raíz](capturas/14_string-length_nombre_raiz.png)

#### 15. Número de nodos hijos del nodo raíz
`1' and count(/*[1]/*)='10` revela que el nodo raíz tiene 10 hijos directos (10 elementos `Coffee`).
![15 - count hijos raíz](capturas/15_count_hijos_raiz.png)

#### 16. Nombre de un nodo hijo
`1' and name(/*[1]/*[10])='Coffee` confirma el nombre de cada elemento hijo.
![16 - nombre Coffee](capturas/16_nombre_coffee.png)

#### 17. Primer carácter del primer elemento Coffee
`1' and substring(name(/*[1]/*[1]),1,1)='C`
![17 - substring primer Coffee](capturas/17_substring_primer_coffee.png)

#### 18-19. Número de campos por cada Coffee
Primero se prueba con `count(/*[1]/*[1]/*)>'4` (mayor que 4) y después se ajusta a `='5`, confirmando que el primer `Coffee` tiene 5 campos.
![18 - count campos mayor 4](capturas/18_count_campos_mayor4.png)
![19 - count campos igual 5](capturas/19_count_campos_igual5.png)

#### 20-21. El número de campos no es constante entre elementos
Al repetir la prueba sobre el segundo `Coffee` con `='5` la condición falla; bajando a `='4` se confirma que no todos los elementos tienen la misma cantidad de campos (pista de que hay un campo oculto en algunos registros).
![20 - count 2º Coffee = 5 (falla)](capturas/20_count_2do_coffee_5_falla.png)
![21 - count 2º Coffee = 4 (confirma)](capturas/21_count_2do_coffee_4_confirma.png)

#### 22-23. Nombres de los campos
Se extraen los nombres de cada campo del nodo: `ID`, `Name`, etc.
![22 - nombre campo ID](capturas/22_nombre_campo_id.png)
![23 - nombre campo Name](capturas/23_nombre_campo_name.png)

#### 24. Verificación carácter a carácter
`1' and substring(name(/*[1]/*[1]/*[1]),2,1)='D` — segundo carácter del nombre del campo.
![24 - segundo carácter campo](capturas/24_segundo_caracter_campo.png)

#### 25. Descubrimiento de un campo oculto (Secret)
En el primer `Coffee`, que tenía 5 campos en lugar de 4, se identifica un campo adicional llamado `Secret`, y se usa `string-length(Secret)>='20` para empezar a acotar su longitud.
![25 - descubriendo campo Secret](capturas/25_descubriendo_campo_secret.png)

### Automatización con script Python

Repetir manualmente este proceso para los 10 elementos `Coffee` sería extremadamente lento, así que se automatiza toda la lógica anterior en un script en Python (usando `requests` y `pwntools` para el log de progreso), capaz de:

- Determinar el número de etiquetas primarias, secundarias y terciarias.
- Calcular la longitud de cada nombre de etiqueta.
- Extraer el nombre de cada etiqueta carácter a carácter.
- Extraer el valor (texto) de cada etiqueta hoja.
- Reconstruir el XML completo y formatearlo con `xmllint`.

#### 26. Variables globales y `GetNumPrimaryTags()`
![26 - cabecera y GetNumPrimaryTags](capturas/26_script_cabecera_getnumprimarytags.png)

#### 27. `GetNumOfSecodaryTags()` y `GetNumOfTerciaryTags()`
![27 - GetNumOfSecodaryTags / GetNumOfTerciaryTags](capturas/27_script_getnumofsecondary_terciary.png)

#### 28. `GetLengthOfTag()` y `GetLengthOfSubTag()`
![28 - GetLengthOfTag / GetLengthOfSubTag](capturas/28_script_getlengthoftag_subtag.png)

#### 29. `GetLengthOfSubSubTag()` y `GetLengthOfSubSubSubTag()`
![29 - GetLengthOfSubSubTag / GetLengthOfSubSubSubTag](capturas/29_script_getlengthofsubsubtag.png)

#### 30. `GetNameOfPrimaryTags()`
![30 - GetNameOfPrimaryTags](capturas/30_script_getnameofprimarytags.png)

#### 31. `GetNameOfSubTag()`
![31 - GetNameOfSubTag](capturas/31_script_getnameofsubtag.png)

#### 32. `GetNameOfSubSubTag()`
![32 - GetNameOfSubSubTag](capturas/32_script_getnameofsubsubtag.png)

#### 33. `GetValueOfSubSubTag()`
Esta función extrae el valor final de cada campo hoja, usando un juego de caracteres ampliado (`Allcharacters`: letras, mayúsculas, dígitos y puntuación) para cubrir textos descriptivos y precios.

![33 - GetValueOfSubSubTag](capturas/33_script_getvalueofsubsubtag.png)

#### 34. `selectTag()` y bloque principal
El bloque `__main__` encadena todas las funciones anteriores, pide al usuario qué etiqueta primaria quiere volcar, y al final reconstruye el árbol XML completo con `ElementTree`, escribiéndolo en `pwned.xml` y formateándolo con `xmllint`.

![34 - selectTag + main](capturas/34_script_selecttag_main.png)

### Ejecución del script

#### 35. Primera parte de la ejecución
El script detecta automáticamente 1 etiqueta primaria (`Coffees`), 10 subetiquetas (`Coffee`), y dentro del primer `Coffee` sus etiquetas terciarias (`ID`, `Name`...).

![35 - Ejecución parte 1](capturas/35_ejecucion_script_parte1.png)

#### 36. Extracción de Name y Desc en curso
Fuerza bruta carácter a carácter de los campos `Name` ("Affogato") y `Desc` (la descripción larga del café).

![36 - Ejecución parte 2](capturas/36_ejecucion_script_parte2.png)

#### 37. Extracción de Price, Secret y comienzo del segundo Coffee
Se obtienen `Price` ("$4.69") y el campo oculto `Secret`, confirmando que efectivamente contenía un dato adicional no visible en la interfaz normal de la aplicación.

![37 - Ejecución parte 3](capturas/37_ejecucion_script_parte3.png)

### Verificación de los datos extraídos

Al tratarse de un laboratorio propio, se tuvo acceso a la consola de la VM para comprobar que los datos extraídos vía XPath Injection coincidían exactamente con el archivo real en el servidor.

#### 38. Listado de archivos del módulo vulnerable
![38 - ls en el servidor](capturas/38_verificacion_ls_servidor.png)

#### 39. Apertura de `coffee.xml` en el servidor
![39 - nano coffee.xml](capturas/39_verificacion_nano_abriendo.png)

#### 40. Contenido del archivo real en el servidor
![40 - contenido coffee.xml](capturas/40_verificacion_contenido_xml.png)

#### 41. Vista completa confirmando el campo `Secret`
Se confirma que el campo `Secret` extraído mediante la inyección coincide con el valor real almacenado en el XML del servidor, al que un usuario normal de la aplicación nunca tendría acceso por la interfaz.

![41 - vista completa XML](capturas/41_verificacion_xml_completo.png)

---

## Lecciones aprendidas

### Desde el punto de vista del atacante
- Ante un punto de entrada que maneja datos estructurados en XML, probar primero con una comilla simple es una forma rápida y de bajo ruido de detectar si hay inyección.
- No asumir que toda inyección en un campo de búsqueda es SQL: comprobar la ausencia de respuesta ante sintaxis típica de SQLi (`order by`, `sleep()`) ayudó a descartar esa hipótesis y a pivotar hacia funciones propias de XPath (`name()`, `count()`, `string-length()`, `substring()`).
- La extracción ciega carácter a carácter es viable manualmente para validar la técnica, pero automatizarla con un script es imprescindible para explotar documentos XML de tamaño real en un tiempo razonable.
- No se puede asumir que todos los nodos de un mismo nivel tienen la misma estructura (en este caso, un elemento `Coffee` tenía un campo extra, `Secret`, que los demás no tenían) — hay que comprobarlo nodo a nodo en vez de generalizar.

### Desde el punto de vista del defensor
- El problema de raíz es construir consultas XPath concatenando directamente la entrada del usuario. La solución pasa por usar consultas parametrizadas (XPath variables) en lugar de interpolar strings.
- Aplicar validación estricta de entrada (whitelisting de formato numérico, por ejemplo, si el campo solo debería aceptar un ID) habría bloqueado la mayoría de estos payloads.
- Los mensajes de error diferenciados ("Item Not Found" vs. resultados) facilitan ataques blind: conviene homogeneizar las respuestas de la aplicación ante entradas inválidas o maliciosas.
- Nunca almacenar información sensible (como el campo `Secret` encontrado aquí) en el mismo documento de datos accesible por una consulta no autenticada, por muy "oculto" que parezca estar a nivel de interfaz.

*Writeup educativo en entorno controlado.*
