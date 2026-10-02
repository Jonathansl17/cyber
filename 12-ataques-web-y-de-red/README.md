# Ataques web y de red

## Conceptos previos

- Aplicación web: programa que corre en un servidor y que el usuario usa desde el navegador mediante peticiones HTTP.
- Petición y respuesta HTTP: el navegador manda un método (`GET`, `POST`), una ruta, cabeceras y a veces un cuerpo; el servidor responde con un código (`200`, `403`, `404`, `500`) y contenido.
- Cookie de sesión: valor que el servidor entrega tras el login y que el navegador reenvía solo en cada petición a ese sitio; quien la tenga "es" el usuario (ver [`08-autenticacion`](../08-autenticacion/)).
- Navegador como intérprete: todo lo que llega como HTML se dibuja y todo lo que llega como `<script>` se ejecuta con los permisos de ese sitio.
- Origen (origin): la combinación esquema + dominio + puerto (`https://banco.com:443`); la same-origin policy impide que el JavaScript de un origen lea las respuestas de otro.
- Base de datos SQL: almacén de tablas que se consulta con sentencias como `SELECT * FROM users WHERE id = 5`.
- Intérprete: cualquier programa que recibe texto y lo ejecuta como instrucciones (el motor SQL, el navegador, la shell, el parser de XML).
- Entrada no confiable: todo dato que llega de fuera del programa (formulario, URL, cabecera, cookie, archivo subido, respuesta de otra API); se asume que puede ser malicioso.
- Capa 2 y capa 3: la capa 2 mueve tramas dentro de una red local usando direcciones MAC; la capa 3 mueve paquetes entre redes usando direcciones IP (ver [`03-fundamentos-de-redes`](../03-fundamentos-de-redes/)).
- ARP: protocolo que pregunta "quién tiene la IP X" y recibe "la tiene la MAC Y"; no verifica quién responde.
- VLAN y trunk: una VLAN separa un switch en varias redes lógicas; un puerto trunk transporta varias VLAN marcando cada trama con una etiqueta 802.1Q de 4 bytes.
- DNS y resolver: el DNS traduce nombres a IP; el resolver es el servidor que hace esa consulta por el cliente y guarda la respuesta en caché durante su TTL (ver [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/)).
- Punto de acceso (AP), SSID y BSSID: el AP es el equipo que emite la red Wi-Fi; el SSID es su nombre visible y el BSSID es la MAC de su radio.
- Hash de contraseña: resultado de pasar la contraseña por una función de un solo sentido; el sistema guarda el hash, no la contraseña (ver [`10-criptografia`](../10-criptografia/)).
- Pila (stack) y heap: dos zonas de memoria de un proceso; la pila guarda variables locales y la dirección de retorno de cada función, el heap guarda memoria pedida en tiempo de ejecución con `malloc`/`new`.
- CWE: Common Weakness Enumeration, el catálogo de MITRE que da un número a cada tipo de falla de software (CWE-89 es SQL injection).
- Laboratorio legal: máquina virtual propia o plataforma diseñada para practicar (DVWA, OWASP Juice Shop, PortSwigger Web Security Academy, TryHackMe). Esta nota explica mecanismos para reconocerlos y corregirlos; la práctica ofensiva se hace solo ahí, nunca contra sistemas ajenos.

## Web Based Attacks and OWASP10

### Web Based Attacks and OWASP10

Un **ataque web** es un abuso de una aplicación accesible por HTTP que aprovecha cómo procesa la entrada, cómo controla quién hace qué o cómo está configurada, para leer, cambiar o destruir datos que no deberían estar a su alcance. El **OWASP Top 10** es un documento de concienciación que ordena las diez categorías de riesgo más críticas para aplicaciones web, construido con datos de pruebas reales y una encuesta a profesionales. OWASP (Open Worldwide Application Security Project) es una fundación sin fines de lucro que publica guías, herramientas y laboratorios abiertos.

Existe porque un equipo de desarrollo no puede estudiar los cientos de tipos de falla del catálogo CWE uno por uno. El Top 10 agrupa esas fallas en diez cajas, cada una con varias CWE dentro, y las ordena por cuántas aplicaciones las tienen, qué tan fáciles son de explotar y cuánto daño causan. Así una empresa sabe por dónde empezar, y un auditor tiene un vocabulario común para escribir hallazgos.

Cómo se arma: OWASP pide a empresas de auditoría y de herramientas el número de aplicaciones probadas y cuántas tenían al menos una instancia de cada CWE. Ocho de las diez categorías salen de esos datos; las otras dos salen de la encuesta a la comunidad, porque los datos solo reflejan lo que las herramientas automáticas ya saben detectar y llegan con años de retraso. Para la puntuación de explotabilidad e impacto se usan los CVSS de los CVE asociados a cada CWE.

Analogía: es como la lista de las diez causas más comunes de incendios en casas que publica el cuerpo de bomberos. No cubre todos los incendios posibles, pero si revisas esas diez cosas evitas la mayoría.

Ejemplo: una tienda en línea contrata una auditoría. El informe no dice "encontramos 37 problemas sueltos", sino "4 hallazgos de A01 Broken Access Control, 2 de A05 Injection, 1 de A02 Security Misconfiguration". Con eso el equipo sabe que el problema de fondo es el control de acceso y prioriza ahí.

> [!WARNING]
> El Top 10 es una lista de concienciación, no un estándar de verificación completo: "cumplir el Top 10" no significa que la aplicación sea segura. Para verificar se usa OWASP ASVS (Application Security Verification Standard).

```
OWASP Top 10 → ranking de categorías de riesgo; sirve para priorizar y concienciar.
OWASP ASVS → lista de requisitos verificables; sirve para auditar o exigir en un contrato.
CWE → catálogo de tipos de falla individuales (CWE-89, CWE-79...).
CVE → una falla concreta en un producto concreto (CVE-2021-44228, Log4Shell).
```

### OWASP Top 10 2021

La **edición 2021 del OWASP Top 10** es la versión publicada en septiembre de 2021, la que todavía citan la mayoría de cursos, certificaciones y herramientas. Las diez categorías, en orden:

1. **A01:2021 Broken Access Control.** El servidor no comprueba que el usuario tenga permiso para el recurso o la acción que pide. Ejemplo: la URL `/factura?id=1001` muestra tu factura; cambiar el número a `1002` muestra la de otra persona, porque el servidor solo verifica que hay sesión, no de quién es la factura. A eso se le llama IDOR (Insecure Direct Object Reference). Subió del puesto 5 al 1: el 94 % de las aplicaciones probadas tenía alguna forma de esta falla. Mitigación: denegar por defecto, comprobar la propiedad del objeto en el servidor en cada petición, nunca confiar en que el botón esté oculto en la interfaz.
2. **A02:2021 Cryptographic Failures.** Datos sensibles sin cifrar o cifrados mal: HTTP sin TLS, contraseñas con MD5 sin sal, claves escritas en el código, TLS 1.0. Antes se llamaba "Sensitive Data Exposure", que era el síntoma; el nuevo nombre señala la causa. Mitigación: TLS 1.2 o 1.3 en todo, contraseñas con bcrypt, scrypt o Argon2, AES-256-GCM para datos en reposo, claves en un gestor de secretos (ver [`10-criptografia`](../10-criptografia/)).
3. **A03:2021 Injection.** Datos del usuario que llegan a un intérprete mezclados con el código: SQL injection, OS command injection, LDAP injection y, desde 2021, también Cross-Site Scripting. Se desarrolla abajo en SQL Injection y XSS. Mitigación: consultas parametrizadas, escape según el contexto de salida, validación por lista blanca.
4. **A04:2021 Insecure Design.** Categoría nueva en 2021: fallas que ningún parche de código arregla porque están en el diseño. Ejemplo: la recuperación de contraseña pregunta "¿nombre de tu primera mascota?", dato que está en redes sociales; o un cine que permite reservar 600 butacas sin pagar y bloquea la sala entera. Mitigación: modelado de amenazas antes de programar (ver [`13-frameworks-de-amenazas`](../13-frameworks-de-amenazas/)), límites de negocio, casos de abuso en los requisitos.
5. **A05:2021 Security Misconfiguration.** El software es correcto pero está mal configurado: cuentas por defecto (`admin/admin`), listado de directorios activado, mensajes de error con la traza completa, buckets en la nube públicos, cabeceras de seguridad ausentes. Incluye XXE (XML External Entities), el abuso de un parser XML que resuelve entidades externas. Mitigación: plantillas de hardening repetibles, quitar lo que no se usa, revisar la configuración en cada despliegue (ver [`14-defensa-y-hardening`](../14-defensa-y-hardening/)).
6. **A06:2021 Vulnerable and Outdated Components.** La aplicación usa librerías, frameworks o servidores con vulnerabilidades conocidas. Caso típico: Log4Shell (CVE-2021-44228, CVSS 10,0) en la librería Log4j de Java. Mitigación: inventario de dependencias (SBOM, Software Bill of Materials), escaneo automático con herramientas como OWASP Dependency-Check o `npm audit`, parches con plazos.
7. **A07:2021 Identification and Authentication Failures.** Fallas al confirmar quién es el usuario: permitir contraseñas débiles, no frenar la fuerza bruta, IDs de sesión en la URL, sesiones que no expiran al cerrar sesión. Mitigación: MFA, bloqueo o retraso tras intentos fallidos, regenerar el ID de sesión tras el login (ver [`08-autenticacion`](../08-autenticacion/)).
8. **A08:2021 Software and Data Integrity Failures.** Se confía en código o datos sin verificar que no fueron alterados: actualizaciones sin firma, pipelines de CI/CD que bajan scripts de cualquier sitio, deserialización de objetos que vienen del cliente. El caso SolarWinds (2020) es el ejemplo clásico: una actualización legítima y firmada llevaba código malicioso inyectado en el build. Mitigación: firmas digitales, verificación de hashes, deserializar solo formatos de datos simples como JSON con esquema.
9. **A09:2021 Security Logging and Monitoring Failures.** No se registran los eventos importantes (logins fallidos, cambios de permisos) o nadie los mira. Sin registros, un intruso puede estar meses dentro sin que nadie lo note. Mitigación: registrar eventos de seguridad con hora y usuario, enviarlos a un SIEM, alertas con responsable (ver [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/)).
10. **A10:2021 Server-Side Request Forgery (SSRF).** La aplicación descarga una URL que le da el usuario (por ejemplo, "importar imagen desde URL") y el atacante le da una URL interna. Como la petición sale del servidor, llega a sitios que desde Internet no se ven: el servicio de metadatos de la nube en `169.254.169.254`, que entrega credenciales temporales, o paneles de administración internos. Entró por la encuesta, no por los datos. Mitigación: lista blanca de destinos, bloquear rangos privados y de enlace local tras resolver el DNS, IMDSv2 en AWS (ver [`18-cloud`](../18-cloud/)).

Analogía de toda la lista: A01 es un portero que deja pasar a cualquiera con pulsera sin mirar de qué fiesta es; A03 es un empleado que lee en voz alta, como orden, todo lo que pone en una nota que le pasó un cliente; A05 es un edificio bueno con la puerta trasera abierta; A06 es una cerradura de un modelo que todos saben abrir.

> [!TIP]
> Para el examen, memoriza el orden con la regla "AC, Crypto, Inject, Design, Config, Components, Auth, Integrity, Logging, SSRF". Las tres nuevas de 2021 son A04 Insecure Design, A08 Software and Data Integrity Failures y A10 SSRF.

