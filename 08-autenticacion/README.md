# Autenticación

## Conceptos previos

- Identidad: el "quién" que un sistema reconoce (un usuario, un servicio, un equipo), normalmente con un identificador único como un nombre de usuario o un correo.
- Credencial: la prueba que se presenta para demostrar una identidad (contraseña, código, llave física, certificado).
- Sujeto / principal: la entidad que pide acceso; en Kerberos se llama principal (`ana@EMPRESA.LOCAL`).
- Recurso: lo que se quiere usar (un archivo, una base de datos, una API, una impresora).
- Hash: huella de longitud fija que se calcula de un dato y no se puede revertir; se guarda en lugar de la contraseña (detalle en [`10-criptografia`](../10-criptografia/)).
- Cifrado simétrico / asimétrico: con una sola llave compartida, o con un par llave pública + llave privada (detalle en [`10-criptografia`](../10-criptografia/)).
- Token: un dato firmado o aleatorio que el servidor entrega tras autenticar y que se presenta después en vez de repetir la contraseña.
- Dominio / directorio: una base central de usuarios y equipos de una organización (Active Directory en Windows, OpenLDAP en Linux).
- Phishing: engañar a alguien para que entregue sus credenciales en una página o mensaje falso.
- Puerto: número que identifica un servicio en un equipo (repaso en [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/)).

## Authentication vs Authorization

### Authentication vs Authorization

La **autenticación** (authentication, AuthN) es el proceso de seguridad que comprueba que alguien es quien dice ser; la autorización (authorization, AuthZ) es el proceso que decide qué puede hacer esa identidad ya comprobada.

Existen por separado porque responden preguntas distintas. Saber que quien entra es Ana no dice nada sobre si Ana puede borrar la base de datos de nómina. Si un sistema mezcla las dos cosas aparecen los fallos clásicos: aplicaciones que verifican el login pero no comprueban si el usuario 105 tiene permiso para ver la factura del usuario 106 (eso es un IDOR, una referencia directa insegura, que se ve en [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/)).

Analogía: en un aeropuerto, el agente que mira tu pasaporte y tu cara te autentica; la tarjeta de embarque que dice "puerta 12, asiento 14C, solo clase turista" te autoriza. Pasar el control de pasaportes no te deja sentarte en primera clase.

El orden siempre es el mismo:

```
Identificación   ->  "soy ana"                 (digo quién soy)
Autenticación    ->  contraseña + código       (lo demuestro)
Autorización     ->  rol "contabilidad": leer  (el sistema decide qué puedo hacer)
Accounting       ->  "ana leyó nomina.xlsx 09:14" (queda registrado)
```

La autorización se implementa con modelos de control de acceso. Los tres que más aparecen:

- DAC (Discretionary Access Control): el dueño del recurso decide quién entra. Los permisos `rwx` de Linux son DAC.
- MAC (Mandatory Access Control): una política central impone etiquetas (secreto, confidencial) que ni el dueño puede cambiar. SELinux es MAC.
- RBAC (Role-Based Access Control): los permisos se asignan a roles ("contabilidad", "soporte") y los usuarios reciben roles. Es lo normal en empresas.

Ejemplo: en una API, el login devuelve un token que dice `sub=ana`. Cada petición pasa dos filtros: el middleware de autenticación valida que el token sea auténtico y no haya vencido (si falla, HTTP 401 Unauthorized, que pese al nombre significa "no autenticado"); luego la lógica de autorización comprueba si `ana` tiene permiso sobre `/facturas/106` (si no, HTTP 403 Forbidden).

```
$ curl -i https://api.ejemplo.test/facturas/106
HTTP/1.1 401 Unauthorized          <- no mandé token: no sé quién eres

$ curl -i -H "Authorization: Bearer eyJhbGciOi..." https://api.ejemplo.test/facturas/106
HTTP/1.1 403 Forbidden             <- sé que eres ana, pero esa factura no es tuya
```

### AAA

AAA (Authentication, Authorization, Accounting) es un marco de control de acceso que agrupa en un solo servicio las tres funciones: comprobar identidad, decidir permisos y registrar lo que se hizo.

Accounting (contabilidad o registro) es la tercera A y la que se olvida: guardar quién entró, cuándo, desde dónde, cuánto tiempo y qué consumió. Sirve para facturar (un ISP cobra por tiempo conectado), para auditoría y para investigar un incidente. Sin accounting, un atacante con credenciales robadas no deja rastro útil. Los protocolos que implementan AAA en redes son RADIUS y TACACS+ (ver RADIUS más abajo). Algunos textos añaden una cuarta pieza, la identificación, y hablan de IAAA.

Analogía: un hotel. La recepción revisa tu cédula (autenticación), la tarjeta-llave abre solo tu habitación y el gimnasio (autorización), y el sistema anota cada vez que la tarjeta abrió una puerta (accounting).

> [!WARNING]
> HTTP 401 se llama "Unauthorized" pero significa "no autenticado"; el que significa "no autorizado" es 403 Forbidden. Es una trampa de examen y una fuente real de bugs.

```
Identificación → afirmar una identidad ("soy ana").
Authentication (AuthN) → demostrar esa identidad; falla con HTTP 401.
Authorization (AuthZ) → decidir qué puede hacer; falla con HTTP 403.
Accounting → registrar qué hizo, cuándo y desde dónde.
AAA → marco que junta las tres; RADIUS y TACACS+ lo implementan.
DAC → el dueño decide permisos (rwx de Linux).
MAC → una política central impone etiquetas (SELinux).
RBAC → permisos por rol, usuarios reciben roles.
```

## MFA & 2FA

### MFA & 2FA

La autenticación multifactor (MFA, Multi-Factor Authentication) es un método de autenticación que exige pruebas de al menos dos categorías distintas de factor; 2FA (Two-Factor Authentication) es el caso particular de exactamente dos.

Existe porque la contraseña sola se roba con demasiada facilidad: phishing, filtraciones de bases de datos, reutilización entre sitios, keyloggers. Con MFA, una contraseña robada ya no basta; el atacante necesita además algo que no está en la base de datos filtrada.

Analogía: el cajero automático. La tarjeta es algo que tienes y el PIN es algo que sabes. Quien te roba la billetera tiene la tarjeta pero no el PIN; quien te mira teclear tiene el PIN pero no la tarjeta.

### Los factores

Un factor de autenticación es una categoría de prueba de identidad que se distingue por el tipo de cosa que el atacante tendría que robar para suplantarte. Las categorías, que hay que saber recitar:

- Algo que sabes (knowledge): contraseña, PIN, respuesta a una pregunta secreta.
- Algo que tienes (possession): teléfono con app autenticadora, llave de seguridad USB/NFC, tarjeta inteligente.
- Algo que eres (inherence): huella, rostro, iris, voz.
- Algunos marcos añaden factores contextuales: algún lugar donde estás (geolocalización, red de la oficina) y algo que haces (patrón de tecleo). Se usan más como señales de riesgo que como factores por sí solos.

