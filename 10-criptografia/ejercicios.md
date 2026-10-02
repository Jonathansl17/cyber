# Ejercicios: Criptografía

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo se hace sobre equipos, máquinas virtuales, claves y dominios tuyos. Cuando hay tráfico
que capturar o cifrar entre máquinas, se usan dos VMs en una red aislada (host-only o
interna), nunca contra servicios de terceros.

## Ejercicio 1: Haz Diffie-Hellman a mano y en Python

Nodo: Key Exchange ([README](README.md#diffie-hellman-con-números-pequeños)).

Objetivo: reproducir el intercambio Diffie-Hellman de la nota, primero a mano con los números
de juguete y luego en Python, comprobar que los dos lados llegan a la misma llave sin haberla
enviado, y repetirlo con un primo real de 2048 bits generado con openssl.

Necesitas: Python 3 y OpenSSL (ambos vienen en Linux). 20 minutos. No hace falta red.

### Pasos

1. Haz a mano el ejemplo de la nota: público `p = 23`, `g = 5`; Ana elige `a = 6`, Luis
   elige `b = 15`. Calcula `A = 5^6 mod 23` y `B = 5^15 mod 23`, intercambia A y B, y saca la
   llave por los dos lados. Verifícalo en Python.
   ```bash
   python3 -c 'p,g,a,b=23,5,6,15; A=pow(g,a,p); B=pow(g,b,p); print("A=",A,"B=",B,"sA=",pow(B,a,p),"sB=",pow(A,b,p))'
   ```
   - `python3` → el intérprete de Python 3.
   - `-c '...'` → ejecuta el programa que va entre comillas en vez de leerlo de un archivo.
   - `p,g,a,b=23,5,6,15` → asigna el primo público, el generador y los secretos de Ana y Luis.
   - `pow(g,a,p)` → calcula `g^a mod p` de forma eficiente (exponenciación modular); así salen `A` y `B`.
   - `pow(B,a,p)` y `pow(A,b,p)` → cada lado eleva el valor que recibió a su propio secreto: la llave compartida vista por Ana (`sA`) y por Luis (`sB`).
   - `print(...)` → muestra los valores en una línea.

   Salida esperada: `A= 8 B= 19 sA= 2 sB= 2`. Los dos lados obtienen `2` y nunca lo enviaron.

2. Comprueba que el espía lo tiene difícil incluso con números minúsculos: resuelve el
   logaritmo discreto a fuerza bruta para recuperar el secreto `a` a partir de `A`.
   ```bash
   python3 -c 'p,g,A=23,5,8; print([x for x in range(p) if pow(g,x,p)==A])'
   ```
   - `python3 -c` → (ver paso 1).
   - `p,g,A=23,5,8` → los valores públicos que ve el espía: primo, generador y la `A` que mandó Ana.
   - `[x for x in range(p) if pow(g,x,p)==A]` → prueba todos los exponentes de 0 a `p-1` y se queda con los que cumplen `g^x mod p = A`; es la fuerza bruta del logaritmo discreto.

   Da `[6]`: con `p=23` se prueba a mano; la gracia es que con `p` de 2048 bits esto es inviable.

3. Pasa a números reales: genera parámetros DH de 2048 bits con openssl. Tarda unos segundos.
   ```bash
   openssl dhparam -out dh2048.pem 2048
   openssl dhparam -in dh2048.pem -noout -text | head -3
   ```
   - `openssl dhparam` → subcomando de OpenSSL que genera o inspecciona parámetros Diffie-Hellman (`p` y `g`).
   - `-out dh2048.pem` → archivo donde se guardan los parámetros generados (formato PEM por defecto).
   - `2048` → tamaño en bits del primo `p` que se genera. Como no se indica `-2`, `-3` ni `-5`, el generador `g` es 2, el valor por defecto.
   - `-in dh2048.pem` → lee los parámetros de ese archivo en vez de generarlos.
   - `-noout` → no vuelve a escribir los parámetros codificados en PEM.
   - `-text` → imprime los parámetros en texto legible (tamaño, `prime`, `generator`).
   - `| head -3` → pasa la salida a `head`, que muestra solo las 3 primeras líneas (`-3` equivale a `-n 3`).

   Verás `DH Parameters: (2048 bit)` y un `prime` enorme. Ese es el `p` que haría imposible el
   paso 2.

4. Haz el intercambio con ese primo real en Python, con secretos grandes y aleatorios. Primero
   saca `p` y `g` del archivo en formato que Python entienda, y luego ejecuta el intercambio.
   ```bash
   # extrae p y g (ambos en hex) del dh2048.pem
   P=$(openssl asn1parse -in dh2048.pem | awk -F: '/INTEGER/{print $NF; exit}')
   G=$(openssl asn1parse -in dh2048.pem | awk -F': *' '/INTEGER/{c++; if(c==2){print $NF; exit}}')
   python3 - "$P" "$G" <<'EOF'
   import sys, secrets
   p = int(sys.argv[1], 16)
   g = int(sys.argv[2], 16)
   a = secrets.randbelow(p - 2) + 1
   b = secrets.randbelow(p - 2) + 1
   A, B = pow(g, a, p), pow(g, b, p)
   print("bits de p:", p.bit_length())
   print("misma llave:", pow(B, a, p) == pow(A, b, p))
   EOF
   ```
   Imprime `bits de p: 2048` y `misma llave: True`: la propiedad `(g^a)^b = (g^b)^a` se cumple
   igual con 2048 bits que con 23, pero ahora el logaritmo discreto es inviable.

### Resultado esperado

El intercambio de juguete reproducido a mano y en Python con llave compartida 2, la fuerza
bruta que recupera el secreto solo porque `p` es diminuto, y el mismo intercambio con un `p`
de 2048 bits donde esa fuerza bruta ya no es posible.

### Comprueba que lo lograste

- ¿Por qué los dos lados llegan al mismo número sin enviarlo? Porque `(g^a)^b mod p` es igual
  a `(g^b)^a mod p`; cada uno eleva lo que recibió a su propio secreto.
- ¿Qué protege el tamaño de `p`? La dificultad del logaritmo discreto: con 23 se resuelve a
  mano, con 2048 bits no hay cómputo que lo haga en tiempo útil.
- ¿DH autentica a los extremos? No; por sí solo es vulnerable a MITM, y por eso en TLS el
  servidor firma sus valores DH con su certificado.

### Limpieza

```bash
rm -f dh2048.pem
```

## Ejercicio 2: Crea una PKI con openssl y revoca un certificado

Nodo: PKI ([README](README.md#pki)).

Objetivo: montar la cadena de confianza de la nota (CA raíz, CA intermedia y certificado
hoja) con openssl, verificar la cadena con `openssl verify`, revocar la hoja y generar la
CRL, y comprobar que tras la revocación la verificación con `-crl_check` falla.

Necesitas: OpenSSL (viene en Linux; probado con OpenSSL 3.x). 40 minutos. No hace falta red.

### Pasos

1. Crea un directorio de trabajo y la estructura que la CA necesita para llevar su base de
   datos de certificados emitidos y revocados.
   ```bash
   mkdir -p ~/pki-lab/ca/newcerts && cd ~/pki-lab
   touch ca/index.txt
   echo 1000 > ca/serial
   echo 1000 > ca/crlnumber
   ```

2. Genera la CA raíz autofirmada (como la de la nota: vida larga, queda como ancla de confianza).
   ```bash
   openssl req -x509 -newkey rsa:2048 -sha256 -days 3650 -nodes \
     -keyout raiz.key -out raiz.crt -subj "/C=CR/O=Lab/CN=Lab Root CA"
   ```

3. Genera la CA intermedia (clave + CSR) y fírmala con la raíz, marcándola como CA.
   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout intermedia.key -out intermedia.csr \
     -subj "/C=CR/O=Lab/CN=Lab Intermediate CA"
   openssl x509 -req -in intermedia.csr -CA raiz.crt -CAkey raiz.key -CAcreateserial \
     -days 1825 -sha256 \
     -extfile <(printf "basicConstraints=critical,CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign\n") \
     -out intermedia.crt
   ```

4. Escribe el archivo de configuración de la CA intermedia, que `openssl ca` usa para emitir
   y para revocar. Guárdalo completo como `~/pki-lab/intermedia.cnf`.
   ```ini
   [ ca ]
   default_ca = CA_default

   [ CA_default ]
   dir               = ./ca
   database          = $dir/index.txt
   new_certs_dir     = $dir/newcerts
   serial            = $dir/serial
   crlnumber         = $dir/crlnumber
   certificate       = ./intermedia.crt
   private_key       = ./intermedia.key
   default_md        = sha256
   default_days      = 365
   default_crl_days  = 30
   policy            = policy_any
   x509_extensions   = leaf_ext

   [ policy_any ]
   commonName = supplied

   [ leaf_ext ]
   basicConstraints = critical,CA:FALSE
   ```

5. Genera el certificado hoja y fírmalo con la intermedia usando `openssl ca` (así queda
   anotado en `index.txt` y podrá revocarse después).
   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout hoja.key -out hoja.csr \
     -subj "/C=CR/O=Lab/CN=www.lab.local"
   openssl ca -batch -config intermedia.cnf -in hoja.csr -out hoja.crt
   ```
   Termina en `Database updated`.

6. Verifica la cadena con el comando de la nota. La hoja se valida subiendo por la intermedia
   hasta la raíz.
   ```bash
   openssl verify -CAfile raiz.crt -untrusted intermedia.crt hoja.crt
   ```
   Esperado: `hoja.crt: OK`.

7. Revoca la hoja y genera la CRL firmada por la intermedia.
   ```bash
   openssl ca -config intermedia.cnf -revoke hoja.crt
   openssl ca -config intermedia.cnf -gencrl -out intermedia.crl
   openssl crl -in intermedia.crl -noout -text | grep -A1 "Revoked Certificates"
   ```
   Debes ver `Serial Number: 1000` en la lista de revocados.

8. Comprueba que, con la CRL, la verificación ahora rechaza la hoja.
   ```bash
   cat raiz.crt intermedia.crt > cadena.crt
   openssl verify -crl_check -CAfile cadena.crt -CRLfile intermedia.crl hoja.crt
   ```
   Esperado: `error 23 ... certificate revoked`. El mismo certificado que era `OK` ahora se
   rechaza porque está en la CRL.

### Resultado esperado

Una PKI de tres niveles en `~/pki-lab`, la hoja verificada como `OK` antes de revocar, una
CRL firmada con el serial de la hoja, y la misma hoja rechazada con `certificate revoked`
después. Eso demuestra el ciclo completo: emisión, confianza y revocación.

### Comprueba que lo lograste

- ¿Por qué la hoja pasa la verificación sin tener la intermedia en el almacén de confianza?
  Porque se la pasas con `-untrusted`; openssl encadena hoja → intermedia → raíz, y la raíz sí
  es de confianza.
- ¿Qué cambia entre el paso 6 y el 8 si el certificado es el mismo? Que en el 8 compruebas la
  CRL con `-crl_check`, y como el serial está revocado, la cadena deja de ser válida.

### Limpieza

```bash
rm -rf ~/pki-lab
```

## Ejercicio 3: Compara FTP y SFTP en Wireshark

Nodo: FTP vs SFTP ([README](README.md#ftp-vs-sftp)).

Objetivo: subir un archivo por FTP y por SFTP entre dos VMs tuyas mientras capturas el
tráfico, y ver con tus ojos que en FTP aparecen `USER` y `PASS` en texto claro mientras que
en SFTP todo son paquetes SSH cifrados.

Necesitas: dos VMs Linux en una red interna aislada (host-only o interna), nada de internet.
Una hace de servidor (IP de ejemplo `10.0.0.10`) con `vsftpd` y el servidor SSH ya incluido;
la otra de cliente con un cliente FTP, `sftp` y Wireshark. Cuenta de prueba en el servidor,
nunca una real. 45 minutos.

- Servidor Debian/Ubuntu: `sudo apt install vsftpd` (SSH suele venir con `openssh-server`).
- Servidor Arch: `sudo pacman -S vsftpd openssh && sudo systemctl enable --now vsftpd sshd`
- Cliente Debian/Ubuntu: `sudo apt install ftp wireshark openssh-client`
- Cliente Arch: `sudo pacman -S inetutils wireshark-qt openssh`

### Pasos

1. En el servidor, arranca vsftpd y crea un usuario de prueba solo para el laboratorio.
   ```bash
   sudo systemctl enable --now vsftpd
   sudo useradd -m pruebaftp && echo 'pruebaftp:Lab12345' | sudo chpasswd
   ```

2. En el cliente, arranca la captura en la interfaz de la red interna. Puedes usar la GUI de
   Wireshark o `tshark` en terminal. Deja la captura corriendo.
   ```bash
   sudo tshark -i eth1 -f "host 10.0.0.10" -w ~/captura.pcapng
   ```
   Ajusta `eth1` a tu interfaz de la red interna (mírala con `ip a`).

3. En otra terminal del cliente, sube un archivo por FTP con la cuenta de prueba.
   ```bash
   echo "contenido de laboratorio" > ~/subida.txt
   ftp 10.0.0.10
   # dentro del prompt: usuario pruebaftp, contraseña Lab12345
   # ftp> put subida.txt
   # ftp> bye
   ```

4. Sube el mismo archivo por SFTP (va por SSH en el puerto 22).
   ```bash
   sftp pruebaftp@10.0.0.10
   # sftp> put subida.txt
   # sftp> bye
   ```

5. Para la captura (Ctrl-C en la terminal de tshark) y busca las credenciales del FTP en claro.
   ```bash
   tshark -r ~/captura.pcapng -Y 'ftp.request.command == "USER" || ftp.request.command == "PASS"' \
     -T fields -e ftp.request.command -e ftp.request.arg
   ```
   Esperado: dos líneas, `USER pruebaftp` y `PASS Lab12345`. La contraseña viaja legible.

6. Comprueba que el tráfico SFTP no enseña nada: filtra por SSH y verás solo paquetes cifrados.
   ```bash
   tshark -r ~/captura.pcapng -Y 'ssh' -T fields -e tcp.dstport -e ssh.message_code | head
   ```
   Verás el puerto 22 y mensajes de SSH, pero ni el usuario ni la contraseña ni el contenido
   del archivo aparecen en claro.

### Resultado esperado

Una captura donde el FTP revela `USER pruebaftp` y `PASS Lab12345` en texto plano y el SFTP
solo muestra SSH cifrado en el puerto 22, aunque el archivo transferido sea el mismo. Es la
idea de la nota: el protocolo seguro es el mismo trasiego metido en un túnel cifrado.

### Comprueba que lo lograste

- ¿Qué puerto usa cada uno? FTP control en 21 (datos en 20 o pasivo); SFTP todo por el 22 de SSH.
- ¿Podrías recuperar la contraseña del SFTP de la captura? No: va dentro del canal SSH cifrado,
  solo ves bytes ilegibles.

### Limpieza

```bash
sudo userdel -r pruebaftp          # en el servidor
sudo systemctl disable --now vsftpd
rm -f ~/captura.pcapng ~/subida.txt
```

## Ejercicio 4: Levanta un túnel IPsec y observa ESP en Wireshark

Nodo: IPSEC ([README](README.md#ipsec)).

Objetivo: unir dos VMs con un túnel IPsec/IKEv2 usando strongSwan, confirmar la SA con
`swanctl --list-sas` como en la nota, y comparar en Wireshark un ping sin túnel (ICMP
legible) con el mismo ping dentro del túnel (solo paquetes ESP, sin ver el ICMP).

Necesitas: dos VMs Linux en una red interna aislada, con IPs de ejemplo `10.0.0.1` y
`10.0.0.2`. strongSwan en ambas. Wireshark o tshark para capturar. 1 hora.

- Debian/Ubuntu: `sudo apt install strongswan strongswan-swanctl tshark`
- Arch: `sudo pacman -S strongswan wireshark-cli`

### Pasos

1. En las dos VMs, escribe las credenciales compartidas. Guarda en cada una
   `/etc/swanctl/conf.d/lab.conf` con el mismo secreto (es laboratorio; en producción serían
   certificados). Este es el de la VM `10.0.0.1`:
   ```
   connections {
       lab {
           version = 2
           local_addrs  = 10.0.0.1
           remote_addrs = 10.0.0.2
           local  { auth = psk }
           remote { auth = psk }
           children {
               net {
                   local_ts  = 10.0.0.1/32
                   remote_ts = 10.0.0.2/32
                   esp_proposals = aes256gcm16-x25519
                   start_action = trap
               }
           }
       }
   }
   secrets {
       ike-lab { secret = "SecretoDeLaboratorio" }
   }
   ```
   En la VM `10.0.0.2`, intercambia `local_addrs`/`remote_addrs` y `local_ts`/`remote_ts`.

2. Carga la configuración y arranca strongSwan en ambas VMs.
   ```bash
   sudo systemctl enable --now strongswan
   sudo swanctl --load-all
   ```

3. Antes del túnel, captura un ping en claro para tener la referencia. En la VM `10.0.0.1`:
   ```bash
   sudo tshark -i eth1 -f "host 10.0.0.2" -w ~/sin-tunel.pcapng &
   ping -c 3 10.0.0.2
   kill %1
   tshark -r ~/sin-tunel.pcapng -Y icmp -T fields -e ip.proto -e icmp.type | head
   ```
   Verás protocolo `1` (ICMP) y los tipos de echo request/reply: el ping es legible.

4. Fuerza el túnel y confirma la SA como en la nota.
   ```bash
   sudo swanctl --initiate --child net
   sudo swanctl --list-sas
   ```
   Esperado: una línea `ESTABLISHED, IKEv2` y debajo el hijo `INSTALLED, TUNNEL,
   ESP:AES_GCM_16-256`, igual que el ejemplo del README.

5. Con el túnel arriba, captura otra vez el mismo ping y mira el protocolo.
   ```bash
   sudo tshark -i eth1 -f "host 10.0.0.2" -w ~/con-tunel.pcapng &
   ping -c 3 10.0.0.2
   kill %1
   tshark -r ~/con-tunel.pcapng -Y "esp || icmp" -T fields -e ip.proto -e icmp.type | head
   ```
   Esperado: protocolo `50` (ESP) y ninguna línea de ICMP visible: el ping viaja cifrado dentro
   de ESP.

### Resultado esperado

Una SA establecida con IKEv2 y ESP, y dos capturas comparables: sin túnel el ICMP se lee como
protocolo 1 con sus echo request/reply; con túnel solo hay protocolo 50 (ESP) y el ICMP
desaparece de la vista. Eso demuestra que IPsec protege el paquete entero en capa 3.

### Comprueba que lo lograste

- ¿Qué número de protocolo IP identifica a ESP y a AH? ESP es 50, AH es 51; aquí ves 50 porque
  ESP es el que cifra y el que se usa casi siempre.
- ¿Por qué no ves el ICMP con el túnel arriba? Porque el paquete ICMP va dentro de la carga
  cifrada de ESP; Wireshark solo ve el envoltorio ESP.

### Limpieza

```bash
sudo swanctl --terminate --child net
sudo systemctl disable --now strongswan
rm -f ~/sin-tunel.pcapng ~/con-tunel.pcapng
```

## Ejercicio 5: Mide el coste de MD5 frente a bcrypt

Nodo: Hashing ([README](README.md#hashing)) y Salting ([README](README.md#salting)).

Objetivo: generar tú mismo los hashes de 20 contraseñas con MD5 y con bcrypt, medir con
hashcat cuántos hashes por segundo prueba cada algoritmo (benchmark y un ataque de diccionario
contra tus propios hashes), y concluir con tus números por qué un hash lento con sal y coste
protege mucho mejor las contraseñas que un hash rápido como MD5.

Necesitas: una VM Linux tuya. Python 3 con el módulo `bcrypt` (o `htpasswd`), y `hashcat`.
Trabajas solo contra hashes que tú generas. 45 minutos. Nota: el benchmark de hashcat usa
GPU si hay; sin GPU corre en CPU y los números son más bajos, pero la comparación entre MD5 y
bcrypt sigue siendo válida.

- Debian/Ubuntu: `sudo apt install hashcat python3-bcrypt apache2-utils`
- Arch: `sudo pacman -S hashcat python-bcrypt apache` (htpasswd viene en `apache`)

### Pasos

1. Inventa 20 contraseñas tuyas en un archivo, una por línea. Mézclalas: algunas débiles y
   comunes, otras largas y raras.
   ```bash
   mkdir -p ~/bcrypt-lab && cd ~/bcrypt-lab
   cat > passwords.txt <<'EOF'
   123456
   password
   qwerty
   Verano2026!
   gato
   dragon
   admin123
   Luna-Roja-7
   hunter2
   iloveyou
   P@ssw0rd
   montaña44
   correcthorse
   zzzzzz
   Jony-2026-cyber
   ftp2024
   sol!sol!sol!
   9a8b7c6d
   Secreto123
   xkcd-staple-battery
   EOF
   wc -l passwords.txt      # debe decir 20
   ```

2. Genera los hashes MD5 de esas contraseñas con hashlib (formato que hashcat espera en modo 0:
   solo el hash hex).
   ```bash
   python3 -c 'import hashlib,sys; [print(hashlib.md5(l.strip().encode()).hexdigest()) for l in open("passwords.txt")]' > md5.txt
   head -3 md5.txt
   ```

3. Genera los hashes bcrypt con coste 10 (el mínimo que pide OWASP). Con el módulo `bcrypt`:
   ```bash
   python3 -c 'import bcrypt; [print(bcrypt.hashpw(l.strip().encode(), bcrypt.gensalt(10)).decode()) for l in open("passwords.txt")]' > bcrypt.txt
   head -2 bcrypt.txt
   ```
   Alternativa sin el módulo, con htpasswd (coste con `-C 10`):
   ```bash
   : > bcrypt.txt
   while read -r p; do htpasswd -bnBC 10 "" "$p" | cut -d: -f2 >> bcrypt.txt; done < passwords.txt
   ```
   Cada línea empieza por `$2b$10$` (o `$2y$10$` con htpasswd) e incluye la sal: son 20 hashes
   distintos aunque haya contraseñas repetidas, porque cada uno lleva su propia sal.

4. Crea un diccionario pequeño que contenga solo algunas de tus contraseñas (las débiles), para
   que el ataque acierte unas y falle otras.
   ```bash
   cat > dict.txt <<'EOF'
   123456
   password
   qwerty
   gato
   admin123
   hunter2
   iloveyou
   zzzzzz
   ftp2024
   Secreto123
   EOF
   ```

5. Mide la velocidad bruta de cada algoritmo con el benchmark de hashcat. Modo 0 es MD5, modo
   3200 es bcrypt (confirmado en la [wiki de hashcat](https://hashcat.net/wiki/doku.php?id=example_hashes)).
   ```bash
   hashcat --benchmark -m 0
   hashcat --benchmark -m 3200
   ```
   Apunta el `Speed` de cada uno. En MD5 lo normal son miles de millones (GH/s) o cientos de
   millones; en bcrypt coste 10, apenas cientos o unos miles por segundo. Esa diferencia de
   varios órdenes de magnitud es el objetivo del ejercicio.

6. Lanza el ataque de diccionario contra tus propios hashes y compara cuánto tarda en romper
   los que están en el diccionario.
   ```bash
   hashcat -m 0    -a 0 md5.txt    dict.txt --potfile-disable
   hashcat -m 3200 -a 0 bcrypt.txt dict.txt --potfile-disable
   ```
   Con `--show` ves qué contraseñas cayeron:
   ```bash
   hashcat -m 0    md5.txt    --show
   hashcat -m 3200 bcrypt.txt --show
   ```
   En ambos caen las que metiste en el diccionario, pero fíjate en el tiempo y en el `Speed`
   del resumen: MD5 acaba al instante; bcrypt tarda mucho más por el mismo trabajo.

### Resultado esperado

Dos velocidades medidas por ti: la de MD5 (enorme) y la de bcrypt coste 10 (minúscula en
comparación), y un ataque de diccionario que rompe las contraseñas débiles de ambos ficheros
pero a ritmos muy distintos. La conclusión escrita: con MD5 un atacante prueba miles de
millones por segundo y revienta cualquier contraseña corta casi gratis; bcrypt con sal y coste
hace cada intento caro (el coste frena la fuerza bruta) y la sal única impide tablas
precalculadas y atacar a todos los usuarios a la vez, así que el mismo diccionario le cuesta
órdenes de magnitud más tiempo por cada usuario.

### Comprueba que lo lograste

- ¿Cuántas veces más rápido prueba hashcat MD5 que bcrypt en tu hardware? Divide los dos
  `Speed`: suelen ser varios millones de veces, y por eso MD5 jamás debe usarse para contraseñas.
- ¿Por qué hay 20 hashes bcrypt distintos aunque repitas contraseñas, pero en MD5 las
  repetidas dan el mismo hash? Por la sal: bcrypt la incrusta y la hace única por hash; MD5 sin
  sal produce el mismo resultado para la misma entrada, lo que permite tablas precalculadas.
- ¿Frena la sal el crackeo de `123456`? No; una contraseña débil cae igual. Lo que frena la
  fuerza bruta es el coste (el hash lento); la sal frena las tablas y el ataque masivo.

### Limpieza

```bash
rm -rf ~/bcrypt-lab
```