```
A01 Broken Access Control → el servidor no verifica permisos por objeto o acción.
A02 Cryptographic Failures → datos sensibles sin cifrar o con criptografía débil.
A03 Injection → datos del usuario ejecutados como código; incluye XSS.
A04 Insecure Design → falla de diseño; ningún parche la arregla.
A05 Security Misconfiguration → software correcto mal configurado; incluye XXE.
A06 Vulnerable and Outdated Components → dependencias con CVE conocidos.
A07 Identification and Authentication Failures → login, contraseñas y sesiones débiles.
A08 Software and Data Integrity Failures → código o datos no verificados; deserialización insegura.
A09 Security Logging and Monitoring Failures → sin registros o sin nadie que los mire.
A10 SSRF → el servidor hace peticiones a destinos que elige el atacante.
```

### OWASP Top 10 2025

La **edición 2025 del OWASP Top 10** es la octava entrega de la lista, construida con datos de más de 2,8 millones de aplicaciones y con el mismo método de ocho categorías por datos y dos por encuesta. Las diez categorías, en orden:

1. **A01:2025 Broken Access Control.** Sigue primero. Absorbe SSRF, que en 2021 era A10: una petición forjada desde el servidor es, en el fondo, el servidor accediendo a algo que el usuario no debería alcanzar.
2. **A02:2025 Security Misconfiguration.** Sube del 5 al 2, porque cada vez más comportamiento de las aplicaciones depende de configuración (nube, contenedores, feature flags) y no de código.
3. **A03:2025 Software Supply Chain Failures.** Amplía A06:2021 Vulnerable and Outdated Components: ya no es solo "una librería vieja", sino cualquier compromiso en la cadena que va de las dependencias al sistema de build y a la distribución (paquetes maliciosos con nombres parecidos, cuentas de mantenedores robadas, servidores de build comprometidos). Entró por la encuesta.
4. **A04:2025 Cryptographic Failures.** Baja del 2 al 4; mismo contenido que en 2021.
5. **A05:2025 Injection.** Baja del 3 al 5, pero sigue siendo la categoría con más CVE asociados. Va desde XSS (muy frecuente, impacto menor) hasta SQL injection (menos frecuente, impacto alto).
6. **A06:2025 Insecure Design.** Baja del 4 al 6; OWASP lo atribuye a que el modelado de amenazas se volvió más común.
7. **A07:2025 Authentication Failures.** Mismo puesto, nombre más corto (antes "Identification and Authentication Failures").
8. **A08:2025 Software or Data Integrity Failures.** Mismo puesto. Ahora se centra en la verificación de integridad a nivel de artefactos y datos concretos, mientras que A03 cubre la cadena entera.
9. **A09:2025 Security Logging and Alerting Failures.** Mismo puesto; cambia "Monitoring" por "Alerting" para insistir en que un buen log sin alerta casi no sirve.
10. **A10:2025 Mishandling of Exceptional Conditions.** Categoría nueva, con 24 CWE: errores mal manejados, fallos lógicos y, sobre todo, sistemas que fallan abiertos. Ejemplo: si el servicio que valida permisos no responde y el código atrapa la excepción y continúa como si la respuesta fuera "permitido", cualquier caída se convierte en acceso libre. Mitigación: fail closed (ante la duda, denegar), manejar cada excepción de forma explícita, no mostrar trazas al usuario.

Analogía: es la misma lista de causas de incendio publicada cuatro años después. Algunas causas bajan porque la gente aprendió (los detectores de humo se hicieron obligatorios), otras suben porque aparecieron aparatos nuevos en las casas.

Ejemplo de A10 con código. Lo vulnerable y lo corregido, en pseudocódigo Python:

```python
# Vulnerable: fail open. Any exception means "allowed".
def can_access(user, resource):
    try:
        return authz_service.check(user, resource)
    except Exception:
        return True

# Fixed: fail closed, keep the cause in the log.
def can_access(user, resource):
    try:
        return authz_service.check(user, resource)
    except AuthzUnavailable as err:
        log.error("authz check failed for %s: %s", user.id, err)
        return False
```

Muchos exámenes (Security+ SY0-701 incluido) y herramientas siguen citando la numeración 2021. Cuando leas "A03 Injection" o "A10 SSRF", comprueba el año antes de corregir a alguien.

```
2021 → 2025
A01 Broken Access Control → A01 (y absorbe A10:2021 SSRF).
A02 Cryptographic Failures → A04.
A03 Injection → A05.
A04 Insecure Design → A06.
A05 Security Misconfiguration → A02.
A06 Vulnerable and Outdated Components → A03 Software Supply Chain Failures (ampliada).
A07 Identification and Authentication Failures → A07 Authentication Failures.
A08 Software and Data Integrity Failures → A08 Software or Data Integrity Failures.
A09 Security Logging and Monitoring Failures → A09 Security Logging and Alerting Failures.
(nueva) → A10 Mishandling of Exceptional Conditions.
```

## Common Attacks

### DoS vs DDoS

Un **DoS** (Denial of Service) es un ataque que agota un recurso del objetivo (ancho de banda, conexiones, CPU, memoria) para que los usuarios legítimos no puedan usarlo; un **DDoS** (Distributed Denial of Service) es el mismo ataque lanzado desde miles de máquinas a la vez, normalmente una botnet de equipos infectados.

Por qué importa: ataca la disponibilidad, la A de la tríada CIA (ver [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/)). No roba nada, pero una tienda caída una hora en Black Friday pierde ventas reales. La diferencia entre DoS y DDoS no es de técnica sino de escala y de defensa: un DoS sale de una o pocas IP y se bloquea filtrando esas IP; un DDoS sale de decenas de miles de IP repartidas por el mundo, muchas de ellas routers domésticos y cámaras IoT, y filtrar por IP ya no sirve.

Cómo funciona, por la capa que agota:

- Volumétrico (capa 3/4): llena el enlace. Se mide en Gbps. Si tu conexión es de 10 Gbps y llegan 100 Gbps de basura, el tráfico bueno no cabe, da igual lo potente que sea el servidor. Variante clave: la **amplificación**, donde el atacante manda consultas pequeñas a servidores públicos (DNS, NTP, memcached) con la IP de origen falsificada como la de la víctima; las respuestas, mucho más grandes, llegan todas a la víctima. CISA documenta factores de hasta 28-54 veces en DNS, 556 en NTP y más de 10 000 en memcached.
- De protocolo (capa 4): agota tablas de estado. Se mide en paquetes por segundo. El SYN flood manda miles de `SYN` y nunca completa el handshake; el servidor guarda cada conexión medio abierta en la cola hasta que se llena (ver el handshake en [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/)).
- De aplicación (capa 7): peticiones HTTP que parecen normales pero son caras, como búsquedas complejas o generación de PDF. Se mide en peticiones por segundo. Con 5000 rps a una búsqueda que tarda 200 ms en la base de datos, el servidor queda saturado aunque el ancho de banda esté casi vacío.

```
DoS                          DDoS (con amplificación)
atacante ──► víctima         atacante ──(consultas pequeñas, origen falso = víctima)──► 1000 resolvers abiertos
                                                                                          │
1 origen, fácil de filtrar                       respuestas grandes ×50 ◄─────────────────┘
                                                        │
                                                        ▼
                                                     víctima (recibe de 1000 IP legítimas)
```

Analogía: un DoS es una persona que llama sin parar al teléfono de una pizzería para que nadie más pueda pedir; un DDoS es mil personas llamando a la vez desde mil teléfonos distintos, y bloquear un número ya no sirve.

Ejemplo de detección de un SYN flood en un servidor Linux:

```
$ ss -s
Total: 8312
TCP:   8120 (estab 95, closed 12, orphaned 0, timewait 9)

$ ss -tan state syn-recv | wc -l
7984
```

Unas 8000 conexiones en `SYN-RECV` contra 95 establecidas es la firma de un SYN flood: la cola está llena de handshakes que nunca terminan.

Mitigación: SYN cookies (`net.ipv4.tcp_syncookies = 1`, el servidor no guarda estado hasta recibir el `ACK` final), rate limiting por IP y por ruta, un CDN o servicio de scrubbing que absorba el volumen antes de que llegue a tu enlace, Anycast para repartir la carga, y del lado de Internet el filtrado de IP de origen falsificada (BCP 38) que impide la amplificación. Para capa 7, caché y WAF con desafíos.

Criterio de examen: si la pregunta habla de "muchas fuentes" o "botnet", es DDoS; si habla de respuestas más grandes que las peticiones con origen falsificado, es un ataque de amplificación o reflexión.

```
DoS → una fuente; se filtra por IP.
DDoS → miles de fuentes (botnet); requiere absorber el volumen aguas arriba.
Volumétrico → satura el enlace; se mide en Gbps.
De protocolo → satura tablas de estado (SYN flood); se mide en pps.
De aplicación → peticiones caras en capa 7; se mide en rps.
Amplificación → respuesta mucho mayor que la consulta, dirigida a la víctima con IP falsificada.
```

### MITM

Un **MITM** (Man-in-the-Middle, también llamado on-path attack) es un ataque en el que el atacante se coloca en el camino entre dos partes que creen hablar directamente, de modo que puede leer, modificar o inyectar lo que se dicen.

Por qué existe: muchos protocolos de red no comprueban quién está al otro lado. ARP acepta cualquier respuesta, DHCP acepta al primer servidor que contesta, un cliente Wi-Fi se conecta al SSID que conoce sin saber si es el AP real. Si además el tráfico va en claro (HTTP, Telnet, FTP), el intermediario lo ve todo.

Cómo se llega a estar "en el medio":

- ARP poisoning: el atacante en la misma LAN envía respuestas ARP falsas diciendo "la IP del gateway tiene mi MAC". El tráfico de la víctima hacia Internet pasa por él (detalle en Spoofing).
- Rogue DHCP: responde antes que el servidor real y se anuncia como gateway y DNS.
- Evil Twin: un AP falso con el mismo SSID (sección propia abajo).
- DNS poisoning: la víctima resuelve el nombre del banco a la IP del atacante (sección propia abajo).
- Una vez en el medio, intenta bajar la conexión a texto plano (SSL stripping) o presentar un certificado falso.

```
Sin ataque:   víctima ─────────────────────────► gateway ──► Internet

ARP poisoning:
              víctima ──► atacante ──► gateway ──► Internet
                 ▲            │
                 └────────────┘  (reenvía todo para no levantar sospechas, pero lee y puede cambiar)
```

Analogía: es el cartero que abre tus cartas, las lee, cambia el número de cuenta de una factura y las vuelve a cerrar antes de entregarlas.

Ejemplo: en una cafetería, el portátil de un cliente navega por `http://` a un foro. Un atacante en la misma red hace ARP poisoning y ve el usuario y la contraseña en el cuerpo del `POST`. Si el sitio hubiera usado HTTPS con HSTS, el atacante solo vería tráfico cifrado y el navegador se negaría a aceptar su certificado falso.

Mitigación: cifrado con autenticación del servidor en todo (TLS 1.2/1.3, SSH, VPN), HSTS para que el navegador nunca use HTTP con ese dominio, no aceptar advertencias de certificado, y en la LAN Dynamic ARP Inspection y DHCP snooping en los switches. Detección: un IDS que alerte de cambios de MAC para la IP del gateway.

> [!WARNING]
> HTTPS solo protege si el certificado se valida. Un usuario que hace clic en "continuar de todos modos" ante un aviso de certificado le entrega la conexión al intermediario.