La regla de oro es que cuentan las **categorías**, no la cantidad. Contraseña + pregunta secreta son dos cosas que sabes: sigue siendo un solo factor, porque el mismo phishing roba las dos.

### TOTP

TOTP (Time-based One-Time Password, RFC 6238) es un algoritmo de contraseña de un solo uso que genera un código de 6 dígitos a partir de un secreto compartido y la hora actual.

Al activar 2FA, el servidor genera un secreto aleatorio (típicamente 160 bits) y te lo muestra como código QR; la app (Google Authenticator, Aegis, el gestor de contraseñas) lo guarda. Desde ese momento, ambos lados calculan lo mismo sin hablarse:

```
T      = floor(segundos_desde_1970 / 30)        # cambia cada 30 s
HMAC   = HMAC-SHA1(secreto, T)                   # 20 bytes
código = truncar(HMAC) mod 10^6                  # 6 dígitos
```

- `T` es el contador de ventanas de 30 segundos; por eso el código cambia cada medio minuto.
- HMAC es una función que mezcla una llave con un mensaje y produce una huella; sin el secreto no se puede predecir.
- El servidor suele aceptar también la ventana anterior y la siguiente (±30 s) por si el reloj del teléfono está desfasado.

Funciona sin internet en el teléfono, porque solo necesita el secreto y el reloj. HOTP (RFC 4226) es su antecesor: usa un contador que avanza con cada uso en vez de la hora.

Ejemplo con el vector de prueba del propio RFC 6238 (secreto ASCII `12345678901234567890`, segundo 59, código de 8 dígitos):

```
$ oathtool --totp -d 8 --now "1970-01-01 00:00:59 UTC" \
    $(printf 12345678901234567890 | xxd -p)
94287082
```

El punto débil de TOTP es que el código se puede teclear en una página falsa: un proxy de phishing (Evilginx, por ejemplo) lo reenvía al sitio real en menos de 30 segundos.

### FIDO2 / passkeys

FIDO2 es un estándar de autenticación con criptografía de llave pública, formado por WebAuthn (la API del navegador, W3C) y CTAP2 (el protocolo entre el navegador y el autenticador), que reemplaza el secreto compartido por un par de llaves por sitio.

Así funciona:

```
REGISTRO
  autenticador (llave USB / teléfono) genera par de llaves SOLO para "banco.com"
  llave privada -> se queda en el chip, nunca sale
  llave pública -> se envía al servidor de banco.com

LOGIN
  servidor   ---- reto aleatorio (challenge) ---->  navegador
  navegador  añade el origen real: "https://banco.com"
  autenticador: verifica tu huella/PIN (verificación local)
                firma (challenge + origen) con la llave privada
  navegador  ---- firma ---->  servidor
  servidor verifica la firma con la llave pública guardada
```

La pieza clave es que el navegador mete el origen (el dominio real) en lo firmado. Si estás en `banco-com.login.xyz`, el autenticador no tiene llave para ese dominio o la firma no valida en `banco.com`. Por eso FIDO2 es resistente al phishing: no hay nada que el usuario pueda teclear y entregar.

Una passkey es una credencial FIDO2 "descubrible" que además se puede sincronizar entre dispositivos (llavero de iCloud, gestor de Google, Bitwarden). Juntan dos factores en un gesto: la posesión del dispositivo y la huella o PIN local que lo desbloquea. El servidor solo guarda una llave pública, así que una filtración de su base de datos no expone nada reutilizable.

### Ataques de fatiga MFA

La fatiga MFA (MFA fatigue, MFA bombing o push spamming) es un ataque de ingeniería social que consiste en disparar notificaciones push de aprobación una y otra vez hasta que la víctima acepta una por cansancio o confusión.

Requisito: el atacante ya tiene la contraseña (de una filtración o phishing). Entonces inicia sesión 20, 50, 100 veces seguidas; cada intento manda un "¿Eres tú? Aprobar / Rechazar" al teléfono. A las 2 de la madrugada, o con un mensaje de WhatsApp que dice "soy de TI, acepta para que pare", la víctima toca Aprobar. Es la técnica T1621 (Multi-Factor Authentication Request Generation) de MITRE ATT&CK, y fue la puerta de entrada en el ataque de Lapsus$ a Uber en 2022.

Defensas, en orden de eficacia:

- Number matching: la pantalla de login muestra un número (p. ej. 47) que hay que teclear en la app; aprobar a ciegas deja de funcionar.
- Límite de pushes (p. ej. bloquear tras 3 rechazos en 10 minutos) y alerta al SOC.
- Mostrar contexto en el push: ciudad, IP, aplicación.
- Migrar a FIDO2, que no tiene botón de "aprobar" que se pueda abusar.

Otros ataques a MFA que hay que conocer: SIM swapping (convencer a la operadora de pasar tu número a otra SIM para recibir los SMS), phishing en tiempo real con proxy (AiTM, adversary-in-the-middle) que roba el código o la cookie de sesión ya autenticada, y robo de cookies de sesión con malware, que salta MFA por completo porque la sesión ya existe.

Ejemplo de decisión: una empresa de 500 empleados. Con SMS, un SIM swap compromete una cuenta. Con push simple, un bombardeo nocturno compromete otra. Con number matching, el bombardeo falla, pero un proxy AiTM aún roba la sesión. Con passkeys, el proxy recibe una firma que no vale para su dominio y el ataque se cae.

> [!TIP]
> Ranking práctico de factores, de más débil a más fuerte: SMS < llamada de voz < push simple < TOTP < push con number matching < FIDO2/passkey. NIST SP 800-63B ya marca el SMS como autenticador "restringido".

```
Algo que sabes → contraseña, PIN; se roba con phishing o filtraciones.
Algo que tienes → teléfono, llave FIDO2, tarjeta inteligente.
Algo que eres → biometría; no se puede cambiar si se filtra.
2FA → exactamente dos factores de categorías distintas.
MFA → dos o más categorías distintas; contraseña + pregunta secreta NO es MFA.
TOTP → HMAC(secreto, hora/30 s) → 6 dígitos; phisheable.
HOTP → igual que TOTP pero con contador de usos.
FIDO2 → WebAuthn + CTAP2; firma ligada al dominio; resistente al phishing.
Passkey → credencial FIDO2 descubrible y sincronizable.
MFA fatigue → bombardeo de pushes hasta que aceptas (ATT&CK T1621).
Number matching → teclear el número de la pantalla; frena la fatiga MFA.
SIM swapping → robar tu número para recibir tus SMS.
AiTM → proxy de phishing que reenvía código y roba la cookie de sesión.
```

### MFA resistente al phishing es lo único que corta el phishing

> [!IMPORTANT]
> Todo factor que el usuario pueda leer y teclear (SMS, TOTP, push) se puede retransmitir en tiempo real a un atacante; solo los factores criptográficamente atados al dominio (FIDO2, certificados con mTLS) cierran ese hueco.

