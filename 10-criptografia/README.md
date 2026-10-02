# Criptografía

## Conceptos previos

- Texto plano (plaintext): el dato legible original, antes de protegerlo.
- Texto cifrado (ciphertext): el dato después de cifrarlo; sin la llave parece ruido.
- Llave (key): el valor secreto que controla el cifrado; el algoritmo es público y la seguridad depende solo de la llave (principio de Kerckhoffs).
- Bit de seguridad: medida de cuánto cuesta romper algo por fuerza bruta; "128 bits" significa unos 2^128 intentos, imposible con la tecnología actual.
- Fuerza bruta: probar todas las llaves o contraseñas posibles hasta acertar.
- Integridad: garantía de que un dato no fue alterado. Confidencialidad: que solo lo lean los autorizados. Autenticidad: que viene de quien dice venir. No repudio: que el autor no puede negar haberlo enviado (repaso de la tríada CIA en [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/)).
- Aritmética modular: operar "con reloj"; `17 mod 5 = 2` porque 17 dividido entre 5 deja residuo 2.
- Ataque man-in-the-middle (MITM): alguien que se coloca entre dos partes y lee o altera lo que se dicen.
- Handshake: el intercambio inicial de mensajes con el que dos equipos acuerdan cómo van a hablar.
- Puertos y protocolos base (FTP, DNS, HTTP): repaso en [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/).

## Basics of Cryptography

### Hashing

Una **función hash** criptográfica es un algoritmo que transforma una entrada de cualquier tamaño en una huella de tamaño fijo, de forma que es imposible en la práctica volver atrás o encontrar dos entradas con la misma huella.

Existe para comprobar integridad sin guardar ni enviar el dato completo: verificar que una ISO descargada no fue alterada, guardar contraseñas sin guardarlas, firmar documentos (se firma el hash, no el documento entero) y encadenar bloques en Git o en una blockchain.

Analogía: la huella dactilar de una persona. Es pequeña comparada con la persona, identifica a una sola, y teniendo la huella no puedes reconstruir a la persona.

Propiedades que debe cumplir, en este orden:

- Determinista: la misma entrada da siempre la misma salida.
- Tamaño fijo: 1 byte o 4 GB producen una huella del mismo largo (256 bits en SHA-256).
- Resistencia a preimagen: dado un hash, no se puede encontrar una entrada que lo produzca (es "de un solo sentido").
- Resistencia a segunda preimagen: dada una entrada, no se puede encontrar otra distinta con el mismo hash.
- Resistencia a colisiones: no se puede encontrar ningún par de entradas con el mismo hash. Por la paradoja del cumpleaños, un hash de n bits ofrece solo n/2 bits contra colisiones: SHA-256 da 128 bits de seguridad, MD5 apenas 64 en teoría (y mucho menos en la práctica).
- Efecto avalancha: cambiar un solo bit de la entrada cambia alrededor de la mitad de los bits de la salida.

Ejemplo del efecto avalancha (solo cambia la mayúscula):

```
$ printf 'hola' | sha256sum
b221d9dbb083a7f33428d7c2a3c3198ae925614d70210e28716ccaa7cd4ddb79  -
$ printf 'Hola' | sha256sum
e633f4fc79badea1dc5db970cf397c8248bac47cc3acf9915ba60b5d76b0e88f  -
```

Los algoritmos que hay que conocer:

- MD5: 128 bits. Roto desde 2004 (colisiones en segundos en una laptop). Solo sirve como checksum contra corrupción accidental, nunca para seguridad.
- SHA-1: 160 bits. Colisión práctica demostrada en 2017 (SHAttered: dos PDFs distintos con el mismo SHA-1). Obsoleto para firmas y certificados; Git migra a SHA-256.
- SHA-2 (SHA-256, SHA-384, SHA-512): el estándar actual (FIPS 180-4). Sin ataques prácticos.
- SHA-3: diseño interno distinto (esponja Keccak); alternativa por si algún día cae SHA-2.

```
$ printf 'hola' | md5sum
4d186321c1a7f0f354b297e8914ab240  -
$ printf 'hola' | sha1sum
99800b85d3383e3a2fb45eb7d0066a4879a9dad0  -
```

El largo del hash delata el algoritmo: 32 caracteres hex = 128 bits (MD5), 40 = 160 bits (SHA-1), 64 = 256 bits (SHA-256).

### Hash rápido vs hash para contraseñas

Una función de hash para contraseñas (o función de derivación de llaves) es un hash diseñado a propósito para ser lento y configurable, de modo que cada intento de adivinar le cueste caro al atacante.

SHA-256 está diseñado para ser **rápido**: eso es bueno para verificar un archivo de 4 GB y malo para contraseñas, porque el atacante también calcula rápido. Una GPU de gama alta prueba del orden de decenas de miles de millones de SHA-256 por segundo. Por eso las contraseñas se guardan con funciones de derivación lentas a propósito y configurables:

- bcrypt (1999): tiene un factor de coste; cada +1 duplica el tiempo. OWASP pide coste 10 como mínimo. Solo usa los primeros 72 bytes de la contraseña.
- scrypt: además de lento, gasta mucha memoria, lo que encarece los ataques con GPU y hardware dedicado.
- Argon2id (ganador del Password Hashing Competition 2015, RFC 9106): lo recomendado hoy. OWASP sugiere como mínimo 19 MiB de memoria, 2 iteraciones y 1 hilo.
- PBKDF2-HMAC-SHA256: aceptado cuando se exige cumplimiento FIPS, con 600 000 iteraciones según OWASP.

Ejemplo: un atacante con una GPU que hace 10 000 millones de MD5 por segundo prueba un diccionario de 10 millones de palabras en una milésima de segundo. Contra bcrypt con coste 12, la misma GPU hace del orden de miles por segundo: el mismo diccionario le toma horas por cada usuario.

```
$ python3 -c 'import bcrypt; print(bcrypt.hashpw(b"Secreto123", bcrypt.gensalt(12)))'
b'$2b$12$Q9m1yWb7sF0t...22 caracteres de sal...31 caracteres de hash'
      |   |
      |   coste 12 = 2^12 rondas
      versión del algoritmo
```

> [!WARNING]
> Hashear no es cifrar: un hash no tiene llave ni se "descifra". Cuando alguien dice que "descifró" un hash, en realidad adivinó la entrada probando candidatos hasta que una produjo el mismo hash.

```
Hash → huella de tamaño fijo; de un solo sentido.
Preimagen → no se puede ir del hash a la entrada.
Colisión → dos entradas con el mismo hash; un hash de n bits da n/2 bits contra colisiones.
Avalancha → 1 bit distinto en la entrada cambia ~50 % de la salida.
MD5 → 128 bits; roto; solo checksum accidental.
SHA-1 → 160 bits; colisión práctica (SHAttered 2017); obsoleto.
SHA-256 → 256 bits; estándar actual; rápido, NO apto solo para contraseñas.
bcrypt → hash lento con factor de coste (mínimo 10); límite 72 bytes.
Argon2id → hash lento y costoso en memoria; recomendado para contraseñas.
```

### Salting

Una **sal** (salt) es un valor aleatorio, único por cada contraseña y no secreto, que se mezcla con la contraseña antes de calcular el hash para que dos contraseñas iguales produzcan hashes distintos.

Sin sal aparecen dos problemas. Primero, las tablas precalculadas: alguien calcula una sola vez el hash de 1 000 millones de contraseñas comunes y luego cualquier hash filtrado se busca en esa tabla en milisegundos (las rainbow tables son una versión comprimida de esa idea). Segundo, el patrón: si 200 usuarios tienen `password`, los 200 hashes son idénticos y el atacante crackea uno y tiene los 200.

Analogía: dos personas que hornean el mismo pastel con la misma receta obtienen pasteles idénticos. Si cada una añade un ingrediente secreto distinto al azar (y lo anota en la etiqueta), los pasteles salen diferentes, aunque cualquiera pueda leer la etiqueta.