```
MITM / on-path → el atacante queda entre dos partes y lee o modifica el tráfico.
SSL stripping → baja la conexión de HTTPS a HTTP para leerla.
HSTS → cabecera que obliga al navegador a usar solo HTTPS con ese dominio.
Eavesdropping (sniffing) → solo escucha; no necesariamente está en el camino ni modifica.
```

### CSRF

El **CSRF** (Cross-Site Request Forgery) es un ataque que hace que el navegador de un usuario autenticado envíe, sin que él lo sepa, una petición que cambia algo en un sitio donde tiene sesión abierta, aprovechando que el navegador adjunta las cookies de ese sitio de forma automática.

Por qué funciona: el servidor reconoce al usuario por la cookie de sesión, y el navegador la manda en toda petición a ese dominio, venga de donde venga el clic o el formulario. Si la acción "transferir dinero" solo exige la cookie, cualquier página que el usuario visite puede disparar esa petición. El atacante no ve la respuesta (la same-origin policy lo impide); solo necesita que la acción ocurra.

```
1. Usuario inicia sesión en banco.com        → navegador guarda cookie de sesión
2. Usuario visita foro-malicioso.com         → la página contiene un formulario oculto
3. El formulario se envía solo a banco.com   → el navegador adjunta la cookie de banco.com
4. banco.com ve una petición válida con sesión válida → ejecuta la transferencia
```

Analogía: es como dejar un cheque firmado en blanco en tu escritorio; alguien que pasa escribe el monto y el banco lo cobra porque la firma es auténtica.

Ejemplo con código, vulnerable y corregido (Flask/Jinja, ilustrativo). La versión vulnerable acepta cualquier `POST` con sesión válida; una página de otro dominio con un formulario que apunte a `/transfer` con monto y cuenta de destino rellenos lograría la transferencia.

```python
# Vulnerable: session cookie is the only check.
@app.post("/transfer")
def transfer():
    require_login()
    do_transfer(session["user_id"], request.form["to"], request.form["amount"])
    return redirect("/ok")
```

```python
# Fixed: per-session anti-CSRF token compared in constant time.
@app.post("/transfer")
def transfer():
    require_login()
    sent = request.form.get("csrf_token", "")
    if not hmac.compare_digest(sent, session["csrf_token"]):
        abort(403)
    do_transfer(session["user_id"], request.form["to"], request.form["amount"])
    return redirect("/ok")
```

```html
<!-- The legitimate form carries the token; another site cannot read it. -->
<form method="post" action="/transfer">
  <input type="hidden" name="csrf_token" value="{{ session.csrf_token }}">
  ...
</form>
```

Y la cookie de sesión con el atributo que el navegador usa para no mandarla en peticiones cruzadas:

```
Set-Cookie: session=7f3a...; Secure; HttpOnly; SameSite=Lax
```

Mitigación: token anti-CSRF impredecible por sesión o por formulario (el atacante no puede leerlo desde otro origen), cookies `SameSite=Lax` o `Strict`, no cambiar estado con `GET`, comprobar las cabeceras `Origin` o `Sec-Fetch-Site`, y pedir reautenticación en acciones críticas (cambiar correo, transferir).

```
CSRF → el navegador de la víctima hace una petición que ella no quiso; el atacante no lee la respuesta.
XSS → el atacante ejecuta su script dentro del sitio; puede leer y hacer todo lo que el usuario puede.
Token anti-CSRF → valor secreto en el formulario que otro origen no puede conocer.
SameSite → atributo de cookie que impide enviarla en peticiones iniciadas desde otro sitio.
```

### Spoofing

El **spoofing** es una técnica de suplantación que falsifica un identificador (dirección IP, MAC, respuesta ARP, remitente de correo, número de teléfono, nombre DNS) para que el receptor crea que el mensaje viene de una fuente de confianza.

Por qué funciona: la mayoría de protocolos antiguos fueron diseñados para redes donde todos confiaban en todos. Un paquete IP lleva la IP de origen que el emisor quiera escribir; SMTP acepta cualquier `From:`; ARP no autentica las respuestas. El spoofing casi nunca es el ataque final: es el paso que habilita otro (MITM, amplificación DDoS, phishing, saltarse un filtro por IP).

Tipos que pide el examen:

- IP spoofing: se falsifica la IP de origen. Sirve para ataques donde no hace falta ver la respuesta (amplificación, SYN flood). No sirve para una sesión TCP completa desde otra red, porque las respuestas van a la IP real.
- MAC spoofing: se cambia la MAC de la tarjeta para saltar un filtrado por MAC o suplantar un equipo autorizado.
- ARP spoofing (ARP poisoning): se envían respuestas ARP no pedidas que asocian la IP de otro (normalmente el gateway) con la MAC del atacante. Es la base del MITM en LAN.
- Email spoofing: se falsifica el remitente. Se combate con SPF (qué servidores pueden enviar por el dominio), DKIM (firma criptográfica del mensaje) y DMARC (política de qué hacer si fallan).
- Caller ID spoofing: se falsifica el número que aparece en una llamada (vishing, ver [`11-ataques-y-amenazas`](../11-ataques-y-amenazas/)).

Analogía: es escribir en el sobre la dirección del remitente que te dé la gana; Correos lo entrega igual, porque nadie comprueba que viva ahí.

Ejemplo de detección de ARP spoofing: dos IP distintas de la LAN apuntan a la misma MAC, y una de ellas es el gateway.

```
$ ip neigh
192.168.1.1   dev wlan0 lladdr 3c:52:82:aa:bb:cc REACHABLE
192.168.1.50  dev wlan0 lladdr 3c:52:82:aa:bb:cc REACHABLE
```

La MAC `3c:52:82:aa:bb:cc` es la del equipo `.50`; que el gateway `.1` "tenga" la misma MAC indica que alguien está respondiendo por él.

Mitigación: Dynamic ARP Inspection y DHCP snooping en los switches, ARP estático para el gateway en equipos críticos, 802.1X para que solo entren equipos autenticados, filtrado de origen BCP 38 en los routers de borde, SPF/DKIM/DMARC con política `p=reject` en el correo, y sobre todo protocolos con autenticación criptográfica en lugar de confiar en direcciones.

```
IP spoofing → IP de origen falsa; útil sin respuesta (DDoS, amplificación).
MAC spoofing → MAC falsa; salta filtros por MAC.
ARP spoofing → asocia la IP de otro con tu MAC; base del MITM en LAN.
Email spoofing → remitente falso; se combate con SPF, DKIM y DMARC.
DNS spoofing → respuesta DNS falsa; ver DNS Poisoning.
```

### Una dirección no es una identidad

> [!IMPORTANT]
> Una IP, una MAC, un SSID o un remitente se pueden escribir a mano, así que no prueban quién habla. Solo la criptografía (un certificado validado, una firma, una clave compartida) autentica al otro extremo; por eso la defensa común de spoofing, MITM, Evil Twin y DNS poisoning es la misma.

La idea es que todos los ataques de red de esta nota que "suplantan" algo explotan el mismo hueco: un protocolo que decide en quién confiar mirando un campo que el emisor controla. Cambia el campo, no el mecanismo.

```
                  Spoofing ARP     Evil Twin        DNS Poisoning     MITM
Qué se falsifica  MAC del gateway  SSID del AP      IP de un nombre   la ruta entera
Capa              2                2 (Wi-Fi)        7 (DNS)           2 a 7
Qué confía        tabla ARP        cliente Wi-Fi    caché del resolver ambas partes
Defensa que corta TLS / 802.1X     WPA3-Ent + cert  DNSSEC + TLS      TLS validado
```

La última fila tiene el mismo ingrediente en las cuatro columnas: autenticación criptográfica del otro extremo. Ahí está la idea entera.

Analogía: un uniforme de policía se puede comprar; la placa con número verificable en la comisaría, no. Confiar en el uniforme es confiar en la dirección; llamar a la comisaría es validar el certificado.

Límite: el cifrado autenticado protege la confidencialidad y la integridad, pero no la disponibilidad. Un atacante que hace ARP spoofing puede seguir tirando el tráfico aunque no pueda leerlo, y un deauth sigue desconectando aunque la red use WPA2.

```
Dirección (IP, MAC, SSID, From) → la escribe el emisor; no autentica.
Autenticación criptográfica → prueba la identidad con una clave que el atacante no tiene.
```

### SQL Injection

La **SQL injection** (SQLi) es una vulnerabilidad de inyección en la que un dato que envía el usuario se pega dentro de una sentencia SQL como texto, de modo que el motor de base de datos lo interpreta como parte de la consulta y no como un valor.

Por qué existe: es la forma más directa de construir una consulta, concatenar cadenas, y funciona perfectamente con datos normales. El problema aparece cuando el dato contiene caracteres que tienen significado para SQL (comillas, operadores, comentarios). Es CWE-89, está dentro de A03:2021 / A05:2025 Injection, y su impacto va desde saltarse un login hasta leer la base de datos entera o borrarla.

Cómo funciona por dentro: el motor recibe un único texto y lo analiza. Si el texto se armó pegando la entrada del usuario, el motor no puede distinguir qué parte escribió el programador y qué parte el usuario. Una comilla en la entrada cierra la cadena que el programador abrió, y lo que viene después se lee como SQL. Las variantes, de más a menos visible:

- In-band (clásica): el resultado de la consulta alterada aparece en la página, a veces combinando con `UNION` los datos de otra tabla.
- Error-based: el atacante provoca errores de la base de datos y lee datos en los mensajes de error que la aplicación muestra.
- Blind (ciega): la página no muestra datos ni errores, pero cambia (boolean-based) o tarda más (time-based) según si una condición es verdadera, y el atacante extrae la información bit a bit.

Analogía: es como un formulario de banco que dice "Pague a: ______ la cantidad de 100 €". Si el cajero copia lo que pone en la línea sin mirar, alguien escribe en ella "Juan, y además vacíe la cuenta" y el cajero lo ejecuta como una sola orden.

Ejemplo con código, vulnerable y corregido (Python con un driver de base de datos). En la versión vulnerable, si en el campo usuario alguien escribe una comilla seguida de una condición que siempre es verdadera y un marcador de comentario, el `WHERE` deja de comprobar la contraseña y devuelve el primer usuario de la tabla, que suele ser el administrador.

```python
# Vulnerable: user input concatenated into the SQL text.
def login(db, username, password_hash):
    sql = ("SELECT id FROM users WHERE name = '" + username +
           "' AND pw_hash = '" + password_hash + "'")
    return db.execute(sql).fetchone()
```

```python
# Fixed: parameterized query. The SQL text is fixed; values travel separately.
LOGIN_SQL = "SELECT id FROM users WHERE name = %s AND pw_hash = %s"

def login(db, username, password_hash):
    return db.execute(LOGIN_SQL, (username, password_hash)).fetchone()
```

En la versión corregida el motor compila la sentencia primero y recibe los valores después, como datos. Aunque el usuario escriba comillas, el motor busca literalmente un usuario con ese nombre raro y no lo encuentra.

Señal en los logs de una SQLi en curso: muchos errores de sintaxis SQL seguidos desde una misma IP.

```
2026-10-02 10:14:03 ERROR  db: syntax error at or near "'" (route=/login, ip=203.0.113.7)
2026-10-02 10:14:04 ERROR  db: unterminated quoted string (route=/login, ip=203.0.113.7)
2026-10-02 10:14:06 ERROR  db: syntax error at or near "UNION" (route=/search, ip=203.0.113.7)
```