La resistencia al phishing es una propiedad de un autenticador que impide que la prueba de identidad sirva en un sitio distinto del legítimo. La diferencia no es "más o menos seguro", es dónde vive la verificación del dominio: en la cabeza del usuario o en el protocolo.

```
                      SMS    TOTP   Push   Push+número   FIDO2
Secreto en servidor   sí     sí     no     no            no (solo pública)
Usuario puede         sí     sí     sí     sí            no
  entregarlo a otro
Verifica el dominio   no     no     no     no            sí  <- la fila que importa
Resiste proxy AiTM    no     no     no     no            sí
```

La penúltima fila es la idea entera: mientras el dominio lo compruebe una persona mirando la barra de direcciones, el phishing gana.

Límite: FIDO2 no protege contra un equipo ya infectado que roba la cookie de sesión después del login, ni contra el proceso de recuperación de cuenta ("perdí mi llave, mándame un SMS"), que se vuelve el eslabón débil si no se protege igual.

```
Phisheable → SMS, TOTP, push: el usuario puede entregar el código.
Resistente al phishing → FIDO2/passkey, mTLS: la prueba solo vale en el dominio real.
```

## Authentication Methodologies

### Kerberos

Kerberos es un protocolo de autenticación en red basado en tickets que permite a un usuario demostrar su identidad una vez ante un servidor central de confianza y luego acceder a varios servicios sin volver a enviar su contraseña por la red.

Nació en el MIT (proyecto Athena, versión 5 en RFC 4120) para redes donde no se puede confiar en el cable: la contraseña nunca viaja, ni cifrada. Es el protocolo de autenticación por defecto de Active Directory desde Windows 2000. Usa el puerto 88 (TCP y UDP).

Analogía: un parque de diversiones. En la entrada muestras tu cédula una vez y te ponen una pulsera (el TGT). Con la pulsera vas a la taquilla de cada atracción y te dan un tiquete para esa atracción concreta (el ticket de servicio). El operador de la montaña rusa no ve tu cédula, solo el tiquete, y confía en él porque lo emitió la taquilla oficial.

Las piezas:

- KDC (Key Distribution Center): el servidor de confianza; en AD es el controlador de dominio. Tiene dos funciones lógicas: AS (Authentication Service) y TGS (Ticket Granting Service).
- Realm: el dominio de Kerberos, en mayúsculas (`EMPRESA.LOCAL`).
- Principal: identidad en Kerberos; para servicios se nombra con un SPN (Service Principal Name) como `HTTP/web01.empresa.local`.
- Llave de largo plazo: derivada de la contraseña de cada principal. El KDC conoce la de todos.
- `krbtgt`: cuenta especial cuya llave cifra todos los TGT. Quien la tiene, falsifica cualquier TGT.
- TGT (Ticket Granting Ticket): pulsera que prueba que ya te autenticaste; vale ~10 horas en AD.
- Ticket de servicio (ST, o "TGS ticket"): acceso a un servicio concreto, cifrado con la llave de ese servicio.

El flujo completo:

```
 Cliente (ana)                 KDC (controlador de dominio)              Servicio (web01)
      |                     [ AS ]                [ TGS ]                       |
      |                                                                         |
 1    |-- AS-REQ: "soy ana", hora cifrada con llave de ana -->|                 |
      |       (pre-autenticación)                             |                 |
 2    |<-- AS-REP: TGT (cifrado con llave de krbtgt) ---------|                 |
      |            + llave de sesión 1 (cifrada con llave de ana)               |
      |                                                                         |
 3    |-- TGS-REQ: TGT + autenticador + "quiero HTTP/web01" -------->|          |
 4    |<-- TGS-REP: ticket de servicio (cifrado con llave de web01) -|          |
      |             + llave de sesión 2                                         |
      |                                                                         |
 5    |-- AP-REQ: ticket de servicio + autenticador ------------------------------>|
 6    |<-- AP-REP (opcional, autenticación mutua) ---------------------------------|
```

Paso por paso:

1. AS-REQ: el cliente pide un TGT y manda la hora actual cifrada con una llave derivada de su contraseña. Eso es la pre-autenticación: prueba que conoce la contraseña sin enviarla.
2. AS-REP: el AS descifra la hora con la llave de ana que tiene guardada; si cuadra, devuelve el TGT (que ana no puede leer, está cifrado con la llave de `krbtgt`) y una llave de sesión cifrada con la llave de ana.
3. TGS-REQ: cuando ana quiere usar `web01`, manda el TGT y un autenticador (hora cifrada con la llave de sesión 1) al TGS.
4. TGS-REP: el TGS abre el TGT con la llave de `krbtgt`, saca la llave de sesión 1, valida el autenticador y emite un ticket de servicio cifrado con la llave de `web01`.
5. AP-REQ: ana presenta ese ticket a `web01`. El servicio lo descifra con su propia llave y confía, sin consultar al KDC.
6. AP-REP: opcionalmente el servidor demuestra su identidad al cliente (autenticación mutua).

Criterios numéricos: el desfase de reloj máximo es 5 minutos por defecto (los autenticadores llevan hora para impedir replays, ataques que reenvían un mensaje capturado); por eso Kerberos rompe si NTP falla. El TGT dura 10 horas y se renueva hasta 7 días en AD. El tipo de cifrado recomendado es AES-256 (RC4 está obsoleto).

Ejemplo en Linux contra un dominio de laboratorio:

```
$ kinit ana@EMPRESA.LOCAL
Password for ana@EMPRESA.LOCAL:
$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: ana@EMPRESA.LOCAL

Valid starting       Expires              Service principal
10/02/2026 09:00:00  10/02/2026 19:00:00  krbtgt/EMPRESA.LOCAL@EMPRESA.LOCAL
10/02/2026 09:05:12  10/02/2026 19:00:00  HTTP/web01.empresa.local@EMPRESA.LOCAL
```

La primera línea es el TGT (servicio `krbtgt`); la segunda, el ticket de servicio para la web.

Ataques conocidos, a nivel conceptual (se practican en labs como los de TryHackMe y PortSwigger listados abajo):

- Kerberoasting (ATT&CK T1558.003): cualquier usuario del dominio puede pedir un ticket de servicio para cualquier SPN; como va cifrado con la llave de la cuenta de servicio, se lleva offline y se crackea su contraseña. Defensa: contraseñas de servicio de 25+ caracteres o gMSA (cuentas administradas con contraseña rotada automáticamente), y AES en lugar de RC4.
- AS-REP Roasting: si una cuenta tiene desactivada la pre-autenticación, cualquiera pide su AS-REP y crackea offline la parte cifrada con su contraseña. Defensa: nunca desactivar la pre-autenticación.
- Golden Ticket: con la llave de `krbtgt` se fabrican TGT de cualquier usuario. Defensa: proteger los DC y rotar `krbtgt` dos veces tras un compromiso.
- Silver Ticket: con la llave de un servicio se fabrican tickets solo para ese servicio.
- Pass-the-Ticket: robar un ticket de la memoria de un equipo y reutilizarlo.