Cómo funciona:

```
REGISTRO
  sal = aleatorio(16 bytes)            # distinta por usuario
  guardar: usuario, sal, hash(sal + contraseña)

LOGIN
  leer sal del usuario
  comparar hash(sal + contraseña_tecleada) con el guardado
```

La sal se guarda en claro junto al hash (bcrypt y Argon2 la incrustan en el propio string). No necesita ser secreta: su trabajo no es esconderse, es obligar al atacante a atacar cada hash por separado. NIST pide al menos 32 bits; lo habitual son 16 bytes (128 bits) generados con un generador aleatorio criptográfico.

Ejemplo con números redondos: dos usuarios con la contraseña `password`.

```
sin sal:
  ana   sha256("password")       = 5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8
  luis  sha256("password")       = 5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8  <- idéntico

con sal:
  ana   sha256("x9Kq"+"password") = 551b654600ecfa5a52a4faf284b8c00938b4dc20ccfb6d37658e1d341c491f3c
  luis  sha256("7fPz"+"password") = e874586baa6103cac0cc256cbc5af15bcd160dc509825dcf249aa726949e5b41
```

(Las sales de 4 caracteres son para que el ejemplo se lea; en la realidad son de 16 bytes.)

Una pimienta (pepper) es otro valor que se mezcla, pero secreto y común a todos, guardado fuera de la base de datos (en un HSM o en la configuración del servidor). Si roban solo la base de datos, sin la pimienta no pueden ni empezar.

> [!NOTE]
> La sal no hace más difícil adivinar una contraseña débil concreta: `123456` con sal se crackea igual de rápido. Lo que impide es el ataque masivo con tablas y el atacar a todos a la vez; contra contraseñas débiles lo que ayuda es el hash lento.

```
Sal → aleatoria, única por usuario, pública; rompe tablas precalculadas y patrones.
Rainbow table → tabla precalculada hash → contraseña; inútil contra hashes con sal.
Pimienta → secreto común guardado fuera de la base de datos.
```

### Key Exchange

El **intercambio de llaves** (key exchange) es un protocolo criptográfico que permite a dos partes que nunca se han visto acordar una llave secreta compartida hablando por un canal que cualquiera puede escuchar.

El problema: el cifrado simétrico es rápido, pero ambos lados necesitan la misma llave. ¿Cómo se la mandas a un servidor de otro continente si todo lo que envías puede ser leído? Diffie-Hellman (1976) fue la primera solución pública.

Analogía de la pintura: Ana y Luis acuerdan en público un color base, amarillo. Cada uno elige en secreto un color propio (Ana rojo, Luis azul), lo mezcla con el amarillo y envía la mezcla en público. Cada uno añade su color secreto a la mezcla que recibió. Ambos terminan con amarillo + rojo + azul. Quien espió solo vio amarillo, amarillo+rojo y amarillo+azul, y "desmezclar" pintura es muy difícil. Esa dificultad de deshacer es, en matemáticas, el logaritmo discreto.

### Diffie-Hellman con números pequeños

Diffie-Hellman es un protocolo de intercambio de llaves en el que cada parte combina un número secreto propio con valores públicos, de modo que ambas llegan al mismo resultado sin haberlo enviado nunca. Con números de juguete:

```
Público (todos lo ven):   p = 23 (primo)     g = 5 (generador)

Ana elige secreto a = 6                     Luis elige secreto b = 15
A = g^a mod p = 5^6 mod 23 = 8             B = g^b mod p = 5^15 mod 23 = 19

          Ana ---------- A = 8 ---------->  Luis
          Ana <--------- B = 19 ----------  Luis

Ana calcula: s = B^a mod p = 19^6 mod 23 = 2
Luis calcula: s = A^b mod p = 8^15 mod 23 = 2
                    -> llave compartida s = 2
```

- `p` es un primo y `g` un número base; los dos son públicos.
- `a` y `b` nunca salen de cada equipo.
- Funciona porque `(g^a)^b = (g^b)^a = g^(ab)`.
- El espía ve `p=23, g=5, A=8, B=19`. Para sacar `s` necesita `a` o `b`, es decir, resolver "¿a qué potencia elevo 5 para obtener 8 módulo 23?". Con 23 se prueba a mano; con un `p` de 2048 bits es inviable.

Criterios reales: DH clásico (finite field) con `p` de al menos 2048 bits; ECDH (sobre curvas elípticas) con X25519 o P-256, que da la misma seguridad con llaves de 256 bits. TLS 1.3 solo permite variantes efímeras (DHE/ECDHE): se generan `a` y `b` nuevos en cada conexión y se tiran después. Eso da **forward secrecy** (secreto hacia adelante): si mañana roban la llave privada del servidor, las conversaciones grabadas hoy siguen siendo ilegibles.

Límite: DH por sí solo no autentica. Un MITM puede hacer un DH con Ana y otro con Luis y quedarse en medio leyendo todo. Por eso en TLS el servidor firma sus valores DH con su llave privada, respaldada por un certificado (ver PKI).

Hacia el futuro: los computadores cuánticos romperían DH y RSA. NIST publicó en 2024 ML-KEM (FIPS 203), y navegadores y servidores ya usan un intercambio híbrido (X25519 + ML-KEM-768) en TLS 1.3.

```
Key exchange → acordar una llave secreta por un canal público.
Diffie-Hellman → s = g^(ab) mod p; seguro por el logaritmo discreto.
ECDH → DH sobre curvas elípticas (X25519, P-256); llaves más cortas.
Efímero (DHE/ECDHE) → llaves nuevas por conexión; da forward secrecy.
Forward secrecy → robar la llave del servidor no descifra tráfico pasado.
DH sin autenticar → vulnerable a MITM; se arregla firmando con certificado.
```

### Private vs Public Keys

La criptografía **simétrica** es un tipo de cifrado que usa la misma llave privada para cifrar y descifrar; la criptografía **asimétrica** (o de llave pública) es un tipo de cifrado que usa un par de llaves matemáticamente relacionadas, una pública que se reparte y una privada que nunca sale de su dueño.

Las dos existen porque se complementan. La simétrica es muy rápida (AES cifra gigabytes por segundo con soporte del procesador) pero exige compartir la llave antes. La asimétrica resuelve el reparto (la llave pública se puede publicar en un cartel) pero es entre 100 y 1000 veces más lenta. En la práctica se usan juntas: la asimétrica para acordar o transportar una llave, la simétrica para cifrar los datos. Eso se llama cifrado híbrido y es lo que hacen TLS, SSH, PGP y S/MIME.

Analogía: la simétrica es una caja fuerte con una llave copiada para cada persona; la asimétrica es un buzón con ranura. Cualquiera puede echar una carta por la ranura (llave pública), pero solo el dueño tiene la llave que abre el buzón (llave privada).

### Simétrica

Un algoritmo simétrico es un cifrado en el que la misma llave cifra y descifra, como un candado con llaves copiadas. Los que hay que conocer:

- AES (Advanced Encryption Standard): bloque de 128 bits, llaves de 128, 192 o 256 bits. El estándar.
- ChaCha20: cifrado de flujo rápido sin hardware especial; común en móviles.
- DES (56 bits) y 3DES: obsoletos. RC4: roto.
- Modos de operación: AES-GCM y ChaCha20-Poly1305 son AEAD (cifran y además autentican, detectan alteraciones); ECB es el modo a evitar porque bloques iguales dan cifrados iguales y se ven patrones.
- Problema de escala: con n personas que quieren hablar de dos en dos hacen falta n(n-1)/2 llaves; para 100 personas, 4 950 llaves. Con asimétrica, 100 pares.

### Asimétrica: RSA

RSA es un algoritmo de llave pública cuya seguridad descansa en lo difícil que es factorizar el producto de dos primos grandes. Con números de juguete:

```
1. Elegir dos primos:    p = 61, q = 53
2. n = p * q           = 3233          (parte de ambas llaves)
3. φ(n) = (p-1)(q-1)   = 3120
4. Elegir e = 17        (coprimo con 3120)
5. d tal que e*d mod φ(n) = 1  ->  d = 2753

Llave pública:  (e = 17,  n = 3233)
Llave privada:  (d = 2753, n = 3233)

Cifrar m = 65:     c = 65^17   mod 3233 = 2790
Descifrar c=2790:  m = 2790^2753 mod 3233 = 65
```

Quien conoce `n = 3233` y quisiera `d` necesita `p` y `q`. Factorizar 3233 es trivial; factorizar un `n` de 2048 bits (617 dígitos) no lo es. Tamaños: 2048 bits como mínimo hoy, 3072 para 128 bits de seguridad. RSA real añade relleno aleatorio (OAEP para cifrar, PSS para firmar); el "RSA de libro" de arriba es inseguro tal cual.

### Asimétrica: ECC

ECC (Elliptic Curve Cryptography) es una familia de algoritmos de llave pública basada en operaciones sobre puntos de una curva elíptica, que logra la misma seguridad que RSA con llaves mucho más cortas: una llave ECC de 256 bits equivale a RSA de 3072 bits. Se usa en ECDH (intercambio), ECDSA y Ed25519 (firmas). Llaves cortas = handshakes más rápidos, ideal para móviles e IoT.

```
$ ssh-keygen -t ed25519 -C "ana@laptop"
Generating public/private ed25519 key pair.
Your identification has been saved in /home/ana/.ssh/id_ed25519       <- privada, nunca se comparte
Your public key has been saved in /home/ana/.ssh/id_ed25519.pub       <- pública, va al servidor
```

### Firmas digitales

Una **firma digital** es un mecanismo criptográfico que usa la llave privada del autor para producir un valor que cualquiera puede verificar con su llave pública, probando quién firmó y que el contenido no cambió.

Es la asimétrica "al revés": para cifrar se usa la pública del destinatario; para firmar, la privada del autor.

```
FIRMAR (autor)                                VERIFICAR (cualquiera)
documento -> hash SHA-256 -> h               documento recibido -> hash -> h'
firma = operar(h, llave PRIVADA del autor)   abrir(firma, llave PÚBLICA del autor) -> h
                                             ¿h == h'?  sí -> auténtico e íntegro
```

Da integridad, autenticidad y no repudio. No da confidencialidad: el documento firmado se lee igual. Se usa en certificados, actualizaciones de software, commits de Git firmados, DNSSEC y S/MIME.

> [!TIP]
> Regla para no confundirse: cifrar para alguien → su llave pública; firmar como tú → tu llave privada. La llave privada solo sale en dos verbos: descifrar y firmar.

```
Simétrica → una sola llave; rápida; problema: repartirla (AES, ChaCha20).
Asimétrica → par pública/privada; lenta; resuelve el reparto (RSA, ECC).
Cifrado híbrido → asimétrica para la llave, simétrica para los datos.
AES → bloque de 128 bits; llaves de 128/192/256.
AEAD (GCM, Poly1305) → cifra y detecta alteraciones.
ECB → modo inseguro; bloques iguales se ven iguales.
RSA → factorizar n = p·q; mínimo 2048 bits.
ECC → curvas elípticas; 256 bits ≈ RSA 3072.
Firma digital → hash firmado con la privada, verificado con la pública; no cifra.
Cifrar → llave pública del destinatario. Firmar → llave privada del autor.
```

### PKI

La **PKI** (Public Key Infrastructure) es el conjunto de autoridades, políticas, formatos y procesos que emiten, distribuyen, validan y revocan certificados digitales para que una llave pública pueda asociarse con confianza a una identidad.

El problema que resuelve es el hueco que dejó la sección anterior: si descargo "la llave pública de mi banco", ¿cómo sé que es del banco y no de un atacante que la cambió por la suya? Alguien de confianza tiene que dar fe. Esa es la CA.

Analogía: el registro civil y la cédula. No conoces al desconocido, pero confías en su cédula porque la emitió el registro, y la cédula tiene marcas que solo el registro puede poner. Si la cédula fue reportada como robada, aparece en una lista.

Un certificado X.509 v3 (RFC 5280) contiene, entre otros:

- Subject: a quién pertenece. SAN (Subject Alternative Name): los dominios exactos para los que vale (`banco.com`, `www.banco.com`).
- Issuer: qué CA lo firmó.
- Validez: `Not Before` y `Not After`.
- La llave pública del dueño.
- Usos permitidos (servidor TLS, cliente, firma de código, correo).
- La firma de la CA sobre todo lo anterior.

### Cadena de confianza

La cadena de confianza es la secuencia de certificados en la que cada uno está firmado por el siguiente, desde el certificado del sitio hasta una CA raíz que el sistema ya considera confiable.

```
  [ CA raíz ]  autofirmada, viene preinstalada en el sistema/navegador (trust store),
      |        guardada offline en un HSM; vive 20-25 años
      | firma
  [ CA intermedia ]  la que firma día a día; si se compromete, se revoca sin tocar la raíz
      | firma
  [ Certificado hoja ]  www.banco.com; vida corta
```

El servidor envía la hoja y la intermedia. El navegador verifica cada firma con la llave pública del nivel superior, sube hasta una raíz que ya tiene guardada, y además comprueba que el dominio coincida con el SAN, que la fecha esté dentro de la validez y que nada esté revocado. Si un eslabón falla, aparece la pantalla de advertencia.

```
$ openssl s_client -connect ejemplo.com:443 -servername ejemplo.com </dev/null 2>/dev/null | head -6
CONNECTED(00000003)
---
Certificate chain
 0 s:CN = ejemplo.com
   i:C = US, O = Let's Encrypt, CN = R11
 1 s:C = US, O = Let's Encrypt, CN = R11
   i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
```

`s:` es el sujeto y `i:` el emisor: la hoja la emitió R11, R11 la emitió ISRG Root X1, que está en el trust store.