Mitigación, en orden de eficacia: consultas parametrizadas (prepared statements) o un ORM que las use; validación por lista blanca para lo que no puede ir como parámetro (nombres de columna en un `ORDER BY`: elegir entre valores fijos); mínimo privilegio en la cuenta de base de datos (la app no necesita `DROP`); no mostrar errores de base de datos al usuario; WAF como capa extra, nunca como única defensa.

> [!WARNING]
> Escapar comillas a mano o filtrar palabras como `SELECT` no es una defensa: siempre hay una codificación o un contexto que se escapa del filtro. La única corrección de raíz es separar código y datos con parámetros.

```
SQL injection → la entrada del usuario cambia la estructura de una consulta SQL.
In-band → el resultado sale en la propia respuesta.
Blind → no sale nada; se deduce por cambios o retrasos en la respuesta.
Consulta parametrizada → el texto SQL es fijo y los valores viajan aparte.
```

### XSS

El **XSS** (Cross-Site Scripting) es una vulnerabilidad de inyección en la que la aplicación incluye datos del usuario en una página sin codificarlos, de modo que el navegador de otra persona los ejecuta como código JavaScript dentro del origen del sitio vulnerable.

Por qué es grave: el script inyectado corre con todos los permisos del sitio. Puede leer cookies que no tengan `HttpOnly`, leer lo que hay en la página (datos personales, tokens anti-CSRF), hacer peticiones en nombre del usuario, cambiar el contenido para mostrar un login falso o registrar teclas. Es CWE-79 y, por número de casos, la inyección más frecuente.

Los tres tipos, según dónde vive el dato malicioso:

- Reflected (reflejado): el dato va en la petición (un parámetro de la URL) y el servidor lo devuelve en la respuesta en ese mismo momento. Hace falta que la víctima abra un enlace preparado. Ejemplo típico: una página de búsqueda que muestra "Resultados para: <lo que buscaste>".
- Stored (almacenado o persistente): el dato se guarda en el servidor (un comentario, un nombre de perfil) y se sirve a todos los que visitan esa página. Es el más peligroso porque no requiere engañar a nadie con un enlace: basta con visitar la página.
- DOM-based: el servidor nunca ve el dato malicioso; es el propio JavaScript del sitio el que toma algo controlable por el usuario (el fragmento `#` de la URL, `location.search`, `postMessage`) y lo escribe en la página con una función que interpreta HTML (`innerHTML`, `document.write`).

```
Reflected:  atacante ──enlace──► víctima ──petición con dato──► servidor ──lo devuelve──► navegador lo ejecuta
Stored:     atacante ──comentario──► servidor (BD)    víctima ──visita──► servidor ──página con dato──► ejecuta
DOM:        víctima abre URL#dato ──► JS del sitio lee location.hash ──► innerHTML ──► ejecuta (servidor no ve nada)
```

Analogía: es como un tablón de anuncios de un edificio donde el portero pega todo lo que le dan sin leerlo. Si alguien entrega un papel que dice "a todos los vecinos: dejen las llaves en portería", los vecinos lo obedecen porque está en el tablón oficial.

Ejemplo con código, vulnerable y corregido. En la versión vulnerable del lado servidor, si el parámetro `q` contiene una etiqueta `<script>` o un atributo de evento como `onerror` dentro de una etiqueta de imagen, el navegador lo ejecuta.

```python
# Vulnerable (reflected): raw input inserted into HTML.
@app.get("/search")
def search():
    q = request.args.get("q", "")
    return "<p>Results for: " + q + "</p>"
```

```python
# Fixed: HTML-encode on output (or use a template engine with autoescape on).
from markupsafe import escape

@app.get("/search")
def search():
    q = request.args.get("q", "")
    return "<p>Results for: " + str(escape(q)) + "</p>"
```

Con el escape, `<` se convierte en `&lt;` y `>` en `&gt;`: el navegador dibuja los símbolos en pantalla en vez de crear una etiqueta.

Para el tipo DOM, la corrección está en el JavaScript del cliente:

```javascript
// Vulnerable (DOM-based): URL fragment parsed as HTML.
const name = decodeURIComponent(location.hash.slice(1));
document.getElementById("greeting").innerHTML = "Hello " + name;

// Fixed: textContent never parses HTML.
document.getElementById("greeting").textContent = "Hello " + name;
```

Y una capa de defensa en profundidad, la cabecera CSP, que le dice al navegador que solo ejecute scripts del propio sitio y nunca scripts en línea:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'
Set-Cookie: session=7f3a...; Secure; HttpOnly; SameSite=Lax
```

Mitigación: codificar la salida según el contexto (HTML, atributo, JavaScript, URL; cada uno tiene su escape), motores de plantillas con autoescape activado, `textContent` en lugar de `innerHTML`, sanitizar con una librería probada (DOMPurify) cuando de verdad hay que aceptar HTML, CSP, y cookies `HttpOnly` para que un XSS no pueda leer la sesión.

```
Reflected XSS → el dato va en la petición y vuelve en la respuesta; requiere un enlace.
Stored XSS → el dato se guarda en el servidor y afecta a todos los visitantes.
DOM-based XSS → el JS del cliente escribe el dato como HTML; el servidor no lo ve.
Output encoding → convierte caracteres especiales en entidades para que se muestren, no se ejecuten.
CSP → cabecera que limita de dónde puede cargar y ejecutar scripts el navegador.
HttpOnly → la cookie no es accesible desde JavaScript.
```

### Toda inyección es datos tratados como código

> [!IMPORTANT]
> SQL injection, XSS, command injection y path traversal son el mismo error: un dato del usuario llega a un intérprete pegado al código, y el intérprete no sabe dónde termina uno y empieza el otro. La corrección siempre es mantenerlos separados: parámetros, codificación de salida o una API que no interprete.

Una inyección es una falla de diseño de interfaz entre el programa y un intérprete que ocurre cuando ambos comparten un único canal de texto para instrucciones y datos. Cambia el intérprete y cambian los caracteres peligrosos, pero el mecanismo es idéntico.

```
                       SQL Injection        XSS                   Command Injection      Directory Traversal
Intérprete             motor SQL            navegador (HTML/JS)   shell del SO           sistema de archivos
Caracteres peligrosos  ' ; --               < > " '               ; | & $()              ../  y separadores
Separación correcta    consulta param.      escape de salida      exec con lista args    resolver ruta y comparar con base
Raíz común             dato concatenado al código del intérprete
```

La última fila es la misma en las cuatro columnas: ahí está la idea entera.

Analogía: un intérprete humano que traduce en voz alta todo lo que lee en una hoja, sin distinguir entre lo que dijo el jefe y lo que alguien escribió al margen.

Límite: validar la entrada (lista blanca de formatos) ayuda, pero no sustituye a la separación. Un nombre como "O'Brien" es legítimo y lleva una comilla; la validación no puede rechazarlo, y la consulta parametrizada lo maneja sin problema.

```
Inyección → dato y código comparten canal; el intérprete ejecuta el dato.
Separación de código y datos → la defensa de raíz para todas las variantes.
Validación de entrada → defensa complementaria; no reemplaza a la separación.
```

### Evil Twin

Un **Evil Twin** es un punto de acceso Wi-Fi falso que copia el SSID (y a veces el BSSID) de una red legítima para que los clientes se conecten a él creyendo que es la red real, y así ponerse en medio de su tráfico.

Por qué funciona: un cliente Wi-Fi recuerda redes por su nombre, no por una identidad verificable. Si ve dos AP con el mismo SSID, elige normalmente el de señal más fuerte. En redes abiertas o con WPA2-Personal (una contraseña compartida que el atacante conoce, como la del Wi-Fi de una cafetería) no hay forma de que el cliente distinga el AP real del falso. A menudo se combina con un ataque de deauth para echar a los clientes del AP real y forzar la reconexión, y con un portal cautivo falso que pide credenciales.

```
          AP real "Cafe_WiFi"   (-70 dBm, lejos)
                 ▲  x desconectado por deauth
  portátil ──────┘
     │
     └──────────► Evil Twin "Cafe_WiFi"  (-40 dBm, en la mesa de al lado) ──► Internet
                  ve DNS, HTTP en claro, portal de login falso
```

Analogía: alguien se pone en la puerta de un restaurante con un delantal igual al de los camareros y toma los pedidos y el dinero de los clientes que llegan.

Ejemplo: en un aeropuerto, un atacante emite "Airport_Free_WiFi" con más potencia que el AP oficial. 30 viajeros se conectan, les aparece una página que pide "inicia sesión con tu correo para continuar", y 5 la rellenan.

Mitigación: WPA2/WPA3-Enterprise (802.1X con EAP-TLS o PEAP) con validación del certificado del servidor RADIUS configurada en los clientes, de modo que un AP falso no puede probar su identidad; WPA3-Personal (SAE) impide descifrar el tráfico de otros pero no impide un gemelo si se conoce la contraseña; WIDS/WIPS que detecte BSSID desconocidos anunciando el SSID corporativo; VPN y HTTPS con HSTS en redes públicas; desactivar la autoconexión a redes abiertas.

```
Evil Twin → AP del atacante que imita el SSID de una red legítima; objetivo: MITM o robo de credenciales.
Rogue AP → AP no autorizado conectado a la red interna; objetivo: puerta trasera a la LAN.
WIDS/WIPS → sistema que detecta (y bloquea) AP y comportamientos Wi-Fi anómalos.
```

### VLAN Hopping

El **VLAN Hopping** es un ataque de capa 2 en el que un equipo conectado a una VLAN consigue enviar tráfico a otra VLAN a la que no pertenece, saltándose la segmentación sin pasar por el router o firewall que debería filtrarlo.

Por qué importa: las VLAN se usan para separar redes (usuarios, servidores, VoIP, invitados) sin comprar switches distintos. Si un equipo de la VLAN de invitados llega a la VLAN de servidores, toda la política de segmentación cae. Hay dos técnicas:

- Switch spoofing: muchos switches Cisco traen los puertos en modo DTP (Dynamic Trunking Protocol) `dynamic auto` o `dynamic desirable`, que negocian un trunk si el otro extremo lo pide. Un equipo que habla DTP convierte su puerto de acceso en trunk y recibe y envía tráfico de todas las VLAN.
- Double tagging: el atacante está en la native VLAN del trunk (la VLAN cuyas tramas viajan sin etiqueta, la 1 por defecto). Envía una trama con dos etiquetas 802.1Q: la externa con la native VLAN y la interna con la VLAN destino. El primer switch quita la externa (porque la native va sin etiqueta) y reenvía por el trunk; el segundo switch lee la interna y entrega la trama en la VLAN destino. Es unidireccional: la respuesta no vuelve.

```
Double tagging (native VLAN = 1, víctima en VLAN 20)