> [!NOTE]
> Kerberos autentica; no decide permisos. El ticket lleva los grupos del usuario (el PAC en AD), pero es el servicio quien decide qué permite con ellos.

```
KDC → servidor de confianza (el DC); contiene AS y TGS.
AS → autentica al usuario y entrega el TGT.
TGS → canjea un TGT por tickets de servicio.
TGT → "pulsera" cifrada con la llave de krbtgt; ~10 h.
Ticket de servicio → acceso a un SPN, cifrado con la llave del servicio.
SPN → nombre de un servicio en Kerberos (HTTP/web01...).
krbtgt → cuenta cuya llave firma todos los TGT; su robo = Golden Ticket.
Pre-autenticación → hora cifrada con la llave del usuario; sin ella, AS-REP Roasting.
Kerberoasting → pedir tickets de servicio y crackear offline la contraseña del servicio.
Puerto 88 → Kerberos; desfase máximo de reloj 5 min.
```

### RADIUS

RADIUS (Remote Authentication Dial-In User Service, RFC 2865) es un protocolo cliente-servidor AAA que centraliza la autenticación, la autorización y el registro de usuarios que se conectan a la red a través de equipos como puntos de acceso Wi-Fi, switches o concentradores VPN.

Problema que resuelve: una empresa con 40 puntos de acceso y 3 VPN no puede mantener 43 listas de usuarios. Con RADIUS, cada equipo de red (llamado NAS, Network Access Server, que es el cliente RADIUS) le pregunta al servidor central "¿dejo entrar a ana?", y el servidor consulta el directorio (AD, LDAP) y responde.

Analogía: el portero de una discoteca con una radio. No tiene la lista de invitados; le dice tu nombre a la oficina por radio y la oficina responde "pasa, zona VIP" o "no está".

```
 Laptop de ana        Punto de acceso (NAS)           Servidor RADIUS          Directorio
 (suplicante)         (cliente RADIUS)                (FreeRADIUS / NPS)       (AD / LDAP)
     |-- EAP: usuario --->|                                  |                     |
     |                    |-- Access-Request (UDP 1812) ---->|-- ¿ana válida? ---->|
     |                    |<-- Access-Challenge -------------|<-- sí, grupo wifi --|
     |<-- reto EAP -------|                                  |                     |
     |-- respuesta ------>|-- Access-Request --------------->|                     |
     |                    |<-- Access-Accept + VLAN 20 ------|                     |
     |<== conectada en VLAN 20 ==                            |                     |
     |                    |-- Accounting-Request (UDP 1813) ->| "ana inició sesión" |
```

Cómo funciona por dentro:

- Transporte UDP: puerto 1812 para autenticación y 1813 para accounting (los antiguos 1645/1646 aún aparecen en equipos viejos).
- El NAS y el servidor comparten un secreto (shared secret). Con él se ofusca solo el campo de contraseña (con MD5), no el paquete entero: el nombre de usuario y los atributos viajan en claro.
- Mensajes: Access-Request, Access-Accept, Access-Reject, Access-Challenge, y Accounting-Request/Response.
- La respuesta Accept trae atributos de autorización (VLAN asignada, tiempo máximo de sesión), así que RADIUS **mezcla** autenticación y autorización en el mismo paso.
- Es la pieza central de 802.1X (control de acceso por puerto) y de WPA2/WPA3-Enterprise, donde el método real de autenticación viaja dentro de EAP (EAP-TLS con certificados, PEAP con contraseña).

Ejemplo de prueba en laboratorio con FreeRADIUS:

```
$ radtest ana Secreto123 127.0.0.1 0 testing123
Sent Access-Request Id 42 from 0.0.0.0:51320 to 127.0.0.1:1812 length 73
Received Access-Accept Id 42 from 127.0.0.1:1812 to 127.0.0.1:51320 length 20
```

Debilidad reciente: Blast-RADIUS (CVE-2024-3596, 2024) mostró que un atacante en medio puede forjar un Access-Accept aprovechando colisiones de MD5. Mitigación: exigir el atributo Message-Authenticator en todos los paquetes, o llevar RADIUS dentro de TLS (RadSec, TCP 2083).

Su pariente es TACACS+ (Cisco), que usa TCP 49, cifra el cuerpo completo del paquete y separa autenticación, autorización y accounting; se usa sobre todo para administrar equipos de red (quién puede ejecutar qué comando en un router).

```
RADIUS → AAA para acceso a la red (Wi-Fi, VPN, 802.1X); UDP 1812/1813.
NAS → el equipo de red que actúa de cliente RADIUS.
Shared secret → secreto NAS-servidor; solo protege el campo de contraseña.
Access-Accept → respuesta positiva que trae atributos (VLAN, tiempo).
802.1X → control de acceso por puerto que usa RADIUS + EAP.
RadSec → RADIUS sobre TLS, TCP 2083.
TACACS+ → AAA de Cisco para administrar equipos; TCP 49; cifra todo; separa las tres A.
```

### LDAP

LDAP (Lightweight Directory Access Protocol, RFC 4511) es un protocolo de acceso a directorios que permite consultar y modificar una base de datos jerárquica de usuarios, grupos, equipos y otros objetos, y que también se usa para verificar credenciales.

Un directorio es una base de datos optimizada para leer mucho y escribir poco, organizada como un árbol. Active Directory, OpenLDAP y FreeIPA hablan LDAP. Aplicaciones como Jenkins, GitLab o un portal interno lo usan para "iniciar sesión con la cuenta de la empresa".

Analogía: la guía telefónica de una empresa organizada por país, sede y departamento. Para encontrar a Ana bajas por las ramas: Costa Rica, sede San José, Contabilidad, Ana.

Cómo se nombra cada objeto, con un DN (Distinguished Name), que es su ruta completa en el árbol:

```
dc=empresa,dc=local                     <- raíz (domain components)
 ├── ou=Usuarios                        <- unidad organizativa
 │    ├── cn=Ana Rojas                  <- DN: cn=Ana Rojas,ou=Usuarios,dc=empresa,dc=local
 │    └── cn=Luis Mora
 └── ou=Grupos
      └── cn=contabilidad               <- miembros: Ana Rojas
```

- `dc` (domain component), `ou` (organizational unit), `cn` (common name), `uid` (identificador de usuario).
- Las operaciones son Bind (autenticarse ante el directorio), Search, Compare, Add, Modify, Delete y Unbind.

La autenticación en LDAP es el Bind. Hay tres tipos:

- Anónimo: sin credenciales; debería estar deshabilitado o muy limitado.
- Simple: DN + contraseña **en texto claro** por la red, salvo que el canal vaya cifrado.
- SASL: delega en otro mecanismo, típicamente Kerberos (GSSAPI).