Vida de los certificados: el máximo permitido por el CA/Browser Forum era 398 días, y la votación SC-081 lo reduce por etapas desde marzo de 2026 hasta 47 días en marzo de 2029. Por eso la emisión se automatiza con ACME (el protocolo de Let's Encrypt).

### Revocación: CRL y OCSP

La revocación es el mecanismo de PKI que anula un certificado antes de su vencimiento, por ejemplo porque se filtró su llave privada.

- CRL (Certificate Revocation List): lista firmada por la CA con los números de serie revocados; el cliente la descarga periódicamente. Simple, pero puede pesar megas y llegar tarde.
- OCSP (Online Certificate Status Protocol, RFC 6960): el cliente pregunta en línea "¿el serial 0x4A3F sigue vigente?". Más fresco, pero le dice a la CA qué sitios visitas y, si la CA no responde, los navegadores suelen dejar pasar (soft-fail).
- OCSP stapling: el propio servidor consulta OCSP y "grapa" la respuesta firmada al handshake, ahorrándole la consulta al cliente.

La tendencia es volver a CRLs con certificados de vida corta: Let's Encrypt apagó sus respondedores OCSP en agosto de 2025 por privacidad y coste.

Transparencia de certificados (Certificate Transparency): las CA publican cada certificado en registros públicos de solo-agregar, así un dueño puede detectar un certificado emitido para su dominio sin que lo pidiera.

> [!WARNING]
> El candado del navegador solo dice que el canal está cifrado con quien tiene un certificado válido para ese dominio; no dice que el sitio sea honesto. Un sitio de phishing en `banco-seguro-login.com` tiene su candado igual.

```
PKI → sistema que emite, valida y revoca certificados.
CA → autoridad que firma certificados y da fe de la identidad.
Certificado X.509 → llave pública + identidad + validez + firma de la CA.
SAN → los dominios exactos para los que vale el certificado.
CA raíz → autofirmada, en el trust store, offline.
CA intermedia → firma en el día a día; protege a la raíz.
Cadena de confianza → hoja → intermedia → raíz, verificando cada firma.
CRL → lista de seriales revocados que se descarga.
OCSP → consulta en línea del estado de un certificado.
OCSP stapling → el servidor adjunta la respuesta OCSP al handshake.
Certificate Transparency → registro público de todos los certificados emitidos.
```

### Obfuscation

La **ofuscación** (obfuscation) es una técnica que transforma código o datos para que sean difíciles de entender por una persona, sin usar un secreto y sin impedir que el programa siga funcionando.

Existe para frenar el análisis, no para impedirlo: proteger la propiedad intelectual de un JavaScript o una app Android (ProGuard renombra `calcularDescuento` a `a.b()`), y, del lado atacante, esconder malware de los antivirus y de los analistas (un PowerShell ofuscado con concatenaciones y reemplazos para que no aparezca la palabra `Invoke-Mimikatz`).

Analogía: escribir el diario con letra horrible y en abreviaturas propias. Quien tenga paciencia lo lee; no hay candado, solo molestia.

### Ofuscación vs cifrado vs codificación

Codificación, ofuscación y cifrado son tres tipos de transformación reversible de datos que se distinguen por si hace falta un secreto para revertirlas y por el propósito con que se aplican. Las tres transforman datos, pero con propósitos distintos, y confundirlas es un error grave de diseño:

- La **codificación** (encoding) es una transformación pública y reversible que cambia el formato de los datos para que puedan transportarse o guardarse, sin ninguna intención de ocultarlos. Base64, URL encoding (`%20`), hexadecimal, UTF-8.
- El **cifrado** (encryption) es una transformación reversible solo con una llave secreta, cuyo fin es la confidencialidad.
- El **hashing** es una transformación de un solo sentido para integridad (sección Hashing).
- La **ofuscación** es una transformación reversible sin llave, cuyo fin es dificultar la lectura humana.

```
                 ¿reversible?        ¿necesita secreto?   ¿para qué?
Codificación     sí, por cualquiera  no                   transportar/guardar formato
Ofuscación       sí, con esfuerzo    no                   dificultar el análisis
Cifrado          sí, con la llave    sí (la llave)        confidencialidad
Hashing          no                  no (salvo HMAC)      integridad / contraseñas
```

### Base64

Base64 es un esquema de codificación que representa bytes arbitrarios usando solo 64 caracteres imprimibles (`A-Z a-z 0-9 + /`), tomando grupos de 3 bytes (24 bits) y convirtiéndolos en 4 caracteres de 6 bits cada uno. Por eso crece un 33 % y por eso a veces termina en `=` o `==` (relleno cuando el último grupo no llega a 3 bytes). Existe porque muchos sistemas (correo, JSON, URLs) solo transportan texto, no binario.

```
$ printf 'hola' | base64
aG9sYQ==
$ printf 'aG9sYQ==' | base64 -d
hola
$ printf 'admin:secreto' | base64
YWRtaW46c2VjcmV0bw==
```

El último ejemplo es exactamente lo que viaja en la cabecera HTTP Basic Auth (`Authorization: Basic YWRtaW46c2VjcmV0bw==`). Cualquiera que vea esa cabecera tiene la contraseña: base64 no protege nada. Lo mismo pasa con los JWT: su contenido está en base64url y se lee sin llave; lo que los protege contra cambios es la firma, no la codificación.

> [!IMPORTANT]
> Codificar y ofuscar no protegen nada; solo el cifrado con una llave secreta da confidencialidad. Si para revertir algo no hace falta un secreto, cualquiera lo revierte.

```
Codificación → cambia el formato; reversible por cualquiera (base64, hex, URL).
Ofuscación → dificulta la lectura humana; reversible con esfuerzo; sin llave.
Cifrado → oculta el contenido; reversible solo con la llave.
Hashing → huella de un solo sentido; no se revierte.
Base64 → 3 bytes → 4 caracteres; +33 %; "=" de relleno; no es seguridad.
```

## Secure vs Unsecure Protocols

### FTP vs SFTP

FTP (File Transfer Protocol) es un protocolo de transferencia de archivos de 1971 (versión actual en RFC 959) que envía usuario, contraseña, comandos y datos en texto claro; SFTP (SSH File Transfer Protocol) es un protocolo de transferencia de archivos distinto que funciona como subsistema dentro de una conexión SSH cifrada.

FTP usa dos conexiones: control en TCP 21 y datos en TCP 20 (modo activo) o en un puerto alto negociado (modo pasivo). Eso complica los firewalls y todo viaja legible. SFTP usa una sola conexión TCP 22, con la autenticación de SSH (contraseña o llave) y todo cifrado.

Analogía: FTP es mandar documentos en una caja abierta por mensajería; SFTP es mandarlos dentro de un camión blindado que además ya tiene cerrojo.

Hay tres variantes que se confunden:

- SFTP: protocolo nuevo sobre SSH (TCP 22). No es FTP.
- FTPS: el FTP de siempre con TLS. Explícito (comando `AUTH TLS` en el puerto 21, RFC 4217) o implícito (TLS desde el inicio en TCP 990).
- SCP: copia de archivos sobre SSH, más simple; OpenSSH ya lo implementa internamente con SFTP.

```
$ sftp ana@servidor.lab
Connected to servidor.lab.
sftp> put informe.pdf
Uploading informe.pdf to /home/ana/informe.pdf
informe.pdf                       100%  2048KB  10.2MB/s   00:00
```

```
FTP → TCP 21 control + 20/pasivo datos; todo en claro.
SFTP → subsistema de SSH; TCP 22; todo cifrado; no es FTP.
FTPS → FTP + TLS; explícito en 21 (AUTH TLS) o implícito en 990.
SCP → copia simple sobre SSH.
```

### Un protocolo seguro es la versión insegura metida en un túnel cifrado y autenticado

> [!IMPORTANT]
> Casi todos los protocolos "seguros" de esta parte son el mismo protocolo inseguro de siempre, con la misma semántica, envuelto en una capa que añade cifrado, integridad y autenticación del servidor (TLS, SSH o IPsec), o firmas sobre los datos (DNSSEC, S/MIME).

Un protocolo seguro es un protocolo de comunicación que garantiza confidencialidad, integridad y autenticidad de lo que transporta, normalmente reutilizando un protocolo existente y añadiéndole una capa criptográfica. Saber qué capa añade cada uno, y en qué puerto, resuelve la mitad de las preguntas de examen.

```
Inseguro     Puerto   ->  Seguro              Puerto      Capa que añade
FTP          21       ->  SFTP / FTPS         22 / 990    SSH / TLS
HTTP         80       ->  HTTPS               443         TLS
LDAP         389      ->  LDAPS               636         TLS
RTP          dinám.   ->  SRTP                dinám.      cifrado + HMAC propio
SMTP correo  25       ->  S/MIME              (contenido) firma y cifrado del mensaje
DNS          53       ->  DNSSEC              53          firmas (sin cifrado)
IP           -        ->  IPsec               50/51, UDP 500/4500   AH / ESP
Telnet       23       ->  SSH                 22          SSH
```

La última columna es la idea entera: el protocolo de abajo no cambia, cambia la envoltura.

Ejemplo: si capturas tráfico en tu laboratorio, en FTP ves `USER ana` y `PASS Secreto123` en claro; en SFTP solo ves paquetes SSH cifrados al puerto 22. Lo que se transfiere es el mismo archivo.

Límite: DNSSEC es la excepción a "cifrado": firma pero no oculta. Y un protocolo seguro mal configurado (TLS 1.0, certificados sin validar) no es seguro.

```
Protocolo seguro → mismo protocolo + capa que da cifrado, integridad y autenticación.
Excepción DNSSEC → da integridad y autenticidad, NO confidencialidad.
```

### SSL vs TLS

SSL (Secure Sockets Layer) es el protocolo de cifrado de canal creado por Netscape en los 90, hoy completamente obsoleto e inseguro; TLS (Transport Layer Security) es su sucesor estandarizado por el IETF, que protege con cifrado, integridad y autenticación del servidor una conexión entre dos aplicaciones.

Cuando la gente dice "certificado SSL" se refiere a un certificado X.509 que hoy se usa con TLS: el nombre se quedó por costumbre.

Analogía: es como seguir diciendo "marcar" un número aunque los teléfonos ya no tengan disco.

Línea de tiempo, que hay que saber:

- SSL 2.0 (1995) y SSL 3.0 (1996): prohibidos (RFC 6176 y RFC 7568; SSL 3.0 cayó con POODLE en 2014).
- TLS 1.0 (1999) y TLS 1.1 (2006): obsoletos desde RFC 8996 (2021).
- TLS 1.2 (2008): aceptable con suites modernas (ECDHE + AES-GCM o ChaCha20-Poly1305).
- TLS 1.3 (2018, RFC 8446): el recomendado.

Qué cambió en TLS 1.3:

- Handshake de 1 ida y vuelta (1-RTT) en vez de 2; con reanudación, 0-RTT.
- Eliminó RSA como intercambio de llaves, RC4, 3DES, CBC, SHA-1 y la compresión: solo quedan 5 suites, todas AEAD.
- Forward secrecy obligatorio (solo DHE/ECDHE).
- El certificado del servidor ya va cifrado.

Handshake TLS 1.3 simplificado:

```
Cliente                                           Servidor
  |-- ClientHello: versiones, suites, key_share (valor ECDHE del cliente) -->|
  |<-- ServerHello: suite elegida, key_share (valor ECDHE del servidor) -----|
  |    [a partir de aquí todo va cifrado con llaves derivadas del ECDHE]     |
  |<-- Certificate (cadena X.509) -------------------------------------------|
  |<-- CertificateVerify (firma del handshake con la llave privada) ---------|
  |<-- Finished --------------------------------------------------------------|
  |-- Finished -------------------------------------------------------------->|
  |== datos de aplicación cifrados (HTTP, SMTP, LDAP...) ====================|
```

Las tres piezas de las secciones anteriores aparecen juntas: ECDHE da la llave (Key Exchange), el certificado y la firma prueban que el servidor es quien dice (PKI y firmas), y AES-GCM cifra los datos (simétrica).

```
$ openssl s_client -connect ejemplo.com:443 -tls1_3 </dev/null 2>/dev/null | grep -E 'Protocol|Cipher'
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384

$ openssl s_client -connect ejemplo.com:443 -tls1_1 </dev/null
...alert protocol version...            <- bien: el servidor rechaza TLS 1.1
```

```
SSL 2.0/3.0 → obsoletos y prohibidos (POODLE).
TLS 1.0/1.1 → obsoletos desde RFC 8996 (2021).
TLS 1.2 → aceptable con ECDHE + AEAD.
TLS 1.3 → 1-RTT, solo AEAD, forward secrecy obligatorio, certificado cifrado.
"Certificado SSL" → nombre histórico de un certificado X.509 usado con TLS.
```

### IPSEC

IPsec (Internet Protocol Security, RFC 4301) es un conjunto de protocolos que protege el tráfico en la capa de red (capa 3), autenticando y opcionalmente cifrando cada paquete IP, de modo que cualquier aplicación queda protegida sin modificarse.

A diferencia de TLS, que protege una conexión de una aplicación, IPsec protege todos los paquetes entre dos equipos o dos redes. Es la base de las VPN sitio a sitio (dos oficinas unidas por internet) y de muchas VPN de acceso remoto.

Analogía: TLS es meter cada carta en un sobre cerrado; IPsec es meter todo el camión de correo en un contenedor sellado, cartas incluidas, sin importar qué hay dentro.

Las piezas:

- IKE (Internet Key Exchange, hoy IKEv2): negocia los algoritmos, autentica a los dos extremos (llave precompartida o certificados) y hace un Diffie-Hellman para acordar llaves. Usa UDP 500, y UDP 4500 cuando hay NAT de por medio (NAT-T).
- SA (Security Association): el acuerdo resultante, en un solo sentido (hace falta una por dirección).
- AH (Authentication Header, protocolo IP 51, RFC 4302): da integridad y autenticación del paquete completo, incluida la cabecera IP. **No cifra**. Como protege la cabecera IP, choca con NAT, que la reescribe.
- ESP (Encapsulating Security Payload, protocolo IP 50, RFC 4303): cifra y autentica la carga útil. Es lo que se usa casi siempre.

Modos:

```
Paquete original:      [ IP orig | TCP | datos ]

Modo transporte (ESP): [ IP orig | ESP | TCP | datos | ESP trailer | ICV ]
                                        \_____ cifrado _____/
  -> se cifra la carga; la cabecera IP original queda visible.
  -> uso: host a host (dos servidores que hablan directo).

Modo túnel (ESP):      [ IP nueva | ESP | IP orig | TCP | datos | trailer | ICV ]
                                         \________ cifrado __________/
  -> se cifra el paquete entero y se envuelve en uno nuevo.
  -> uso: gateway a gateway (VPN sitio a sitio); oculta las IPs internas.
```

Ejemplo: la oficina de San José (10.1.0.0/16) y la de Liberia (10.2.0.0/16) se unen con dos firewalls. Un PC de San José envía a 10.2.0.50; su firewall mete el paquete entero dentro de un ESP en modo túnel dirigido a la IP pública del firewall de Liberia. Quien mira internet solo ve tráfico ESP entre dos IPs públicas, sin saber qué equipos internos hablan.

```
$ sudo swanctl --list-sas
oficinas: #1, ESTABLISHED, IKEv2
  local  '200.0.0.1' @ 200.0.0.1[500]
  remote '201.0.0.1' @ 201.0.0.1[500]
  AES_CBC-256/HMAC_SHA2_256_128/PRF_HMAC_SHA2_256/ECP_256
  sanjose-liberia: #1, INSTALLED, TUNNEL, ESP:AES_GCM_16-256
    local  10.1.0.0/16
    remote 10.2.0.0/16
```

```
IPsec → protege paquetes IP en capa 3; base de VPN.
IKEv2 → negocia y autentica; hace DH; UDP 500 (4500 con NAT-T).
SA → acuerdo de seguridad unidireccional.
AH → protocolo 51; integridad + autenticación; NO cifra; choca con NAT.
ESP → protocolo 50; cifra + autentica la carga; el que se usa.
Modo transporte → cifra la carga; host a host.
Modo túnel → cifra el paquete entero y lo envuelve; sitio a sitio.
```

### DNSSEC

DNSSEC (DNS Security Extensions, RFC 4033) es un conjunto de extensiones de DNS que añade firmas digitales a las respuestas, para que un resolvedor pueda comprobar que un registro viene del dueño legítimo de la zona y no fue alterado.

DNS clásico (UDP 53) no tiene ninguna verificación: un atacante que logre inyectar una respuesta falsa antes que la verdadera (envenenamiento de caché, ataque Kaminsky de 2008) puede hacer que `banco.com` resuelva a su IP para miles de usuarios. DNSSEC resuelve la autenticidad e integridad de la respuesta.

Analogía: un notario que sella cada página de la guía telefónica. Cualquiera puede leer la guía (no es secreta), pero una página con un número cambiado no tendría el sello.

Registros nuevos:

- RRSIG: la firma de un conjunto de registros.
- DNSKEY: las llaves públicas de la zona. Se suelen usar dos: ZSK (Zone Signing Key) firma los registros, KSK (Key Signing Key) firma las DNSKEY.
- DS (Delegation Signer): hash de la KSK de la zona hija, publicado y firmado en la zona padre. Es el eslabón de la cadena.
- NSEC / NSEC3: prueba firmada de que un nombre **no** existe.

Cadena de confianza (como la PKI, pero en el árbol DNS):

```
 raíz "."  (su KSK es el ancla de confianza, preinstalada en el resolvedor)
   |  DS de .com firmado por la raíz
 .com
   |  DS de ejemplo.com firmado por .com
 ejemplo.com
   |  DNSKEY de ejemplo.com  -> verifica ->  RRSIG del registro A
 www.ejemplo.com  A  93.184.215.14
```

```
$ dig +dnssec www.ejemplo.com A
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2
www.ejemplo.com.  300  IN  A      93.184.215.14
www.ejemplo.com.  300  IN  RRSIG  A 13 3 300 20261101000000 20261001000000 12345 ejemplo.com. kQ3x...
```

El flag `ad` (Authenticated Data) indica que el resolvedor validó la cadena. El `13` es el algoritmo (ECDSA P-256 con SHA-256).

> [!WARNING]
> DNSSEC no cifra: cualquiera en la red sigue viendo qué dominios consultas. La privacidad de las consultas la dan DoT (DNS over TLS, TCP 853) y DoH (DNS over HTTPS, 443); son complementarios, no sustitutos.

```
DNSSEC → firma respuestas DNS; integridad y autenticidad, sin cifrado.
RRSIG → firma de un conjunto de registros.
DNSKEY → llaves públicas de la zona (ZSK firma registros, KSK firma llaves).
DS → hash de la KSK hija publicado en la zona padre; une la cadena.
NSEC/NSEC3 → prueba firmada de que un nombre no existe.
Flag ad → el resolvedor validó la cadena DNSSEC.
DoT / DoH → cifran las consultas DNS (853 / 443); DNSSEC no.
```

### LDAPS

LDAPS (LDAP over SSL/TLS) es la variante de LDAP que establece una sesión TLS antes de intercambiar cualquier mensaje LDAP, de modo que las credenciales y los datos del directorio viajan cifrados.

El problema: el bind simple de LDAP envía DN y contraseña en claro por TCP 389 (detalle de LDAP y bind en [`08-autenticacion`](../08-autenticacion/)). Cualquier equipo comprometido en la misma red puede capturar las contraseñas de cada aplicación que valida usuarios contra el directorio.

Analogía: es la misma ventanilla de consultas, pero detrás de un vidrio blindado con intercomunicador.

Dos formas de proteger LDAP:

- LDAPS: TLS desde el primer byte, en TCP 636 (y 3269 para el Global Catalog de AD).
- StartTLS: se conecta en claro al 389 y luego envía la operación extendida StartTLS para pasar a TLS. Es el estándar del IETF (RFC 4513). Riesgo: si el cliente no exige que la mejora ocurra, un MITM puede quitar el StartTLS (ataque de downgrade) y el cliente sigue en claro.

El servidor necesita un certificado de una CA en la que confíen los clientes; el error clásico es que la aplicación desactive la validación del certificado "para que funcione", lo que deja la puerta abierta a un MITM.

```
$ openssl s_client -connect dc01.empresa.local:636 </dev/null 2>/dev/null | grep -E 'subject=|Verify return'
subject=CN = dc01.empresa.local
Verify return code: 0 (ok)

$ ldapsearch -H ldaps://dc01.empresa.local -D "cn=svc-app,ou=Servicios,dc=empresa,dc=local" -W -b "dc=empresa,dc=local" "(uid=ana)" cn
```

Microsoft endureció los controladores de dominio para exigir firma de LDAP y channel binding, de modo que los binds simples sin TLS se rechacen.

```
LDAP → TCP 389; bind simple en claro.
LDAPS → TLS desde el inicio; TCP 636 (3269 Global Catalog).
StartTLS → empieza en 389 y sube a TLS; vulnerable a downgrade si no se exige.
```

### SRTP

SRTP (Secure Real-time Transport Protocol, RFC 3711) es un perfil de RTP que cifra y autentica los paquetes de audio y video en tiempo real con una sobrecarga mínima, para que una llamada de voz o videoconferencia no pueda escucharse ni alterarse.

RTP es el protocolo que transporta voz y video en VoIP y WebRTC sobre UDP, en puertos dinámicos. Sin protección, cualquiera en la ruta captura los paquetes y los reproduce como audio (Wireshark tiene un reproductor de llamadas RTP). La voz no tolera retrasos, así que TLS (sobre TCP, con retransmisiones) no sirve; SRTP cifra paquete por paquete sin esperar.

Analogía: hablar por walkie-talkie con un codificador de voz acordado; quien sintonice la frecuencia solo oye ruido.

Cómo funciona:

- Cifrado: AES en modo contador (AES-CM, 128 bits) o AES-GCM; la cabecera RTP queda en claro (el enrutamiento la necesita) y la carga va cifrada.
- Integridad: etiqueta HMAC-SHA1 de 80 bits por paquete, que también evita replays.
- SRTCP: lo mismo para RTCP, el canal de control y estadísticas.
- SRTP no negocia llaves. Las llaves llegan por otro lado: SDES (en el mensaje SIP; solo seguro si SIP va sobre TLS, puerto 5061), DTLS-SRTP (un handshake DTLS sobre el mismo UDP, obligatorio en WebRTC) o ZRTP.

Ejemplo: una centralita con 100 teléfonos IP. Con RTP, un atacante en la VLAN de voz graba todas las llamadas. Con SIP sobre TLS (5061) + SRTP, captura paquetes UDP con contenido ilegible. Cada videollamada de un navegador (Meet, Teams web) ya usa DTLS-SRTP por obligación.

```
RTP → transporte de voz/video sobre UDP; en claro.
SRTP → RTP cifrado (AES) y autenticado (HMAC 80 bits) por paquete.
SRTCP → versión segura del canal de control RTCP.
DTLS-SRTP → negocia llaves de SRTP; obligatorio en WebRTC.
SIP sobre TLS → señalización protegida; puerto 5061.
```

### S/MIME

S/MIME (Secure/Multipurpose Internet Mail Extensions, versión 4.0 en RFC 8551) es un estándar de correo electrónico que usa certificados X.509 para firmar y cifrar el contenido de cada mensaje de extremo a extremo.

El correo nació sin seguridad: SMTP no autentica al remitente del contenido y los mensajes quedan guardados en servidores intermedios. STARTTLS entre servidores cifra el transporte salto por salto, pero el mensaje se guarda legible en cada servidor. S/MIME protege el mensaje en sí: solo el destinatario lo lee y cualquiera puede verificar quién lo firmó.

Analogía: STARTTLS es que el cartero viaje en un camión blindado entre oficinas de correo (pero en cada oficina abren la saca); S/MIME es una carta en sobre lacrado con tu sello personal, que solo el destinatario puede abrir.

Cómo funciona:

```
FIRMAR (Ana)
  hash del mensaje -> firmado con llave PRIVADA de Ana -> se adjunta + certificado de Ana
  Luis verifica con la llave PÚBLICA del certificado de Ana

CIFRAR (Ana -> Luis)
  llave AES aleatoria cifra el mensaje
  esa llave AES se cifra con la llave PÚBLICA de Luis (de su certificado)
  Luis la abre con su llave PRIVADA y descifra el mensaje
```

Es cifrado híbrido otra vez. Para cifrar a alguien necesitas su certificado antes; por eso el primer correo suele ser solo firmado.

S/MIME frente a PGP: los dos firman y cifran mensajes, pero S/MIME confía en CAs jerárquicas (una PKI, encaja en empresas con Outlook) y PGP en una red de confianza entre usuarios. Ninguno cifra las cabeceras (asunto, remitente, destinatario), que siguen visibles.

```
$ openssl smime -sign -in mensaje.txt -signer ana.crt -inkey ana.key -out firmado.eml
$ openssl smime -verify -in firmado.eml -CAfile ca-empresa.crt
Verification successful
```

```
S/MIME → firma y cifra cada correo con certificados X.509; extremo a extremo.
STARTTLS (correo) → cifra el transporte salto a salto; el mensaje queda legible en servidores.
PGP → firma y cifra correos con red de confianza en vez de CAs.
Cabeceras → ni S/MIME ni PGP cifran asunto ni direcciones.
```

## Recursos para aprender y practicar

### Videos

- [Hashing Algorithms and Security - Computerphile](https://www.youtube.com/watch?v=b4b8ktEV4Bg) — Computerphile (Tom Scott); qué es un hash y por qué los algoritmos se vuelven obsoletos (nodo Hashing).
- [SHA: Secure Hashing Algorithm - Computerphile](https://www.youtube.com/watch?v=DMtFhACPnTY) — Computerphile; cómo funciona SHA por dentro (nodo Hashing).
- [Password Hashing, Salts, Peppers | Explained!](https://www.youtube.com/watch?v=--tnZMuoK3E) — Seytonic; sal, pimienta y hashes de contraseñas (nodo Salting).
- [Password Cracking - Computerphile](https://www.youtube.com/watch?v=7U-RbOKanYs) — Computerphile; cracking con hashcat y por qué importa el hash lento (nodos Hashing y Salting).
- [Secret Key Exchange (Diffie-Hellman) - Computerphile](https://www.youtube.com/watch?v=NmM9HA2MQGI) y [Diffie Hellman -the Mathematics bit-](https://www.youtube.com/watch?v=Yjrfm_oRO0w) — Computerphile; la idea y la matemática de DH (nodo Key Exchange).
- [Public Key Cryptography - Computerphile](https://www.youtube.com/watch?v=GSIDS_lvRv4) — Computerphile; llave pública vs privada (nodo Private vs Public Keys).
- [What are Digital Signatures? - Computerphile](https://www.youtube.com/watch?v=s22eJ1eVLTU) — Computerphile; firmas digitales (nodo Private vs Public Keys).
- [Elliptic Curves - Computerphile](https://www.youtube.com/watch?v=NF1pwjL9-DE) — Computerphile; intuición de ECC (nodo Private vs Public Keys).
- [Public Key Infrastructure - CompTIA Security+ SY0-701 - 1.4](https://www.youtube.com/watch?v=xHAMEF7-inQ) y [Certificates - CompTIA Security+ SY0-701 - 1.4](https://www.youtube.com/watch?v=cLa94BZH_9s) — Professor Messer; CA, cadena de confianza, CRL y OCSP (nodo PKI).
- [Hashing and Digital Signatures - CompTIA Security+ SY0-701 - 1.4](https://www.youtube.com/watch?v=EcGmQjl6XEo) — Professor Messer; repaso de examen de hashes y firmas (nodos Hashing y Private vs Public Keys).
- [Transport Layer Security (TLS) - Computerphile](https://www.youtube.com/watch?v=0TLDTodL7Lc) y [SSL, TLS, HTTP, HTTPS Explained](https://www.youtube.com/watch?v=hExRDVZHhig) — Computerphile y PowerCert; el handshake y la evolución SSL → TLS (nodo SSL vs TLS).
- [IPsec Explained](https://www.youtube.com/watch?v=xTH1ZA_qUvA) — PowerCert Animated Videos; AH, ESP, túnel y transporte (nodo IPSEC).
- [What is DNSSEC (Domain Name System Security Extensions)?](https://www.youtube.com/watch?v=Fk2oejzgSVQ) — IBM Technology; firmas en DNS y cadena de confianza (nodo DNSSEC).
- [Secure Communication - CompTIA Security+ SY0-701 - 3.2](https://www.youtube.com/watch?v=uU3e_ntg-3g) — Professor Messer; VPN, IPsec y protocolos seguros (parte Secure vs Unsecure Protocols).

### Lectura y documentación

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — parámetros recomendados de Argon2id, bcrypt, scrypt y PBKDF2, sal y pimienta.
- [FIPS 180-4: Secure Hash Standard](https://csrc.nist.gov/pubs/fips/180-4/upd1/final) y [RFC 9106: Argon2](https://www.rfc-editor.org/rfc/rfc9106) — las especificaciones de SHA-2 y Argon2.
- [RFC 2631: Diffie-Hellman Key Agreement Method](https://www.rfc-editor.org/rfc/rfc2631) — el método DH formalizado.
- [NIST SP 800-57 Part 1 Rev. 5: Recommendation for Key Management](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) — equivalencias de tamaños de llave y vida útil de algoritmos.
- [RFC 5280: X.509 PKI Certificate and CRL Profile](https://www.rfc-editor.org/rfc/rfc5280) y [RFC 6960: OCSP](https://www.rfc-editor.org/rfc/rfc6960) — formato de certificados, CRL y OCSP.
- [How It Works (Let's Encrypt)](https://letsencrypt.org/how-it-works/) — ACME y emisión automática de certificados.
- [RFC 4648: Base16, Base32, Base64](https://www.rfc-editor.org/rfc/rfc4648) — la codificación base64 exacta.
- [RFC 4253: SSH Transport Layer](https://www.rfc-editor.org/rfc/rfc4253) y [RFC 4217: Securing FTP with TLS](https://www.rfc-editor.org/rfc/rfc4217) — la base de SFTP y FTPS.
- [RFC 8446: TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446), [RFC 7568: Deprecating SSLv3](https://www.rfc-editor.org/rfc/rfc7568) y [RFC 8996: Deprecating TLS 1.0 and 1.1](https://www.rfc-editor.org/rfc/rfc8996) — TLS actual y por qué se retiraron las versiones viejas.
- [OWASP Transport Layer Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html) — configuración segura de TLS.
- [RFC 4301: Security Architecture for IP](https://www.rfc-editor.org/rfc/rfc4301), [RFC 4302: AH](https://www.rfc-editor.org/rfc/rfc4302) y [RFC 4303: ESP](https://www.rfc-editor.org/rfc/rfc4303) — IPsec completo.
- [RFC 4033: DNS Security Introduction and Requirements](https://www.rfc-editor.org/rfc/rfc4033) — DNSSEC.
- [RFC 4513: LDAP Authentication Methods and Security Mechanisms](https://www.rfc-editor.org/rfc/rfc4513) — StartTLS y binds seguros.
- [RFC 3711: SRTP](https://www.rfc-editor.org/rfc/rfc3711) y [RFC 8551: S/MIME 4.0](https://www.rfc-editor.org/rfc/rfc8551) — voz segura y correo firmado/cifrado.

### Práctica

- [TryHackMe: Cryptography Basics](https://tryhackme.com/room/cryptographybasics), [Cryptography Concepts](https://tryhackme.com/room/cryptographyconcepts) y [Encryption - Crypto 101](https://tryhackme.com/room/encryptioncrypto101) — gratis; simétrica, asimétrica, intercambio de claves y PKI.
- [TryHackMe: Crack the hash](https://tryhackme.com/room/crackthehash) y [Breaking Crypto the Simple Way](https://tryhackme.com/room/breakingcryptothesimpleway) — gratis; identificar y crackear hashes en laboratorio (nodo Hashing). Retos de cripto por niveles en [picoGym](https://picoctf.org/) (categoría Cryptography).
- [cryptopals](https://cryptopals.com/) — retos de programación de criptografía; el [set 1](https://cryptopals.com/sets/1) arranca con hex, base64 y XOR y avanza hasta romper AES-ECB (nodos Obfuscation y Private vs Public Keys).
- [CryptoHack](https://cryptohack.org/) — gratis; retos de criptografía por niveles, desde codificaciones y XOR hasta RSA y Diffie-Hellman mal implementados (nodos Key Exchange y Private vs Public Keys).
- [picoCTF](https://picoctf.org/) — en picoGym, categoría Cryptography: retos de cifrados clásicos, codificaciones y RSA para resolver (nodos Obfuscation y Hashing).
- [Ejercicio 1: Haz Diffie-Hellman a mano y en Python](ejercicios.md#ejercicio-1-haz-diffie-hellman-a-mano-y-en-python) — reproduce el intercambio de la nota y repítelo con un primo de 2048 bits de `openssl dhparam`.
- [Ejercicio 2: Crea una PKI con openssl y revoca un certificado](ejercicios.md#ejercicio-2-crea-una-pki-con-openssl-y-revoca-un-certificado) — monta raíz, intermedia y hoja, verifica la cadena, revoca y genera la CRL.
- [Ejercicio 3: Compara FTP y SFTP en Wireshark](ejercicios.md#ejercicio-3-compara-ftp-y-sftp-en-wireshark) — sube un archivo por cada uno entre dos VMs y ve `USER`/`PASS` en claro solo en FTP.
- [Ejercicio 4: Levanta un túnel IPsec y observa ESP en Wireshark](ejercicios.md#ejercicio-4-levanta-un-túnel-ipsec-y-observa-esp-en-wireshark) — une dos VMs con strongSwan y compara el ping en claro con el tráfico ESP del túnel.
- [Ejercicio 5: Mide el coste de MD5 frente a bcrypt](ejercicios.md#ejercicio-5-mide-el-coste-de-md5-frente-a-bcrypt) — genera tus hashes y mide con hashcat cuántos por segundo prueba cada algoritmo y por qué bcrypt protege mejor.

## Cuadro resumen

Todo lo visto, en una línea por término.

Basics of Cryptography

```
Hash → huella de tamaño fijo; de un solo sentido.
Preimagen → no se puede ir del hash a la entrada.
Colisión → dos entradas con el mismo hash; un hash de n bits da n/2 bits contra colisiones.
Avalancha → 1 bit distinto en la entrada cambia ~50 % de la salida.
MD5 → 128 bits; roto; solo checksum accidental.
SHA-1 → 160 bits; colisión práctica (SHAttered 2017); obsoleto.
SHA-256 → 256 bits; estándar actual; rápido, NO apto solo para contraseñas.
bcrypt → hash lento con factor de coste (mínimo 10); límite 72 bytes.
Argon2id → hash lento y costoso en memoria; recomendado para contraseñas.
Sal → aleatoria, única por usuario, pública; rompe tablas precalculadas y patrones.
Rainbow table → tabla precalculada hash → contraseña; inútil contra hashes con sal.
Pimienta → secreto común guardado fuera de la base de datos.
Key exchange → acordar una llave secreta por un canal público.
Diffie-Hellman → s = g^(ab) mod p; seguro por el logaritmo discreto.
ECDH → DH sobre curvas elípticas (X25519, P-256); llaves más cortas.
Efímero (DHE/ECDHE) → llaves nuevas por conexión; da forward secrecy.
Forward secrecy → robar la llave del servidor no descifra tráfico pasado.
DH sin autenticar → vulnerable a MITM; se arregla firmando con certificado.
Simétrica → una sola llave; rápida; problema: repartirla (AES, ChaCha20).
Asimétrica → par pública/privada; lenta; resuelve el reparto (RSA, ECC).
Cifrado híbrido → asimétrica para la llave, simétrica para los datos.
AES → bloque de 128 bits; llaves de 128/192/256.
AEAD (GCM, Poly1305) → cifra y detecta alteraciones.
ECB → modo inseguro; bloques iguales se ven iguales.
RSA → factorizar n = p·q; mínimo 2048 bits.
ECC → curvas elípticas; 256 bits ≈ RSA 3072.
Firma digital → hash firmado con la privada, verificado con la pública; no cifra.
Cifrar → llave pública del destinatario. Firmar → llave privada del autor.
PKI → sistema que emite, valida y revoca certificados.
CA → autoridad que firma certificados y da fe de la identidad.
Certificado X.509 → llave pública + identidad + validez + firma de la CA.
SAN → los dominios exactos para los que vale el certificado.
CA raíz → autofirmada, en el trust store, offline.
CA intermedia → firma en el día a día; protege a la raíz.
Cadena de confianza → hoja → intermedia → raíz, verificando cada firma.
CRL → lista de seriales revocados que se descarga.
OCSP → consulta en línea del estado de un certificado.
OCSP stapling → el servidor adjunta la respuesta OCSP al handshake.
Certificate Transparency → registro público de todos los certificados emitidos.
Codificación → cambia el formato; reversible por cualquiera (base64, hex, URL).
Ofuscación → dificulta la lectura humana; reversible con esfuerzo; sin llave.
Cifrado → oculta el contenido; reversible solo con la llave.
Hashing → huella de un solo sentido; no se revierte.
Base64 → 3 bytes → 4 caracteres; +33 %; "=" de relleno; no es seguridad.
```

Secure vs Unsecure Protocols

```
FTP → TCP 21 control + 20/pasivo datos; todo en claro.
SFTP → subsistema de SSH; TCP 22; todo cifrado; no es FTP.
FTPS → FTP + TLS; explícito en 21 (AUTH TLS) o implícito en 990.
SCP → copia simple sobre SSH.
Protocolo seguro → mismo protocolo + capa que da cifrado, integridad y autenticación.
Excepción DNSSEC → da integridad y autenticidad, NO confidencialidad.
SSL 2.0/3.0 → obsoletos y prohibidos (POODLE).
TLS 1.0/1.1 → obsoletos desde RFC 8996 (2021).
TLS 1.2 → aceptable con ECDHE + AEAD.
TLS 1.3 → 1-RTT, solo AEAD, forward secrecy obligatorio, certificado cifrado.
"Certificado SSL" → nombre histórico de un certificado X.509 usado con TLS.
IPsec → protege paquetes IP en capa 3; base de VPN.
IKEv2 → negocia y autentica; hace DH; UDP 500 (4500 con NAT-T).
SA → acuerdo de seguridad unidireccional.
AH → protocolo 51; integridad + autenticación; NO cifra; choca con NAT.
ESP → protocolo 50; cifra + autentica la carga; el que se usa.
Modo transporte → cifra la carga; host a host.
Modo túnel → cifra el paquete entero y lo envuelve; sitio a sitio.
DNSSEC → firma respuestas DNS; integridad y autenticidad, sin cifrado.
RRSIG → firma de un conjunto de registros.
DNSKEY → llaves públicas de la zona (ZSK firma registros, KSK firma llaves).
DS → hash de la KSK hija publicado en la zona padre; une la cadena.
NSEC/NSEC3 → prueba firmada de que un nombre no existe.
Flag ad → el resolvedor validó la cadena DNSSEC.
DoT / DoH → cifran las consultas DNS (853 / 443); DNSSEC no.
LDAP → TCP 389; bind simple en claro.
LDAPS → TLS desde el inicio; TCP 636 (3269 Global Catalog).
StartTLS → empieza en 389 y sube a TLS; vulnerable a downgrade si no se exige.
RTP → transporte de voz/video sobre UDP; en claro.
SRTP → RTP cifrado (AES) y autenticado (HMAC 80 bits) por paquete.
SRTCP → versión segura del canal de control RTCP.
DTLS-SRTP → negocia llaves de SRTP; obligatorio en WebRTC.
SIP sobre TLS → señalización protegida; puerto 5061.
S/MIME → firma y cifra cada correo con certificados X.509; extremo a extremo.
STARTTLS (correo) → cifra el transporte salto a salto; el mensaje queda legible en servidores.
PGP → firma y cifra correos con red de confianza en vez de CAs.
Cabeceras → ni S/MIME ni PGP cifran asunto ni direcciones.
```