Atacante (VLAN 1) ── [802.1Q:1][802.1Q:20][datos] ──► SW1
SW1: la externa es la native → la quita → envía por el trunk:  [802.1Q:20][datos]
SW2: lee 20 → entrega en VLAN 20 ──► víctima
```

Analogía: es un sobre dentro de otro sobre. La oficina de correos local abre el exterior porque va dirigido a ella, ve otro sobre con dirección y lo manda al barrio de destino sin preguntarse cómo llegó ahí.

Ejemplo de configuración defensiva en un switch Cisco: puertos de usuario forzados a acceso, sin DTP, y una native VLAN que nadie usa en los trunks.

```
interface range GigabitEthernet1/0/1 - 24
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
!
interface GigabitEthernet1/0/48
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30
!
vlan dot1q tag native
```

Verificación de que ningún puerto de usuario quedó en trunk:

```
SW1# show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Gi1/0/48    on           802.1q         trunking      999
```

Mitigación: `switchport mode access` y `switchport nonegotiate` en puertos de usuario, trunks estáticos, native VLAN dedicada y sin hosts (o etiquetar también la native), apagar los puertos no usados y meterlos en una VLAN muerta, limitar las VLAN permitidas en cada trunk.

```
Switch spoofing → el atacante negocia un trunk con DTP y accede a todas las VLAN.
Double tagging → dos etiquetas 802.1Q; abusa de la native VLAN; solo ida.
Native VLAN → la VLAN que viaja sin etiqueta en un trunk; por defecto la 1.
DTP → protocolo Cisco que negocia trunks automáticamente; se desactiva con nonegotiate.
```

### DNS Poisoning

El **DNS Poisoning** (DNS cache poisoning o DNS spoofing) es un ataque que mete una respuesta DNS falsa en la caché de un resolver o de un equipo, para que todos los que pregunten por un nombre reciban la IP que eligió el atacante mientras dure el TTL de esa entrada.

Por qué funciona: DNS clásico viaja por UDP 53 sin cifrar ni firmar. Un resolver que hace una pregunta acepta la primera respuesta que coincida en tres cosas: la pregunta, el puerto UDP de origen que usó y un identificador de transacción (TXID) de 16 bits. Si el atacante logra enviar una respuesta falsa que coincida antes de que llegue la verdadera, la caché queda envenenada.

Cómo se ataca la caché:

- Antes de 2008 el puerto de origen era fijo, así que bastaba con acertar el TXID: 65 536 valores posibles, y el atacante podía mandar miles de respuestas por segundo. Dan Kaminsky mostró en 2008 cómo repetir el intento sin esperar a que caduque el TTL, preguntando por subdominios aleatorios inexistentes.
- La defensa inmediata fue aleatorizar también el puerto de origen (RFC 5452): con unos 64 000 puertos y 65 536 TXID, hay alrededor de 4 000 millones de combinaciones.
- Otras rutas: modificar el archivo `hosts` de una víctima con malware, controlar un servidor DNS rogue que se entrega por DHCP, o comprometer la cuenta del registrador del dominio.

```
Resolver ──consulta (TXID=?, puerto=?)──► servidor autoritativo
   ▲                                              │
   │ respuesta falsa (acierta TXID y puerto) ◄── atacante (inunda de respuestas)
   │ respuesta real (llega tarde, se descarta) ◄─┘
   ▼
Caché: banco.com → 198.51.100.66 (TTL 86400)  → todos los clientes van al atacante 24 h
```

Analogía: es como cambiar el número de un restaurante en la guía telefónica del barrio. Quien lo busque durante ese año llamará al número falso, y la guía parece auténtica.

Ejemplo de comprobación defensiva con DNSSEC: la bandera `ad` (Authenticated Data) indica que el resolver validó las firmas de la respuesta.

```
$ dig +dnssec example.com A
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41204
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
example.com.     3600  IN  A      93.184.215.14
example.com.     3600  IN  RRSIG  A 13 2 3600 ...
```

Un registro `RRSIG` acompaña a la respuesta, y si la firma no cuadra el resolver validador devuelve `SERVFAIL` en lugar de la IP falsa.

Mitigación: DNSSEC (firma los registros con una cadena de confianza desde la raíz; ver RFC 4033), aleatorizar puerto y TXID (todos los resolvers modernos), no exponer resolvers recursivos abiertos a Internet, DNS over HTTPS o DNS over TLS entre cliente y resolver para impedir manipulación local, y TLS en el servicio final: aunque el DNS mienta, el atacante no tiene el certificado del banco.

```
DNS cache poisoning → respuesta falsa guardada en la caché del resolver; afecta a todos sus clientes.
DNS spoofing → respuesta falsa a un cliente concreto; a menudo se usa como sinónimo.
DNSSEC → firma los registros; garantiza autenticidad e integridad, no confidencialidad.
DoH / DoT → cifran el canal cliente-resolver; no firman los registros.
```

### Deauth Attack

Un **Deauth Attack** (ataque de desautenticación) es un ataque Wi-Fi que envía tramas de gestión 802.11 de tipo "deauthentication" falsificadas con la MAC del AP, para expulsar a uno o a todos los clientes de la red.

Por qué funciona: en 802.11 original las tramas de gestión (beacon, authentication, deauthentication, disassociation) viajan sin protección criptográfica, incluso en redes WPA2. Cualquiera en alcance de radio puede escribir una trama que diga "soy el AP, quedas desconectado" y el cliente la obedece. No hace falta conocer la contraseña.

Para qué se usa:

- Denegación de servicio: repetir deauths sin parar deja a los clientes sin red.
- Forzar la reconexión para capturar el handshake de 4 vías de WPA2-Personal, que luego se ataca offline con diccionario si la contraseña es débil.
- Empujar a los clientes hacia un Evil Twin.

```
AP (BSSID aa:aa)  ◄──── conectado ────  cliente
                                          ▲
atacante ── trama deauth "origen = aa:aa, motivo 7" ──┘   (sin firma: el cliente no puede comprobarla)
cliente se desconecta → intenta reconectar → (handshake capturable / se va al gemelo)
```

Analogía: es alguien que se pone el chaleco del acomodador del cine y le dice a una persona "tiene que salir de la sala". Como nadie pide credenciales a un acomodador, la persona sale.

Ejemplo de lo que ve un WIDS o un administrador que monitoriza el aire: cientos de deauth en segundos desde la MAC del AP, cuando un AP real casi nunca los envía.

```
10:20:01.112  Deauth  BSSID aa:aa:aa:11:22:33 -> ff:ff:ff:ff:ff:ff  reason 7
10:20:01.115  Deauth  BSSID aa:aa:aa:11:22:33 -> ff:ff:ff:ff:ff:ff  reason 7
... 480 tramas en 10 s ...
```

Mitigación: 802.11w PMF (Protected Management Frames), que firma las tramas de gestión para que el cliente descarte las falsas; WPA3 exige PMF, y en WPA2 se puede activar como opcional u obligatorio. WIDS para detectar ráfagas, contraseñas largas (o WPA3-SAE) para que capturar el handshake no sirva, y cable para equipos críticos. Un jammer de radio sigue siendo posible: ninguna configuración protege la capa física.

```
Deauth attack → tramas de gestión falsas que desconectan clientes; no requiere la clave.
Disassociation → trama hermana con el mismo abuso; también protegida por 802.11w.
802.11w / PMF → protege las tramas de gestión; obligatorio en WPA3.
Jamming → interferencia de radio pura; capa física, sin tramas.
```

### Replay Attack

Un **Replay Attack** (ataque de repetición) es un ataque en el que se captura un mensaje válido (un login, un token, una orden de transferencia, la señal de un mando) y se reenvía más tarde, tal cual, para que el receptor lo acepte otra vez como si fuera nuevo.

Por qué funciona: el atacante no necesita entender ni descifrar el mensaje. Si el receptor solo comprueba que el mensaje es auténtico (firma correcta, hash correcto, sesión válida) pero no que es fresco, una copia exacta pasa todas las comprobaciones. Ejemplos clásicos: un hash de contraseña enviado siempre igual, una cookie de sesión robada que no expira, el código fijo de un mando de garaje antiguo.

Cómo se hace un mensaje "no repetible", que es lo que hay que reconocer:

- Nonce: número aleatorio de un solo uso que el servidor genera y el cliente debe incluir; el servidor rechaza cualquier nonce ya visto.
- Marca de tiempo: el mensaje lleva la hora y se rechaza si es más viejo que una ventana. Kerberos usa 5 minutos por defecto, por eso exige relojes sincronizados (ver [`08-autenticacion`](../08-autenticacion/)).
- Número de secuencia: TLS, IPsec y WPA numeran cada registro o paquete y descartan duplicados.
- Códigos que cambian: TOTP cambia cada 30 s; los mandos de garaje modernos usan rolling codes.

```
Sin protección:   cliente ── "pagar 100 a Ana" + firma ──► servidor  acepta
                  atacante ── (mismo mensaje copiado) ──► servidor  acepta (paga otra vez)

Con nonce:        servidor ── nonce=8f21 ──► cliente
                  cliente ── "pagar 100 a Ana", nonce=8f21, firma ──► servidor  acepta (marca 8f21 como usado)
                  atacante ── copia con nonce=8f21 ──► servidor  rechaza (nonce repetido)
```

Analogía: es fotocopiar una entrada de concierto con código de barras. La primera persona que la escanee entra; si el escáner no marca el código como usado, la fotocopia también entra.

Ejemplo: una API de pagos acepta peticiones firmadas con HMAC pero no exige marca de tiempo. Un atacante que capture una petición de 100 € puede reenviarla 10 veces y generar 1000 € en pagos con firmas perfectamente válidas. Añadir `timestamp` con ventana de 5 min y un `request_id` único guardado en el servidor lo corta.

Mitigación: nonces, marcas de tiempo con ventana corta, números de secuencia, tokens de un solo uso, sesiones que expiran, TLS (que ya numera los registros), y en Windows preferir Kerberos o NTLMv2 frente a protocolos antiguos sin desafío.

```
Replay attack → reenviar un mensaje válido capturado; no necesita descifrarlo.
Nonce → número de un solo uso que hace única cada petición.
Timestamp + ventana → caduca el mensaje a los pocos minutos.
Pass the Hash → caso especial: reutilizar el hash como credencial (sección propia).
Session hijacking → robar una sesión activa y usarla; un replay de la cookie es una forma de hacerlo.
```

### Rogue Access Point

Un **Rogue Access Point** es un punto de acceso inalámbrico conectado a la red interna de una organización sin autorización del equipo de TI, que crea una entrada a la LAN que no pasa por los controles de seguridad.

Por qué aparece y por qué es grave: a veces lo instala un empleado con buena intención (un router doméstico bajo la mesa porque "el Wi-Fi de la oficina va lento"), a veces un atacante con acceso físico. En ambos casos el AP suele estar abierto o con una contraseña débil, y quien se conecte desde el aparcamiento queda dentro de la VLAN de ese puerto, detrás del firewall perimetral, sin pasar por 802.1X ni por el NAC.

```
Internet ── firewall ── switch corporativo ──┬── PCs
                                             └── puerto 14 ── Rogue AP "Linksys" (abierto)
                                                                   ▲
                                              coche en el aparcamiento ┘  → ya está en la LAN interna
```

Analogía: es un empleado que, para no ir a buscar la llave cada vez, deja abierta una ventana del sótano. El edificio sigue teniendo guardias en la puerta, pero ahora hay otra entrada que nadie vigila.

Ejemplo de detección: un barrido inalámbrico encuentra un BSSID que no está en el inventario, y la tabla MAC del switch muestra un fabricante de routers domésticos en un puerto de oficina.

```
SW1# show mac address-table interface Gi1/0/14
Vlan    Mac Address       Type      Ports
----    -----------       ----      -----
  10    c0:56:27:0a:0b:0c DYNAMIC   Gi1/0/14