El patrón típico de una aplicación es "search + bind": se conecta con una cuenta de servicio, busca el DN del usuario por su `uid`, e intenta un Bind con ese DN y la contraseña que tecleó el usuario. Si el Bind funciona, la contraseña es correcta.

Puertos: 389 (LDAP, con o sin StartTLS), 636 (LDAPS, TLS desde el inicio), 3268/3269 (Global Catalog de AD, sin/con TLS). LDAPS se ve en [`10-criptografia`](../10-criptografia/).

Ejemplo:

```
$ ldapsearch -H ldaps://dc01.empresa.local -D "cn=svc-app,ou=Servicios,dc=empresa,dc=local" -W \
    -b "dc=empresa,dc=local" "(uid=ana)" cn memberOf
Enter LDAP Password:
dn: cn=Ana Rojas,ou=Usuarios,dc=empresa,dc=local
cn: Ana Rojas
memberOf: cn=contabilidad,ou=Grupos,dc=empresa,dc=local
```

Riesgo propio: la inyección LDAP. Si la aplicación arma el filtro concatenando lo que tecleó el usuario (`(uid=` + entrada + `)`), una entrada como `*)(uid=*` cambia la consulta. Se evita escapando los caracteres especiales `* ( ) \ NUL`.

```
LDAP → protocolo para consultar/modificar un directorio; TCP 389.
LDAPS → LDAP dentro de TLS; TCP 636.
DN → ruta completa del objeto en el árbol (cn=...,ou=...,dc=...).
Bind → operación de autenticación ante el directorio.
Simple bind → DN + contraseña en claro; solo sobre TLS.
SASL bind → delega en Kerberos u otro mecanismo.
Active Directory → directorio de Microsoft; habla LDAP y Kerberos.
Kerberos vs LDAP → Kerberos autentica con tickets; LDAP guarda y consulta identidades.
```

### SSO

SSO (Single Sign-On) es un esquema de autenticación en el que el usuario inicia sesión una vez ante un proveedor de identidad central y luego accede a varias aplicaciones independientes sin volver a teclear credenciales.

Problema que resuelve: un empleado con 25 aplicaciones SaaS tendría 25 contraseñas (que acabaría reutilizando), y TI tendría que dar de baja 25 cuentas cuando se va. Con SSO hay una sola cuenta, un solo lugar donde exigir MFA y un solo interruptor para desactivar a alguien.

Analogía: el brazalete de un hotel todo incluido. Te identificas una vez en recepción y el brazalete te abre el restaurante, la piscina y el bar; ninguno vuelve a pedirte el pasaporte.

Vocabulario común:

- IdP (Identity Provider): quien autentica (Okta, Microsoft Entra ID, Keycloak, Google).
- SP (Service Provider) o Relying Party: la aplicación que confía en el IdP (Slack, Salesforce, la intranet).
- Federación: relación de confianza previa entre IdP y SP, establecida intercambiando certificados o metadatos.

El riesgo inverso: el IdP se vuelve un punto único. Si roban la sesión del IdP, caen todas las aplicaciones; por eso es el lugar donde poner el MFA más fuerte.

### SAML

SAML 2.0 (Security Assertion Markup Language) es un estándar de federación basado en XML en el que el IdP emite una aserción firmada que le dice al SP quién es el usuario y qué atributos tiene.

Flujo típico iniciado por el SP:

```
 Navegador                     SP (app.empresa.com)                IdP (login.empresa.com)
    |-- GET /dashboard ------------->|                                      |
    |<-- 302 redirect + AuthnRequest-|                                      |
    |-- AuthnRequest --------------------------------------------------->  |
    |<-- página de login (contraseña + MFA) ------------------------------- |
    |-- credenciales ------------------------------------------------------>|
    |<-- formulario auto-POST con SAMLResponse (aserción XML firmada) ------|
    |-- POST /acs  SAMLResponse ---->|                                      |
    |                                | verifica firma con cert del IdP      |
    |<-- cookie de sesión de la app -|                                      |
```

- El navegador hace de mensajero: SP e IdP no hablan directamente.
- La aserción contiene el `NameID` (quién es), atributos (correo, grupos), validez (`NotBefore`, `NotOnOrAfter`, normalmente pocos minutos) y la audiencia (para qué SP vale).
- El SP debe validar la firma, la audiencia y el tiempo. Los ataques clásicos (XML Signature Wrapping) explotan SPs que validan la firma de un nodo pero leen los datos de otro.

Se usa sobre todo en SSO empresarial para aplicaciones web.

### OAuth2/OIDC

OAuth 2.0 (RFC 6749) es un marco de **autorización delegada** que permite a una aplicación obtener un token de acceso limitado para actuar sobre los recursos de un usuario en otro servicio, sin recibir su contraseña. OpenID Connect (OIDC) es una capa de **autenticación** construida encima de OAuth 2.0 que añade un ID Token para decirle a la aplicación quién es el usuario.

El problema de OAuth: una app de impresión de fotos quiere leer tus fotos de Google. Antes tendrías que darle tu contraseña de Google (con acceso a todo y para siempre). Con OAuth le das un token que solo permite `photos.read` y que puedes revocar.

Analogía: la llave de valet parking. Abre el carro y lo enciende, pero no abre la guantera ni la cajuela. No es tu llave maestra.

Roles: Resource Owner (tú), Client (la app de fotos), Authorization Server (emite tokens; p. ej. `accounts.google.com`), Resource Server (la API que guarda las fotos).

Flujo Authorization Code con PKCE, el recomendado hoy para web y móvil:

```
 App (client)                 Navegador / usuario            Authorization Server       API
   | genera code_verifier aleatorio; code_challenge = SHA256(verifier)                    |
   |-- redirect: /authorize?client_id&scope=photos.read&code_challenge -->|              |
   |                             |-- login + "¿permitir leer fotos?" ---->|              |
   |                             |<-- redirect a la app con ?code=abc ----|              |
   |<-- code=abc ----------------|                                        |              |
   |-- POST /token: code=abc + code_verifier ---------------------------->|              |
   |<-- access_token (+ refresh_token) (+ id_token si es OIDC) -----------|              |
   |-- GET /photos  Authorization: Bearer access_token ---------------------------------->|
```

- El `code` es de un solo uso y de corta vida; el intercambio por el token ocurre por un canal directo.
- PKCE (RFC 7636) evita que alguien que intercepte el `code` lo canjee: necesitaría el `code_verifier`, que nunca salió de la app.
- `scope` limita lo que el token permite. El access token suele durar minutos u horas; el refresh token sirve para pedir otro sin volver a loguear.
- El flujo Implicit (token directo en la URL) está desaconsejado.

OIDC agrega `scope=openid` y devuelve un ID Token, que es un JWT (JSON Web Token: JSON firmado en tres partes `cabecera.datos.firma` codificadas en base64url) con claims como:

```
{ "iss": "https://accounts.google.com", "sub": "1098765", "aud": "app-fotos",
  "email": "ana@ejemplo.com", "iat": 1790000000, "exp": 1790003600 }
```