```

Mitigación: 802.1X/NAC en todos los puertos cableados, para que un equipo no autenticado no obtenga acceso; port security que limite el número de MAC por puerto; WIPS que compare BSSID detectados con el inventario y localice los desconocidos; auditorías físicas y de radio periódicas; política que prohíba AP personales.

```
Rogue AP → AP no autorizado dentro de la red; abre la LAN a quien esté cerca.
Evil Twin → AP del atacante que imita un SSID; atrae clientes, no tiene por qué tocar la LAN.
Port security → limita qué MAC y cuántas pueden usar un puerto del switch.
802.1X / NAC → autentica el equipo antes de dar acceso al puerto.
```

### Evil Twin y Rogue AP atacan en direcciones opuestas

> [!IMPORTANT]
> Un Rogue AP es una puerta no autorizada hacia dentro de tu red; un Evil Twin es una trampa hacia fuera que engaña a tus usuarios. El primero se combate controlando los puertos de la LAN (802.1X), el segundo haciendo que los clientes validen la identidad de la red (WPA-Enterprise con certificado).

La diferencia que pregunta el examen es qué está conectado a qué y quién es la víctima.

```
                     Rogue AP                         Evil Twin
Quién lo instala     empleado o intruso               atacante
Conectado a          la LAN interna                   la red del atacante (o nada)
SSID                 cualquiera, suele ser propio     copia el de una red legítima
Víctima              la organización (red interna)    el usuario que se conecta
Dirección            de fuera hacia dentro            del usuario hacia el atacante
Defensa principal    802.1X/NAC + WIPS                WPA-Enterprise con validación de cert + WIPS
```

La fila "Dirección" es la que decide la respuesta.

Analogía: el Rogue AP es una ventana del sótano abierta en tu casa; el Evil Twin es una casa falsa con tu dirección pintada en la puerta para que tus invitados entren ahí.

Límite: un mismo aparato puede ser las dos cosas si está conectado a la LAN y además copia el SSID corporativo; en ese caso aplican ambas defensas.

```
Rogue AP → conectado a la LAN sin permiso; víctima la organización.
Evil Twin → imita un SSID; víctima el usuario.
```

### Buffer Overflow

Un **Buffer Overflow** (desbordamiento de búfer) es una falla de seguridad de memoria en la que un programa escribe en un búfer más datos de los que caben, y el exceso sobrescribe la memoria contigua: otras variables, punteros o la dirección de retorno de la función.

Por qué existe: lenguajes como C y C++ no comprueban los límites de los arrays; funciones antiguas como `strcpy`, `gets` o `sprintf` copian hasta encontrar un terminador sin saber cuánto espacio hay. Es CWE-120 (copia sin comprobar tamaño), con variantes en la pila (CWE-121) y en el heap (CWE-122). Lenguajes con memoria gestionada (Python, Java, Go, Rust seguro) comprueban los límites y lanzan un error en vez de desbordar.

Cómo funciona por dentro, en la pila: al llamar a una función, el procesador guarda en la pila la dirección a la que debe volver cuando termine, y justo debajo coloca las variables locales. Si una variable local es un búfer de 64 bytes y se copian 200, los bytes sobrantes suben por la pila y pisan la dirección de retorno. Al terminar la función, el procesador salta a la dirección que haya ahí. Con datos basura, el programa se cae (DoS); con datos elegidos, el atacante redirige la ejecución a código que él controla.

```
Pila de la función (crece hacia abajo; la copia escribe hacia arriba)

   direcciones altas
 ┌──────────────────────┐
 │ dirección de retorno │ ◄── el desbordamiento llega aquí
 ├──────────────────────┤
 │ canario (si existe)  │ ◄── se comprueba antes de volver; si cambió, el proceso aborta
 ├──────────────────────┤
 │ búfer[64]            │ ◄── la copia empieza aquí y sigue hacia arriba
 └──────────────────────┘
   direcciones bajas
```

Analogía: es llenar un vaso de 250 ml con una jarra de un litro sin mirar. El agua no desaparece: moja todo lo que hay alrededor, incluido el papel donde estaba apuntado a dónde tenías que ir después.

Ejemplo con código, vulnerable y corregido. En la versión vulnerable, cualquier nombre de más de 63 caracteres desborda `buf`.

```c
/* Vulnerable: no bounds check. */
void greet(const char *name) {
    char buf[64];
    strcpy(buf, name);
    printf("Hello %s\n", buf);
}
```

```c
/* Fixed: size-bounded copy, truncation detected and handled. */
#define NAME_MAX_LEN 64

int greet(const char *name) {
    char buf[NAME_MAX_LEN];
    int n = snprintf(buf, sizeof buf, "%s", name);
    if (n < 0 || (size_t)n >= sizeof buf) {
        return -1;              /* reject instead of silently truncating */
    }
    printf("Hello %s\n", buf);
    return 0;
}
```

Las protecciones del sistema y del compilador, que hay que saber nombrar y comprobar:

- Stack canary: valor aleatorio entre el búfer y la dirección de retorno; si cambió al volver, el proceso aborta (`-fstack-protector-strong` en GCC, `/GS` en MSVC).
- DEP / NX (No-eXecute): la pila y el heap se marcan como no ejecutables, así que inyectar código ahí no sirve.
- ASLR (Address Space Layout Randomization): las direcciones de pila, heap y librerías cambian en cada ejecución; el atacante no sabe a dónde saltar. En Linux, `kernel.randomize_va_space = 2`.
- PIE y RELRO: el propio ejecutable también se carga en dirección aleatoria y la tabla de saltos queda de solo lectura.
- `_FORTIFY_SOURCE=2`: el compilador sustituye funciones peligrosas por versiones que comprueban tamaños cuando lo conoce.

```
$ gcc -O2 -fstack-protector-strong -D_FORTIFY_SOURCE=2 -fPIE -pie -Wl,-z,relro,-z,now app.c -o app
$ cat /proc/sys/kernel/randomize_va_space
2
```

Detección en desarrollo: AddressSanitizer (`-fsanitize=address`) se detiene en la primera escritura fuera de límites y dice la línea exacta; el fuzzing alimenta el programa con millones de entradas aleatorias para encontrarlas.

> [!NOTE]
> Canarios, DEP y ASLR hacen la explotación mucho más difícil, pero no arreglan el bug: un desbordamiento sigue pudiendo tirar el proceso. La corrección de raíz es comprobar los límites o usar un lenguaje con memoria segura.

```
Buffer overflow → escribir más allá del búfer; pisa memoria contigua.
Stack canary → valor centinela que detecta la sobrescritura antes de volver.
DEP / NX → impide ejecutar código en pila y heap.
ASLR → aleatoriza direcciones para que el atacante no sepa a dónde saltar.
Memory leak → lo contrario en síntoma: memoria pedida que nunca se libera (siguiente sección).
```

### Memory Leak

Un **Memory Leak** (fuga de memoria) es un defecto de software en el que un programa reserva memoria y nunca la libera cuando deja de necesitarla, de modo que su consumo crece sin límite mientras está en ejecución.

Por qué es un problema de seguridad: no da acceso a nada, pero ataca la disponibilidad. Un servicio que pierde 2 KB por petición, con 500 peticiones por segundo, pierde 1 MB/s, unos 3,6 GB por hora; en un servidor de 4 GB el kernel acabará matando el proceso (OOM killer) o el sistema entero se volverá lento por el swap. Si un atacante sabe qué petición provoca la fuga, puede acelerarla a propósito: es un DoS de bajo volumen. Es CWE-401.

Cómo pasa: en C/C++, olvidar un `free`/`delete`, sobre todo en una ruta de error que sale antes. En lenguajes con recolector de basura (Java, Python, JavaScript), el recolector no puede liberar un objeto al que algo sigue apuntando: cachés que solo crecen, listas globales a las que se añade y nunca se quita, listeners que no se desregistran.

Analogía: es un restaurante donde los camareros ponen platos limpios en cada mesa pero nunca retiran los sucios. Al principio no se nota; a la hora no queda espacio en ninguna mesa y nadie puede comer.

Ejemplo con código, vulnerable y corregido. En la versión con fuga, cada petición con un tamaño inválido sale por el `return` sin liberar.

```c
/* Leaks on the error path. */
int handle(const struct request *req) {
    char *body = malloc(req->len);
    if (body == NULL) return -1;
    if (!is_valid_size(req->len)) return -1;   /* body never freed */
    process(body, req);
    free(body);
    return 0;
}
```

```c
/* Fixed: single exit path that always frees. */
int handle(const struct request *req) {
    int rc = -1;
    char *body = malloc(req->len);
    if (body == NULL) return -1;
    if (is_valid_size(req->len)) {
        process(body, req);
        rc = 0;
    }
    free(body);
    return rc;
}
```

Cómo se detecta: Valgrind o AddressSanitizer al terminar las pruebas, y en producción una gráfica de memoria que solo sube.

```
$ valgrind --leak-check=full ./server --self-test
==4312== LEAK SUMMARY:
==4312==    definitely lost: 204,800 bytes in 100 blocks
==4312==    indirectly lost: 0 bytes in 0 blocks
==4312== ...
==4312==    at 0x4846828: malloc (vg_replace_malloc.c:442)
==4312==    by 0x1091C2: handle (server.c:41)
```

Mitigación: un único punto de salida que libere (o RAII en C++, `defer` en Go, `with` en Python), cachés con tamaño máximo y expulsión, pruebas con Valgrind/ASan en CI, límites de memoria por servicio (cgroups, `MemoryMax=` en systemd) y reinicio automático para que una fuga no tumbe todo el servidor.

```
Memory leak → memoria reservada que nunca se libera; consumo crece hasta el OOM.
Buffer overflow → escritura fuera de límites; corrompe memoria.
Use-after-free → usar memoria ya liberada; el error contrario a la fuga.
OOM killer → mecanismo del kernel Linux que mata procesos cuando se acaba la memoria.
```

### Pass the Hash

El **Pass the Hash** (PtH) es una técnica de movimiento lateral en entornos Windows en la que el atacante usa directamente el hash NTLM de una contraseña, robado de un equipo comprometido, para autenticarse en otros equipos sin conocer ni descifrar la contraseña.

Por qué funciona: en la autenticación NTLM el servidor manda un desafío aleatorio y el cliente responde calculando una función del desafío con el hash NT de la contraseña. La contraseña en claro no interviene en ningún momento: el hash es, a efectos prácticos, la contraseña. Quien tenga el hash puede calcular la respuesta correcta. Los hashes están en la memoria del proceso LSASS de cualquier equipo donde ese usuario haya iniciado sesión, y en la base SAM local. Un atacante con privilegios de administrador local en un puesto puede leerlos. Está catalogado en MITRE ATT&CK como T1550.002.

```
1. Atacante compromete el PC de un usuario (phishing) y obtiene admin local.
2. Lee de la memoria de LSASS el hash NT de un administrador del helpdesk que inició sesión ahí.
3. Usa ese hash para responder desafíos NTLM en el servidor de archivos  ──► acceso como helpdesk
4. Repite en cada equipo donde esa cuenta sea admin  ──► movimiento lateral hasta un controlador de dominio
```

Analogía: es como si la cerradura de un hotel no leyera la tarjeta sino una fotocopia del código magnético; quien fotocopie la banda de cualquier tarjeta abre las mismas puertas que su dueño, sin saber nunca el número de habitación.

Ejemplo de detección en los registros de Windows: un inicio de sesión de red con NTLM desde un puesto de trabajo hacia muchos servidores en minutos, con una cuenta privilegiada.

```
Event ID 4624  An account was successfully logged on.
  Logon Type:                  3   (Network)
  Account Name:                helpdesk-admin
  Workstation Name:            PC-VENTAS-07
  Authentication Package:      NTLM
  Package Name (NTLM only):    NTLM V2
  Source Network Address:      10.0.20.57
```

El mismo patrón repetido contra 40 servidores desde `PC-VENTAS-07`, donde ese usuario nunca trabaja, es la señal. El tipo de logon 9 (`NewCredentials`) con proceso `seclogo` en el equipo origen es otra pista conocida.

Mitigación: Credential Guard (aísla los secretos de LSASS en un entorno virtualizado), cuentas privilegiadas en el grupo Protected Users (no usan NTLM ni guardan credenciales en caché), LAPS para que cada equipo tenga una contraseña de administrador local distinta (un hash robado no sirve en el resto), modelo de niveles (las cuentas de dominio admin nunca inician sesión en puestos de usuario), restringir o deshabilitar NTLM en favor de Kerberos, y MFA en accesos administrativos.

```
Pass the Hash → usar el hash NTLM robado como credencial; no hace falta la contraseña.
Pass the Ticket → lo mismo con un ticket Kerberos robado (ver 08-autenticacion).
Cracking de hashes → intentar recuperar la contraseña a partir del hash; PtH se lo salta.
LSASS → proceso de Windows que guarda en memoria las credenciales de sesión.
LAPS → contraseña única y rotada para el admin local de cada equipo.
```

### Directory Traversal

El **Directory Traversal** (path traversal) es una vulnerabilidad en la que una aplicación construye la ruta de un archivo con un dato del usuario sin restringirla, de modo que secuencias como `../` permiten salir de la carpeta prevista y leer (o escribir) archivos arbitrarios del servidor.

Por qué existe: es habitual servir archivos por nombre (`/download?file=informe.pdf`) y el código más simple pega el nombre a una carpeta base. El sistema de archivos interpreta `..` como "subir un nivel", así que el nombre puede escalar hasta la raíz. Es CWE-22. Objetivos típicos: archivos de configuración con contraseñas, `/etc/passwd` en Linux, claves privadas, el código fuente de la propia aplicación. Si la aplicación además incluye el archivo como código (Local File Inclusion, frecuente en PHP), el impacto sube a ejecución de código.

Analogía: un archivista recibe la petición "tráeme la carpeta Clientes/../../Dirección/Nóminas" y, como solo sigue el camino escrito, sale del archivo de clientes y entra en el despacho de dirección.

Ejemplo con código, vulnerable y corregido (Python). En la versión vulnerable, un nombre de archivo que empiece con varios `../` seguidos de una ruta del sistema devuelve ese archivo del sistema, porque `os.path.join` no impide subir niveles.

```python
# Vulnerable: user input joined to the base directory and opened as-is.
BASE_DIR = "/srv/app/reports"

def read_report(filename):
    with open(os.path.join(BASE_DIR, filename), "rb") as f:
        return f.read()
```

```python
# Fixed: resolve the real path and require it to stay inside the base.
BASE_DIR = Path("/srv/app/reports").resolve()

def read_report(filename):
    target = (BASE_DIR / filename).resolve()
    if not target.is_relative_to(BASE_DIR):
        raise PermissionError(f"path escapes base dir: {filename!r}")
    return target.read_bytes()