"Iniciar sesión con Google" es OIDC, no OAuth a secas.

> [!WARNING]
> OAuth 2.0 no autentica: un access token dice "este portador puede leer fotos", no "este es Ana". Usar un access token como prueba de identidad es el error de diseño que OIDC vino a corregir con el ID Token.

```
SSO → un login en el IdP abre muchas aplicaciones.
IdP → quien autentica y emite aserciones/tokens.
SP / Relying Party → la app que confía en el IdP.
SAML 2.0 → aserciones XML firmadas; SSO empresarial web.
OAuth 2.0 → autorización delegada; access token con scopes; no dice quién eres.
OIDC → autenticación sobre OAuth; añade ID Token (JWT).
PKCE → code_verifier/code_challenge; impide canjear un code robado.
JWT → JSON firmado cabecera.datos.firma en base64url.
```

### Certificates

La autenticación con certificados es un método de autenticación en el que una entidad demuestra su identidad probando que posee la llave privada asociada a un certificado digital X.509 emitido por una autoridad en la que el verificador confía.

Un certificado es un documento firmado por una CA (Certificate Authority) que une una llave pública con una identidad (un dominio, un usuario, un equipo). La mecánica completa de CA, cadenas de confianza y revocación vive en [`10-criptografia`](../10-criptografia/); aquí interesa cómo se usa para autenticar.

Por qué existe: una contraseña es un secreto que se envía (o se demuestra) y se puede adivinar o robar. Un certificado nunca entrega su secreto: el verificador manda un reto y el dueño lo firma con su llave privada, que puede vivir en un chip del que no se puede extraer (tarjeta inteligente, TPM, llave USB).

Analogía: un pasaporte. Lo emite una autoridad que todos reconocen, lleva tu foto (la llave pública) y el agente lo acepta porque confía en quien lo emitió, no porque te conozca. La diferencia es que aquí además tienes que demostrar que eres la cara de la foto firmando un reto.

```
 Verificador                                       Dueño del certificado
    |-- "dame tu certificado" ------------------------->|
    |<-- certificado X.509 (llave pública + identidad) --|
    | ¿lo firmó una CA en la que confío? ¿vigente? ¿revocado?
    |-- reto aleatorio -------------------------------->|
    |<-- firma(reto) con llave privada ----------------|
    | verifica la firma con la llave pública del cert   |
    | -> identidad probada                              |
```

Usos:

- Tarjetas inteligentes (smart cards) y PIV/CAC: el certificado del empleado vive en la tarjeta; Windows lo usa para pedir el TGT de Kerberos (PKINIT).
- EAP-TLS en Wi-Fi corporativo: cada laptop tiene un certificado; no hay contraseña que robar.
- Certificados de SSH: una CA de SSH firma las llaves de usuarios y servidores, evitando distribuir `authorized_keys` a mano.
- mTLS entre servicios.

### mTLS

mTLS (mutual TLS) es una variante del protocolo TLS en la que, además de que el servidor presenta su certificado al cliente, el cliente presenta el suyo al servidor, de modo que ambos lados se autentican criptográficamente.

En TLS normal solo el servidor se identifica (por eso tu navegador sabe que habla con tu banco, pero el banco no sabe quién eres hasta que te logueas). En mTLS el servidor envía un `CertificateRequest` durante el handshake y el cliente responde con su certificado y una firma (`CertificateVerify`). Se usa en APIs entre empresas, en microservicios (las service mesh como Istio lo activan entre todos los pods) y en dispositivos IoT.

Ejemplo contra una API de laboratorio que exige certificado de cliente:

```
$ curl https://api.interna.test/estado
curl: (56) OpenSSL SSL_read: error:0A00045C:SSL routines::tlsv13 alert certificate required

$ curl --cert cliente.crt --key cliente.key --cacert ca-interna.crt https://api.interna.test/estado
{"estado":"ok","cliente":"CN=servicio-facturacion"}
```

El servidor lee la identidad del campo `CN` o `SAN` del certificado y la usa para autorizar.

Desventaja: gestionar el ciclo de vida (emitir, renovar, revocar miles de certificados). Si un certificado de cliente vence a las 3 a. m. el servicio cae; por eso se automatiza con vidas cortas (horas o días) y renovación automática.

```
Certificado X.509 → llave pública + identidad, firmado por una CA.
Autenticación con certificado → firmar un reto con la llave privada del certificado.
Smart card / PIV → certificado en un chip; PKINIT lo usa con Kerberos.
EAP-TLS → Wi-Fi corporativo autenticado con certificados.
TLS → solo el servidor se autentica.
mTLS → servidor y cliente se autentican con certificados.
```

### Local Auth

La autenticación local es el método en el que el propio equipo verifica las credenciales contra una base de datos guardada en su disco, sin consultar a ningún servidor de la red.

Es lo que pasa cuando inicias sesión en tu laptop personal, en un servidor Linux sin dominio o en un router con su usuario `admin`. Ventaja: funciona sin red. Desventaja: cada equipo tiene sus propias cuentas, que nadie revisa ni da de baja de forma central, y la base de datos está en el mismo disco que el atacante puede robar.

Analogía: la llave de tu casa frente al llavero de una oficina. La llave de tu casa la controlas tú, pero si la pierdes nadie se entera; el llavero de la oficina está en un sistema central que sabe quién tiene qué.

En Linux:

- `/etc/passwd` guarda las cuentas (legible por todos) y `/etc/shadow` los hashes (legible solo por root).
- PAM (Pluggable Authentication Modules) es la capa que decide cómo se autentica cada servicio (`login`, `sshd`, `sudo`); con PAM se puede añadir TOTP o conectar a LDAP sin tocar el programa.
- El hash lleva el algoritmo y la sal en el propio campo:

```
$ sudo grep ana /etc/shadow
ana:$y$j9T$Zx3k...sal...$Qk2...hash...:20000:0:99999:7:::
     |  |    |               |
     |  |    sal             hash
     |  parámetros de coste
     algoritmo: $y$ = yescrypt, $6$ = SHA-512-crypt, $1$ = MD5-crypt (obsoleto)
```

En Windows:

- La base SAM (Security Account Manager) guarda las cuentas locales con su hash NT, que es MD4 de la contraseña **sin sal**: dos usuarios con la misma contraseña tienen el mismo hash.
- LSASS es el proceso que verifica el login y mantiene credenciales en memoria; volcar su memoria (técnica de Mimikatz) es un objetivo típico de los atacantes.
- Como Windows acepta el hash NT directamente en NTLM, robarlo permite Pass-the-Hash: autenticarse sin conocer la contraseña. Por eso la cuenta Administrador local con la misma contraseña en 300 equipos es un desastre; LAPS (Local Administrator Password Solution) le pone una contraseña distinta y rotada a cada uno.