```

La corrección no busca `../` en el texto (hay codificaciones como `%2e%2e%2f` o `..\` que esquivan esos filtros): resuelve la ruta final y comprueba que sigue dentro de la carpeta. Mejor todavía, no aceptar nombres de archivo: aceptar un ID y buscarlo en una tabla (`report_id=42` → `informe-q3.pdf`).

Señal en el log del servidor web:

```
203.0.113.7 - - [02/Oct/2026:10:31:02] "GET /download?file=..%2f..%2f..%2fetc%2fpasswd HTTP/1.1" 403 0
203.0.113.7 - - [02/Oct/2026:10:31:03] "GET /download?file=....//....//etc/passwd HTTP/1.1" 403 0
```

Mitigación: mapear IDs a archivos en lugar de aceptar rutas, resolver con `realpath`/`resolve()` y comparar con la base, lista blanca de nombres y extensiones, ejecutar el servicio con un usuario sin permisos fuera de su carpeta (o en un contenedor o chroot), y WAF como capa extra.

```
Directory traversal → salir de la carpeta prevista con ../ en un nombre de archivo; lee archivos.
LFI (Local File Inclusion) → además incluye/ejecuta el archivo local como código.
RFI (Remote File Inclusion) → incluye un archivo desde una URL externa.
Canonicalización → resolver la ruta final real antes de decidir si se permite.
```

## Recursos para aprender y practicar

### Videos

- [Application Attacks - CompTIA Security+ SY0-701 - 2.4](https://www.youtube.com/watch?v=yRSqIGjeb7s) — Professor Messer; escalada de privilegios, directory traversal y otros ataques a aplicaciones (nodos Web Based Attacks and OWASP10 y Directory Traversal).
- [SQL Injection - CompTIA Security+ SY0-701 - 2.3](https://www.youtube.com/watch?v=qFUOLkEk8AQ) — Professor Messer; mecanismo y mitigación de SQLi (nodo SQL Injection).
- [Cross-site Scripting - CompTIA Security+ SY0-701 - 2.3](https://www.youtube.com/watch?v=PKgw0CLZIhE) — Professor Messer; XSS reflejado y almacenado (nodo XSS).
- [Hacking Websites with SQL Injection - Computerphile](https://www.youtube.com/watch?v=_jKylhJtPmI) — Computerphile; por qué concatenar rompe la consulta (nodo SQL Injection).
- [Cracking Websites with Cross Site Scripting - Computerphile](https://www.youtube.com/watch?v=L5l9lSnNMxg) — Computerphile; XSS explicado desde el navegador (nodo XSS).
- [Denial of Service - CompTIA Security+ SY0-701 - 2.4](https://www.youtube.com/watch?v=Z7OntvK--PQ) — Professor Messer; DoS, DDoS y amplificación (nodo DoS vs DDoS).
- [On-path Attacks - CompTIA Security+ SY0-701 - 2.4](https://www.youtube.com/watch?v=M_Af6_8JTuo) — Professor Messer; MITM y ARP poisoning (nodos MITM y Spoofing).
- [DNS Attacks - CompTIA Security+ SY0-701 - 2.4](https://www.youtube.com/watch?v=BoxeL5ybOXI) — Professor Messer; envenenamiento de caché y secuestro de dominios (nodo DNS Poisoning).
- [Wireless Attacks - CompTIA Security+ SY0-701 - 2.4](https://www.youtube.com/watch?v=tSLqrKhUvts) — Professor Messer; desautenticación, RF jamming y 802.11w (nodo Deauth Attack).
- [Replay Attacks - CompTIA Security+ SY0-701- 2.4](https://www.youtube.com/watch?v=ai6qS13gKRo) — Professor Messer; replay, pass the hash y session hijacking (nodos Replay Attack y Pass the Hash).
- [Buffer Overflows - CompTIA Security+ SY0-701 - 2.3](https://www.youtube.com/watch?v=0-qeeI5jTqU) — Professor Messer; desbordamiento y sus protecciones (nodo Buffer Overflow).
- [Running a Buffer Overflow Attack - Computerphile](https://www.youtube.com/watch?v=1S0aBV-Waeo) — Computerphile; qué pasa en la pila durante un desbordamiento (nodo Buffer Overflow).

### Lectura y documentación

- [OWASP Top 10:2021](https://owasp.org/Top10/2021/) y [OWASP Top 10:2025](https://top10.owasp.org/2025/) — las dos ediciones, con CWE por categoría; la [introducción de 2025](https://top10.owasp.org/2025/0x00_2025-Introduction/) explica qué cambió.
- [OWASP Top 10:2025 A10 Mishandling of Exceptional Conditions](https://top10.owasp.org/2025/A10_2025-Mishandling_of_Exceptional_Conditions/) — la categoría nueva con ejemplos de fail open.
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) y [Query Parameterization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html) — consultas parametrizadas en cada lenguaje.
- [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html), [DOM based XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html) y [Content Security Policy Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html) — escape por contexto, sinks peligrosos y CSP.
- [OWASP Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) — tokens, SameSite y comprobación de origen.
- [OWASP Server Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) — SSRF (A10:2021, dentro de A01:2025).
- [OWASP Denial of Service Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html) y [CISA: Understanding Denial-of-Service Attacks](https://www.cisa.gov/news-events/news/understanding-denial-service-attacks) — tipos de DoS y defensas.
- [OWASP: Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal), [Manipulator-in-the-middle attack](https://owasp.org/www-community/attacks/Manipulator-in-the-middle_attack) y [Buffer Overflow Attack](https://owasp.org/www-community/attacks/Buffer_overflow_attack) — fichas de ataque de la comunidad OWASP.
- CWE: [CWE-89](https://cwe.mitre.org/data/definitions/89.html), [CWE-79](https://cwe.mitre.org/data/definitions/79.html), [CWE-352](https://cwe.mitre.org/data/definitions/352.html), [CWE-22](https://cwe.mitre.org/data/definitions/22.html), [CWE-120](https://cwe.mitre.org/data/definitions/120.html), [CWE-401](https://cwe.mitre.org/data/definitions/401.html) — definición, consecuencias y mitigaciones de cada falla.
- MITRE ATT&CK: [T1557 Adversary-in-the-Middle](https://attack.mitre.org/techniques/T1557/) (con [T1557.002 ARP Cache Poisoning](https://attack.mitre.org/techniques/T1557/002/)), [T1498 Network Denial of Service](https://attack.mitre.org/techniques/T1498/), [T1499 Endpoint Denial of Service](https://attack.mitre.org/techniques/T1499/), [T1550.002 Pass the Hash](https://attack.mitre.org/techniques/T1550/002/) — detección y mitigación con casos reales.
- [RFC 4033: DNS Security Introduction and Requirements](https://www.rfc-editor.org/rfc/rfc4033) y [RFC 5452: Measures for Making DNS More Resilient against Forged Answers](https://www.rfc-editor.org/rfc/rfc5452) — DNSSEC y aleatorización de puerto y TXID (nodo DNS Poisoning).
- [NIST SP 800-153: Guidelines for Securing WLANs](https://csrc.nist.gov/pubs/sp/800/153/final) y [NIST SP 800-97: Establishing Wireless Robust Security Networks (802.11i)](https://csrc.nist.gov/pubs/sp/800/97/final) — rogue AP, WIDS y seguridad 802.11 (nodos Evil Twin, Rogue Access Point y Deauth Attack).
- [Microsoft: Credential Guard](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/) y [Protected Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/how-to-configure-protected-accounts) — defensas contra Pass the Hash.
- [AddressSanitizer (Clang)](https://clang.llvm.org/docs/AddressSanitizer.html), [Valgrind Memcheck](https://valgrind.org/docs/manual/mc-manual.html) y [MSVC /GS](https://learn.microsoft.com/en-us/cpp/build/reference/gs-buffer-security-check) — detección de desbordamientos y fugas, y canarios en Windows.

### Práctica

- [PortSwigger Web Security Academy: todos los labs](https://portswigger.net/web-security/all-labs) — gratis y legal; en especial [SQL injection](https://portswigger.net/web-security/sql-injection), [Cross-site scripting](https://portswigger.net/web-security/cross-site-scripting), [DOM-based](https://portswigger.net/web-security/dom-based), [CSRF](https://portswigger.net/web-security/csrf), [Path traversal](https://portswigger.net/web-security/file-path-traversal), [Access control](https://portswigger.net/web-security/access-control) y [SSRF](https://portswigger.net/web-security/ssrf) (nodos SQL Injection, XSS, CSRF, Directory Traversal y OWASP10).
- [TryHackMe: OWASP Top 10 2025: IAAA Failures](https://tryhackme.com/room/owasptopten2025one), [Application Design Flaws](https://tryhackme.com/room/owasptopten2025two) e [Insecure Data Handling](https://tryhackme.com/room/owasptopten2025three) — gratis; la edición 2025 por bloques (nodo Web Based Attacks and OWASP10).
- [TryHackMe: SQL Injection](https://tryhackme.com/room/sqlinjectionlm) — tipos de SQLi y su remediación (nodo SQL Injection).
- [TryHackMe: Intro to Cross-site Scripting](https://tryhackme.com/room/axss) — reflected, stored y DOM (nodo XSS).
- [TryHackMe: CSRF](https://tryhackme.com/room/csrfV2) — cómo se forja la petición y cómo se defiende (nodo CSRF).
- [PortSwigger: Path traversal](https://portswigger.net/web-security/file-path-traversal) — gratis; teoría y labs de directory traversal, de lo básico a filtros evadidos (nodo Directory Traversal).
- [TryHackMe: DVWA](https://tryhackme.com/room/dvwa) y [DVWA en GitHub](https://github.com/digininja/DVWA) — aplicación deliberadamente vulnerable con niveles low/medium/high/impossible; compara el código de cada nivel para ver la corrección (nodos SQL Injection, XSS, CSRF, Directory Traversal).
- [TryHackMe: OWASP Juice Shop](https://tryhackme.com/room/owaspjuiceshop) y [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) — tienda vulnerable moderna que cubre todo el Top 10 (nodo OWASP10).
- [TryHackMe: L2 MAC Flooding & ARP Spoofing](https://tryhackme.com/room/layer2) — ataques de capa 2 en un entorno aislado (nodos Spoofing y MITM).
- [TryHackMe: Wifi Hacking 101](https://tryhackme.com/room/wifihacking101) — WPA, handshake y deauth (nodos Deauth Attack y Evil Twin).
- [TryHackMe: PWN101](https://tryhackme.com/room/pwn101) — gratis; desbordamientos de pila en binarios de práctica (nodo Buffer Overflow). Más práctica de memoria: [pwn.college](https://pwn.college/) (gratis, por módulos) y [Exploit Education Phoenix](https://exploit.education/phoenix/).
- Ejercicio en casa (Memory Leak y Buffer Overflow): compila el ejemplo vulnerable de `handle` y de `greet` con `-fsanitize=address -g`, ejecútalos con una entrada larga e inválida, lee el informe de ASan y comprueba que la versión corregida sale limpia.
- Ejercicio en casa (VLAN Hopping): en Packet Tracer, monta dos switches con un trunk y dos VLAN; ejecuta `show interfaces trunk`, aplica la configuración endurecida de la sección y verifica que ningún puerto de acceso negocia trunk.
- Ejercicio en casa (DNS Poisoning): ejecuta `dig +dnssec` contra un dominio firmado y contra `dnssec-failed.org`, y compara la bandera `ad` y el `SERVFAIL`.

## Cuadro resumen

Todo lo visto, en una línea por término.

Web Based Attacks and OWASP10

```
Ataque web → abuso de cómo una aplicación HTTP procesa la entrada, controla el acceso o está configurada.
OWASP Top 10 → ranking de categorías de riesgo; sirve para priorizar y concienciar.
OWASP ASVS → lista de requisitos verificables; sirve para auditar o exigir en un contrato.
CWE → catálogo de tipos de falla individuales (CWE-89, CWE-79...).
CVE → una falla concreta en un producto concreto (CVE-2021-44228, Log4Shell).
Método del Top 10 → 8 categorías por datos de pruebas + 2 por encuesta a la comunidad.
A01:2021 Broken Access Control → el servidor no verifica permisos por objeto o acción (IDOR); 94 % de apps con alguna forma.
A02:2021 Cryptographic Failures → datos sensibles sin cifrar o con criptografía débil.
A03:2021 Injection → datos del usuario ejecutados como código; incluye XSS.
A04:2021 Insecure Design → falla de diseño; ningún parche la arregla.
A05:2021 Security Misconfiguration → software correcto mal configurado; incluye XXE.
A06:2021 Vulnerable and Outdated Components → dependencias con CVE conocidos.
A07:2021 Identification and Authentication Failures → login, contraseñas y sesiones débiles.
A08:2021 Software and Data Integrity Failures → código o datos no verificados; deserialización insegura.
A09:2021 Security Logging and Monitoring Failures → sin registros o sin nadie que los mire.
A10:2021 SSRF → el servidor hace peticiones a destinos que elige el atacante (p. ej. 169.254.169.254).
A01:2025 Broken Access Control → sigue primero y absorbe SSRF.
A02:2025 Security Misconfiguration → sube del 5 al 2.
A03:2025 Software Supply Chain Failures → amplía A06:2021 a dependencias, build y distribución.
A04:2025 Cryptographic Failures → baja del 2 al 4.
A05:2025 Injection → baja del 3 al 5; la categoría con más CVE.
A06:2025 Insecure Design → baja del 4 al 6.
A07:2025 Authentication Failures → mismo puesto, nombre más corto.
A08:2025 Software or Data Integrity Failures → mismo puesto; integridad de artefactos y datos concretos.
A09:2025 Security Logging and Alerting Failures → mismo puesto; log sin alerta casi no sirve.
A10:2025 Mishandling of Exceptional Conditions → nueva, 24 CWE; errores mal manejados y fail open.
Fail closed → ante un error de verificación, denegar.
```

Common Attacks

```
DoS → una fuente; se filtra por IP.
DDoS → miles de fuentes (botnet); requiere absorber el volumen aguas arriba.
Volumétrico → satura el enlace; se mide en Gbps.
De protocolo → satura tablas de estado (SYN flood); se mide en pps; SYN cookies.
De aplicación → peticiones caras en capa 7; se mide en rps.
Amplificación → respuesta mucho mayor que la consulta (DNS 28-54x, NTP 556x), dirigida a la víctima con IP falsificada.
MITM / on-path → el atacante queda entre dos partes y lee o modifica el tráfico.
SSL stripping → baja la conexión de HTTPS a HTTP para leerla.
HSTS → cabecera que obliga al navegador a usar solo HTTPS con ese dominio.
Eavesdropping (sniffing) → solo escucha; no necesariamente está en el camino ni modifica.
CSRF → el navegador de la víctima hace una petición que ella no quiso; el atacante no lee la respuesta.
Token anti-CSRF → valor secreto en el formulario que otro origen no puede conocer.
SameSite → atributo de cookie que impide enviarla en peticiones iniciadas desde otro sitio.
IP spoofing → IP de origen falsa; útil sin respuesta (DDoS, amplificación); BCP 38 lo filtra.
MAC spoofing → MAC falsa; salta filtros por MAC.
ARP spoofing → asocia la IP de otro con tu MAC; base del MITM en LAN; Dynamic ARP Inspection.
Email spoofing → remitente falso; se combate con SPF, DKIM y DMARC.
Dirección (IP, MAC, SSID, From) → la escribe el emisor; no autentica.
Autenticación criptográfica → prueba la identidad con una clave que el atacante no tiene.
SQL injection → la entrada del usuario cambia la estructura de una consulta SQL (CWE-89).
In-band → el resultado sale en la propia respuesta.
Blind → no sale nada; se deduce por cambios o retrasos en la respuesta.
Consulta parametrizada → el texto SQL es fijo y los valores viajan aparte.
Reflected XSS → el dato va en la petición y vuelve en la respuesta; requiere un enlace.
Stored XSS → el dato se guarda en el servidor y afecta a todos los visitantes.
DOM-based XSS → el JS del cliente escribe el dato como HTML; el servidor no lo ve.
Output encoding → convierte caracteres especiales en entidades para que se muestren, no se ejecuten.
CSP → cabecera que limita de dónde puede cargar y ejecutar scripts el navegador.
HttpOnly → la cookie no es accesible desde JavaScript.
Inyección → dato y código comparten canal; el intérprete ejecuta el dato.
Separación de código y datos → la defensa de raíz para todas las variantes.
Validación de entrada → defensa complementaria; no reemplaza a la separación.
Evil Twin → AP del atacante que imita el SSID de una red legítima; objetivo: MITM o robo de credenciales.
WIDS/WIPS → sistema que detecta (y bloquea) AP y comportamientos Wi-Fi anómalos.
Switch spoofing → el atacante negocia un trunk con DTP y accede a todas las VLAN.
Double tagging → dos etiquetas 802.1Q; abusa de la native VLAN; solo ida.
Native VLAN → la VLAN que viaja sin etiqueta en un trunk; por defecto la 1; usar una dedicada (999).
DTP → protocolo Cisco que negocia trunks automáticamente; se desactiva con nonegotiate.
DNS cache poisoning → respuesta falsa guardada en la caché del resolver; afecta a todos sus clientes durante el TTL.
TXID + puerto aleatorio → 65 536 × ~64 000 combinaciones (RFC 5452).
DNSSEC → firma los registros; garantiza autenticidad e integridad, no confidencialidad.
DoH / DoT → cifran el canal cliente-resolver; no firman los registros.
Deauth attack → tramas de gestión falsas que desconectan clientes; no requiere la clave.
802.11w / PMF → protege las tramas de gestión; obligatorio en WPA3.
Jamming → interferencia de radio pura; capa física, sin tramas.
Replay attack → reenviar un mensaje válido capturado; no necesita descifrarlo.
Nonce → número de un solo uso que hace única cada petición.
Timestamp + ventana → caduca el mensaje a los pocos minutos (Kerberos: 5 min).
Rogue AP → AP no autorizado conectado a la LAN; víctima la organización; 802.1X/NAC + WIPS.
Evil Twin vs Rogue AP → el gemelo engaña al usuario hacia fuera; el rogue abre la red hacia dentro.
Port security → limita qué MAC y cuántas pueden usar un puerto del switch.
Buffer overflow → escribir más allá del búfer; pisa memoria contigua y la dirección de retorno.
Stack canary → valor centinela que detecta la sobrescritura antes de volver.
DEP / NX → impide ejecutar código en pila y heap.
ASLR → aleatoriza direcciones para que el atacante no sepa a dónde saltar (randomize_va_space = 2).
Memory leak → memoria reservada que nunca se libera; consumo crece hasta el OOM (CWE-401).
Use-after-free → usar memoria ya liberada; el error contrario a la fuga.
OOM killer → mecanismo del kernel Linux que mata procesos cuando se acaba la memoria.
Pass the Hash → usar el hash NTLM robado como credencial; no hace falta la contraseña (T1550.002).
Pass the Ticket → lo mismo con un ticket Kerberos robado.
LSASS → proceso de Windows que guarda en memoria las credenciales de sesión.
LAPS → contraseña única y rotada para el admin local de cada equipo.
Directory traversal → salir de la carpeta prevista con ../ en un nombre de archivo; lee archivos (CWE-22).
LFI / RFI → incluye y ejecuta un archivo local / remoto como código.
Canonicalización → resolver la ruta final real antes de decidir si se permite.
```