Ejemplo: un atacante consigue una copia de `/etc/shadow` de un servidor con 50 usuarios. Si los hashes fueran MD5 sin sal, un diccionario de 10 millones de contraseñas se prueba contra los 50 a la vez en segundos. Con yescrypt y sal por usuario, tiene que repetir el trabajo por cada usuario y cada intento es miles de veces más lento.

> [!TIP]
> Para las cuentas locales que deben existir (root, Administrador, admin del router): contraseña única por equipo, deshabilitar el login remoto directo con ellas y registrar su uso.

```
Local auth → el equipo verifica contra su propio disco; sin red, sin control central.
/etc/shadow → hashes de Linux con algoritmo y sal ($y$ yescrypt, $6$ SHA-512).
PAM → capa modular que define cómo autentica cada servicio en Linux.
SAM → base local de Windows; hash NT = MD4 sin sal.
LSASS → proceso de Windows que guarda credenciales en memoria.
Pass-the-Hash → usar el hash NT robado sin conocer la contraseña.
LAPS → contraseña de admin local única y rotada por equipo.
```

## Recursos para aprender y practicar

### Videos

- [Authentication, Authorization, and Accounting - CompTIA Security+ SY0-701 - 1.2](https://www.youtube.com/watch?v=AhaZtj5P2a8) — Professor Messer; AAA y la diferencia autenticación/autorización (nodo Authentication vs Authorization).
- [Multifactor Authentication - CompTIA Security+ SY0-701 - 4.6](https://www.youtube.com/watch?v=MpIzA4fNWew) — Professor Messer; factores, tokens y métodos de MFA (nodo MFA & 2FA).
- [Identity and Access Management - CompTIA Security+ SY0-701 - 4.6](https://www.youtube.com/watch?v=ZoOyyqhptik) — Professor Messer; SSO, LDAP, SAML, OAuth y federación (nodo SSO y LDAP).
- [How TOTP (Time-based One-time Password Algorithm) Works for 2 Factor Authentication](https://www.youtube.com/watch?v=jxxtVzVLm3c) — Lawrence Systems; el algoritmo TOTP por dentro (nodo MFA & 2FA).
- [How Passkeys Work - Computerphile](https://www.youtube.com/watch?v=xYfiOnufBSk) — Computerphile; WebAuthn y passkeys con firma por dominio (nodo MFA & 2FA).
- [How FIDO2 Works And Would It Stop MFA Fatigue Attacks?](https://www.youtube.com/watch?v=F_E2LZK-bFk) — Lawrence Systems; FIDO2 frente a la fatiga MFA (nodo MFA & 2FA).
- [Taming Kerberos - Computerphile](https://www.youtube.com/watch?v=qW361k3-BtU) — Computerphile; Kerberos explicado desde cero con TGT y tickets (nodo Kerberos).
- [Kerberos Authentication Explained | A deep dive](https://www.youtube.com/watch?v=5N242XcKAsM) — Destination Certification; el flujo AS/TGS/AP paso a paso (nodo Kerberos).
- [CertMike Explains RADIUS](https://www.youtube.com/watch?v=oTSF4SuzOa4) — Mike Chapple; RADIUS y AAA (nodo RADIUS).
- [What is LDAP (Lightweight Directory Access Protocol)?](https://www.youtube.com/watch?v=vy3e6ekuqqg) — CBT Nuggets; estructura del directorio y operaciones (nodo LDAP).
- [A Developer's Guide to SAML](https://www.youtube.com/watch?v=l-6QSEqDJPo) — OktaDev; flujo SAML IdP/SP con aserciones (nodo SSO).
- [OAuth 2.0 and OpenID Connect (in plain English)](https://www.youtube.com/watch?v=996OiexHze0) — OktaDev (Nate Barbettini); OAuth vs OIDC y el authorization code flow (nodo SSO).
- [What is Mutual TLS (mTLS)?](https://www.youtube.com/watch?v=RZt9xdVh9Qk) — F5 DevCentral; mTLS y certificados de cliente (nodo Certificates).
- [How NOT to Store Passwords! - Computerphile](https://www.youtube.com/watch?v=8ZtInClXe1Q) — Computerphile; cómo se guardan las contraseñas en local (nodo Local Auth).

### Lectura y documentación

- [NIST SP 800-63B: Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html) — niveles de seguridad, tipos de autenticador, estado del SMS y resistencia al phishing.
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) — buenas prácticas de login, mensajes de error y bloqueo.
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html) — tipos de factor, sus debilidades y recuperación.
- [RFC 6238: TOTP](https://www.rfc-editor.org/rfc/rfc6238) — el algoritmo y los vectores de prueba.
- [W3C Web Authentication Level 3](https://www.w3.org/TR/webauthn-3/) y [passkeys.dev](https://passkeys.dev/) — especificación de WebAuthn y guía práctica de passkeys.
- [MITRE ATT&CK T1621: Multi-Factor Authentication Request Generation](https://attack.mitre.org/techniques/T1621/) — la fatiga MFA con casos reales y mitigaciones.
- [RFC 4120: The Kerberos Network Authentication Service (V5)](https://www.rfc-editor.org/rfc/rfc4120) y [Kerberos authentication overview (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) — el protocolo y su uso en Windows.
- [MITRE ATT&CK T1558: Steal or Forge Kerberos Tickets](https://attack.mitre.org/techniques/T1558/) — Golden/Silver Ticket, Kerberoasting ([T1558.003](https://attack.mitre.org/techniques/T1558/003/)) y AS-REP Roasting.
- [RFC 2865: RADIUS](https://www.rfc-editor.org/rfc/rfc2865) — formato de paquetes y atributos.
- [RFC 4511: LDAP Protocol](https://www.rfc-editor.org/rfc/rfc4511) y [RFC 4513: LDAP Authentication Methods](https://www.rfc-editor.org/rfc/rfc4513) — operaciones, Bind y uso de TLS.
- [SAML 2.0 Technical Overview (OASIS)](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html) y [OWASP SAML Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html) — flujos SAML y cómo validarlos.
- [RFC 6749: OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749) y [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) — los estándares de OAuth y OIDC.
- [RFC 8705: OAuth 2.0 Mutual-TLS Client Authentication](https://www.rfc-editor.org/rfc/rfc8705) — mTLS aplicado a clientes OAuth.

### Práctica

- [PortSwigger Web Security Academy: Authentication](https://portswigger.net/web-security/authentication) (incluye [labs de MFA](https://portswigger.net/web-security/authentication/multi-factor)) y [OAuth 2.0](https://portswigger.net/web-security/oauth) — labs gratuitos de login, 2FA y OAuth (nodos MFA & 2FA y SSO).
- [TryHackMe: Enumeration & Brute Force](https://tryhackme.com/room/enumerationbruteforce) — gratis; fallos en login, recuperación de contraseña y autenticación (nodos MFA & 2FA y Local Auth).
- [TryHackMe: OWASP Top 10 2025: IAAA Failures](https://tryhackme.com/room/owasptopten2025one) — gratis; fallos de identificación, autenticación y autorización (nodos Authentication vs Authorization y SSO).
- [TryHackMe: Monitoring Active Directory](https://tryhackme.com/room/monitoringactivedirectory) — gratis; cómo se ven Kerberos y LDAP en los logs de un dominio (nodos Kerberos y LDAP).
- [TryHackMe: Attacktive Directory](https://tryhackme.com/room/attacktivedirectory) — gratis; Kerberos en un dominio de laboratorio de punta a punta (nodo Kerberos).
- [Ejercicio 1: Reproduce y genera códigos TOTP](ejercicios.md#ejercicio-1-reproduce-y-genera-códigos-totp) — reproducir el vector `94287082` de RFC 6238 y comprobar que tu terminal y una app autenticadora dan el mismo código (nodo MFA & 2FA).
- [Ejercicio 2: Autentica contra FreeRADIUS y captura el tráfico](ejercicios.md#ejercicio-2-autentica-contra-freeradius-y-captura-el-tráfico) — Accept y Reject con `radtest` y ver en la captura de UDP 1812 qué va en claro y qué protege el shared secret (nodo RADIUS).
- [Ejercicio 3: Monta OpenLDAP y observa un bind simple en claro](ejercicios.md#ejercicio-3-monta-openldap-y-observa-un-bind-simple-en-claro) — directorio con dos OUs y tres usuarios, y la contraseña del bind leída en Wireshark (nodo LDAP).
- [Ejercicio 4: Exige certificado de cliente en nginx con mTLS](ejercicios.md#ejercicio-4-exige-certificado-de-cliente-en-nginx-con-mtls) — CA propia con `openssl` y tres pruebas con curl: sin certificado, con el tuyo y con uno de otra CA (nodo Certificates).

## Cuadro resumen

Todo lo visto, en una línea por término.

Authentication vs Authorization

```
Identificación → afirmar una identidad ("soy ana").
Authentication (AuthN) → demostrar esa identidad; falla con HTTP 401.
Authorization (AuthZ) → decidir qué puede hacer; falla con HTTP 403.
Accounting → registrar qué hizo, cuándo y desde dónde.
AAA → marco que junta las tres; RADIUS y TACACS+ lo implementan.
DAC → el dueño decide permisos (rwx de Linux).
MAC → una política central impone etiquetas (SELinux).
RBAC → permisos por rol, usuarios reciben roles.
```

MFA & 2FA

```
Algo que sabes → contraseña, PIN; se roba con phishing o filtraciones.
Algo que tienes → teléfono, llave FIDO2, tarjeta inteligente.
Algo que eres → biometría; no se puede cambiar si se filtra.
2FA → exactamente dos factores de categorías distintas.
MFA → dos o más categorías distintas; contraseña + pregunta secreta NO es MFA.
TOTP → HMAC(secreto, hora/30 s) → 6 dígitos; phisheable.
HOTP → igual que TOTP pero con contador de usos.
FIDO2 → WebAuthn + CTAP2; firma ligada al dominio; resistente al phishing.
Passkey → credencial FIDO2 descubrible y sincronizable.
MFA fatigue → bombardeo de pushes hasta que aceptas (ATT&CK T1621).
Number matching → teclear el número de la pantalla; frena la fatiga MFA.
SIM swapping → robar tu número para recibir tus SMS.
AiTM → proxy de phishing que reenvía código y roba la cookie de sesión.
Phisheable → SMS, TOTP, push: el usuario puede entregar el código.
Resistente al phishing → FIDO2/passkey, mTLS: la prueba solo vale en el dominio real.
```

Authentication Methodologies

```
KDC → servidor de confianza (el DC); contiene AS y TGS.
AS → autentica al usuario y entrega el TGT.
TGS → canjea un TGT por tickets de servicio.
TGT → "pulsera" cifrada con la llave de krbtgt; ~10 h.
Ticket de servicio → acceso a un SPN, cifrado con la llave del servicio.
SPN → nombre de un servicio en Kerberos (HTTP/web01...).
krbtgt → cuenta cuya llave firma todos los TGT; su robo = Golden Ticket.
Pre-autenticación → hora cifrada con la llave del usuario; sin ella, AS-REP Roasting.
Kerberoasting → pedir tickets de servicio y crackear offline la contraseña del servicio.
Puerto 88 → Kerberos; desfase máximo de reloj 5 min.
RADIUS → AAA para acceso a la red (Wi-Fi, VPN, 802.1X); UDP 1812/1813.
NAS → el equipo de red que actúa de cliente RADIUS.
Shared secret → secreto NAS-servidor; solo protege el campo de contraseña.
Access-Accept → respuesta positiva que trae atributos (VLAN, tiempo).
802.1X → control de acceso por puerto que usa RADIUS + EAP.
RadSec → RADIUS sobre TLS, TCP 2083.
TACACS+ → AAA de Cisco para administrar equipos; TCP 49; cifra todo; separa las tres A.
LDAP → protocolo para consultar/modificar un directorio; TCP 389.
LDAPS → LDAP dentro de TLS; TCP 636.
DN → ruta completa del objeto en el árbol (cn=...,ou=...,dc=...).
Bind → operación de autenticación ante el directorio.
Simple bind → DN + contraseña en claro; solo sobre TLS.
SASL bind → delega en Kerberos u otro mecanismo.
Active Directory → directorio de Microsoft; habla LDAP y Kerberos.
Kerberos vs LDAP → Kerberos autentica con tickets; LDAP guarda y consulta identidades.
SSO → un login en el IdP abre muchas aplicaciones.
IdP → quien autentica y emite aserciones/tokens.
SP / Relying Party → la app que confía en el IdP.
SAML 2.0 → aserciones XML firmadas; SSO empresarial web.
OAuth 2.0 → autorización delegada; access token con scopes; no dice quién eres.
OIDC → autenticación sobre OAuth; añade ID Token (JWT).
PKCE → code_verifier/code_challenge; impide canjear un code robado.
JWT → JSON firmado cabecera.datos.firma en base64url.
Certificado X.509 → llave pública + identidad, firmado por una CA.
Autenticación con certificado → firmar un reto con la llave privada del certificado.
Smart card / PIV → certificado en un chip; PKINIT lo usa con Kerberos.
EAP-TLS → Wi-Fi corporativo autenticado con certificados.
TLS → solo el servidor se autentica.
mTLS → servidor y cliente se autentican con certificados.
Local auth → el equipo verifica contra su propio disco; sin red, sin control central.
/etc/shadow → hashes de Linux con algoritmo y sal ($y$ yescrypt, $6$ SHA-512).
PAM → capa modular que define cómo autentica cada servicio en Linux.
SAM → base local de Windows; hash NT = MD4 sin sal.
LSASS → proceso de Windows que guarda credenciales en memoria.
Pass-the-Hash → usar el hash NT robado sin conocer la contraseña.
LAPS → contraseña de admin local única y rotada por equipo.
```
