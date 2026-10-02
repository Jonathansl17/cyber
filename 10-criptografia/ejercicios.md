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
   - `P=$(...)` y `G=$(...)` → guardan en las variables `P` y `G` la salida del comando que va dentro de `$( )`.
   - `openssl asn1parse` → muestra la estructura ASN.1 de un archivo; los parámetros DH son una secuencia de dos `INTEGER` (`p` y `g`), impresos en hexadecimal.
   - `-in dh2048.pem` → archivo a analizar (se lee como PEM, el formato por defecto).
   - `awk -F:` → `awk` procesa la salida línea a línea; `-F:` usa `:` como separador de campos.
   - `'/INTEGER/{print $NF; exit}'` → en la primera línea que contiene `INTEGER` imprime el último campo (`$NF`, el valor hex de `p`) y termina.
   - `awk -F': *'` → separador `:` seguido de cero o más espacios.
   - `'/INTEGER/{c++; if(c==2){print $NF; exit}}'` → cuenta las líneas `INTEGER` y en la segunda imprime su valor (`g`) y termina.
   - `python3 -` → el `-` hace que Python lea el programa de la entrada estándar.
   - `"$P" "$G"` → argumentos que recibe el programa en `sys.argv[1]` y `sys.argv[2]`; las comillas evitan que la shell los parta.
   - `<<'EOF' ... EOF` → here-document: todo lo que hay entre las dos marcas se pasa como entrada estándar; las comillas en `'EOF'` impiden que la shell expanda `$` dentro.
   - `int(sys.argv[1], 16)` → convierte el texto hexadecimal en un entero de Python.
   - `secrets.randbelow(p - 2) + 1` → número aleatorio criptográficamente seguro entre 1 y `p-2`: los secretos `a` y `b`.
   - `pow(g, a, p)` → (ver paso 1).
   - `p.bit_length()` → número de bits de `p`.
   - `pow(B, a, p) == pow(A, b, p)` → compara la llave calculada por cada lado.

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
- `rm` → borra archivos.
- `-f` → no pregunta y no da error si el archivo no existe.
- `dh2048.pem` → el archivo de parámetros generado en el paso 3.

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
   - `mkdir -p ~/pki-lab/ca/newcerts` → crea el directorio y todos los intermedios que falten (`-p`), sin error si ya existen; `newcerts` es donde `openssl ca` guarda una copia de cada certificado emitido.
   - `&& cd ~/pki-lab` → si `mkdir` tuvo éxito, entra en el directorio de trabajo.
   - `touch ca/index.txt` → crea vacío el archivo de base de datos de la CA (lista de emitidos y revocados).
   - `echo 1000 > ca/serial` → escribe `1000` (en hexadecimal) como siguiente número de serie a emitir; `>` redirige la salida al archivo.
   - `echo 1000 > ca/crlnumber` → siguiente número de CRL, también en hexadecimal.

2. Genera la CA raíz autofirmada (como la de la nota: vida larga, queda como ancla de confianza).
   ```bash
   openssl req -x509 -newkey rsa:2048 -sha256 -days 3650 -nodes \
     -keyout raiz.key -out raiz.crt -subj "/C=CR/O=Lab/CN=Lab Root CA"
   ```
   - `openssl req` → subcomando que crea peticiones de certificado (CSR) y, con `-x509`, certificados autofirmados.
   - `-x509` → genera directamente un certificado autofirmado en lugar de una CSR.
   - `-newkey rsa:2048` → crea a la vez una clave nueva RSA de 2048 bits.
   - `-sha256` → firma con el resumen SHA-256.
   - `-days 3650` → validez de 3650 días (unos 10 años).
   - `-nodes` → no cifra la clave privada con contraseña ("no DES"); en OpenSSL 3 está marcada como obsoleta y su nombre nuevo es `-noenc`, pero sigue funcionando.
   - `\` → continúa el comando en la línea siguiente.
   - `-keyout raiz.key` → archivo donde se guarda la clave privada.
   - `-out raiz.crt` → archivo donde se guarda el certificado.
   - `-subj "/C=CR/O=Lab/CN=Lab Root CA"` → fija el sujeto sin preguntar: país (`C`), organización (`O`) y nombre común (`CN`).

3. Genera la CA intermedia (clave + CSR) y fírmala con la raíz, marcándola como CA.
   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout intermedia.key -out intermedia.csr \
     -subj "/C=CR/O=Lab/CN=Lab Intermediate CA"
   openssl x509 -req -in intermedia.csr -CA raiz.crt -CAkey raiz.key -CAcreateserial \
     -days 1825 -sha256 \
     -extfile <(printf "basicConstraints=critical,CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign\n") \
     -out intermedia.crt
   ```
   - `openssl req -newkey rsa:2048 -nodes -keyout intermedia.key` → (ver paso 2); sin `-x509`, lo que se genera es una CSR.
   - `-out intermedia.csr` → archivo de la petición de firma (CSR).
   - `-subj "/C=CR/O=Lab/CN=Lab Intermediate CA"` → sujeto de la intermedia (ver paso 2).
   - `openssl x509` → subcomando para mostrar, convertir y firmar certificados.
   - `-req` → la entrada es una CSR, no un certificado.
   - `-in intermedia.csr` → la CSR que se va a firmar.
   - `-CA raiz.crt` → certificado de la CA que firma (la raíz); pasa a ser el emisor.
   - `-CAkey raiz.key` → clave privada de esa CA, con la que se firma.
   - `-CAcreateserial` → crea el archivo de números de serie de la CA (`raiz.srl`) si no existe.
   - `-days 1825` → validez de 1825 días (unos 5 años).
   - `-sha256` → (ver paso 2).
   - `-extfile <(printf "...")` → archivo con las extensiones X.509 que se añaden; `<( )` es sustitución de procesos de bash, que entrega la salida de `printf` como si fuera un archivo.
   - `basicConstraints=critical,CA:TRUE,pathlen:0` → marca el certificado como CA; `pathlen:0` impide que firme otras CA por debajo; `critical` obliga a todo verificador a entender la extensión.
   - `keyUsage=critical,keyCertSign,cRLSign` → la clave solo puede firmar certificados y CRL.
   - `-out intermedia.crt` → certificado firmado resultante.

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
   - `[ ca ]` → sección que lee `openssl ca` para saber qué CA usar.
   - `default_ca = CA_default` → nombre de la sección con la configuración de la CA por defecto.
   - `[ CA_default ]` → sección con los ajustes de esta CA.
   - `dir = ./ca` → variable propia con el directorio base; se reutiliza como `$dir`.
   - `database = $dir/index.txt` → base de datos en texto de los certificados emitidos y revocados (obligatoria; debe existir aunque esté vacía).
   - `new_certs_dir = $dir/newcerts` → directorio donde se guarda una copia de cada certificado emitido, con su serial como nombre.
   - `serial = $dir/serial` → archivo con el siguiente número de serie, en hexadecimal.
   - `crlnumber = $dir/crlnumber` → archivo con el siguiente número de CRL, en hexadecimal.
   - `certificate = ./intermedia.crt` → certificado de la CA que firma (la intermedia).
   - `private_key = ./intermedia.key` → clave privada de esa CA.
   - `default_md = sha256` → resumen usado al firmar certificados y CRL.
   - `default_days = 365` → validez de los certificados emitidos, en días.
   - `default_crl_days = 30` → días hasta la próxima actualización anunciada en la CRL; sin esto (o `default_crl_hours`) no se puede generar CRL.
   - `policy = policy_any` → sección que dice qué campos del sujeto se exigen.
   - `x509_extensions = leaf_ext` → sección de extensiones que se añaden a cada certificado emitido.
   - `[ policy_any ]` → la política: cada línea es un campo del sujeto.
   - `commonName = supplied` → el `CN` debe venir en la CSR; los campos que no aparecen en la política (aquí `C` y `O`) se eliminan del certificado emitido.
   - `[ leaf_ext ]` → extensiones del certificado hoja.
   - `basicConstraints = critical,CA:FALSE` → el certificado no es una CA y no puede firmar otros certificados.

5. Genera el certificado hoja y fírmalo con la intermedia usando `openssl ca` (así queda
   anotado en `index.txt` y podrá revocarse después).
   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout hoja.key -out hoja.csr \
     -subj "/C=CR/O=Lab/CN=www.lab.local"
   openssl ca -batch -config intermedia.cnf -in hoja.csr -out hoja.crt
   ```
   - `openssl req -newkey rsa:2048 -nodes -keyout hoja.key -out hoja.csr -subj ...` → (ver pasos 2 y 3); el `CN` `www.lab.local` es el nombre del servidor.
   - `openssl ca` → subcomando que actúa como una CA mínima: firma CSR, revoca y genera CRL, llevando la base de datos.
   - `-batch` → no pregunta las confirmaciones "Sign the certificate?" y "commit?"; responde sí.
   - `-config intermedia.cnf` → archivo de configuración del paso 4.
   - `-in hoja.csr` → la CSR que se firma.
   - `-out hoja.crt` → certificado emitido.

   Termina en `Database updated`.

6. Verifica la cadena con el comando de la nota. La hoja se valida subiendo por la intermedia
   hasta la raíz.
   ```bash
   openssl verify -CAfile raiz.crt -untrusted intermedia.crt hoja.crt
   ```
   - `openssl verify` → comprueba que un certificado encadena hasta un ancla de confianza.
   - `-CAfile raiz.crt` → certificados de confianza (el ancla: la raíz).
   - `-untrusted intermedia.crt` → certificados intermedios que se usan para construir la cadena pero que no se consideran de confianza por sí mismos.
   - `hoja.crt` → el certificado que se verifica.

   Esperado: `hoja.crt: OK`.

7. Revoca la hoja y genera la CRL firmada por la intermedia.
   ```bash
   openssl ca -config intermedia.cnf -revoke hoja.crt
   openssl ca -config intermedia.cnf -gencrl -out intermedia.crl
   openssl crl -in intermedia.crl -noout -text | grep -A1 "Revoked Certificates"
   ```
   - `openssl ca -config intermedia.cnf` → (ver paso 5).
   - `-revoke hoja.crt` → marca ese certificado como revocado en `index.txt`.
   - `-gencrl` → genera una CRL nueva con los revocados de la base de datos, firmada por la intermedia.
   - `-out intermedia.crl` → archivo de la CRL.
   - `openssl crl` → subcomando para inspeccionar y convertir CRL.
   - `-in intermedia.crl` → CRL a leer.
   - `-noout` → no reimprime la CRL codificada.
   - `-text` → la muestra en texto legible.
   - `grep -A1 "Revoked Certificates"` → busca esa línea y muestra también la línea siguiente (`-A1`, "after"), donde va el serial.

   Debes ver `Serial Number: 1000` en la lista de revocados.

8. Comprueba que, con la CRL, la verificación ahora rechaza la hoja.
   ```bash
   cat raiz.crt intermedia.crt > cadena.crt
   openssl verify -crl_check -CAfile cadena.crt -CRLfile intermedia.crl hoja.crt
   ```
   - `cat raiz.crt intermedia.crt > cadena.crt` → concatena los dos certificados en un solo archivo.
   - `openssl verify` → (ver paso 6).
   - `-crl_check` → además de la cadena, comprueba si el certificado hoja está revocado; exige tener la CRL de su emisor.
   - `-CAfile cadena.crt` → aquí la raíz y la intermedia como certificados de confianza.
   - `-CRLfile intermedia.crl` → archivo con la CRL (en PEM) que se carga para la comprobación.
   - `hoja.crt` → el certificado que se verifica.

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
- `rm -f` → (ver ejercicio 1).
- `-r` → borra de forma recursiva el directorio y todo su contenido.
- `~/pki-lab` → el directorio de trabajo de la PKI.

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
   - `sudo` → ejecuta el comando como root.
   - `systemctl enable` → deja el servicio activado para que arranque en cada inicio.
   - `--now` → además de habilitarlo, lo arranca en ese momento.
   - `vsftpd` → la unidad del servidor FTP.
   - `useradd` → crea un usuario.
   - `-m` → crea su directorio personal (`/home/pruebaftp`).
   - `pruebaftp` → nombre del usuario.
   - `&&` → ejecuta lo siguiente solo si `useradd` tuvo éxito.
   - `echo 'pruebaftp:Lab12345'` → imprime la pareja `usuario:contraseña`.
   - `| sudo chpasswd` → `chpasswd` lee de la entrada estándar líneas `usuario:contraseña` en claro y fija esas contraseñas (las cifra él).

2. En el cliente, arranca la captura en la interfaz de la red interna. Puedes usar la GUI de
   Wireshark o `tshark` en terminal. Deja la captura corriendo.
   ```bash
   sudo tshark -i eth1 -f "host 10.0.0.10" -w ~/captura.pcapng
   ```
   - `sudo` → capturar en una interfaz requiere privilegios.
   - `tshark` → la versión de terminal de Wireshark.
   - `-i eth1` → interfaz donde se captura. Ajusta `eth1` a tu interfaz de la red interna (mírala con `ip a`).
   - `-f "host 10.0.0.10"` → filtro de captura (sintaxis BPF/libpcap): solo guarda paquetes que vayan a o vengan del servidor.
   - `-w ~/captura.pcapng` → escribe los paquetes en ese archivo en formato pcapng en vez de mostrarlos.

3. En otra terminal del cliente, sube un archivo por FTP con la cuenta de prueba.
   ```bash
   echo "contenido de laboratorio" > ~/subida.txt
   ftp 10.0.0.10
   # dentro del prompt: usuario pruebaftp, contraseña Lab12345
   # ftp> put subida.txt
   # ftp> bye
   ```
   - `echo "contenido de laboratorio" > ~/subida.txt` → crea el archivo de prueba con ese texto (`>` redirige la salida al archivo).
   - `ftp 10.0.0.10` → abre una sesión FTP interactiva con el servidor (puerto 21); pide usuario y contraseña.
   - `put subida.txt` → dentro del cliente, sube ese archivo local al directorio actual del servidor.
   - `bye` → cierra la sesión y sale del cliente.

4. Sube el mismo archivo por SFTP (va por SSH en el puerto 22).
   ```bash
   sftp pruebaftp@10.0.0.10
   # sftp> put subida.txt
   # sftp> bye
   ```
   - `sftp` → cliente de transferencia de archivos sobre SSH.
   - `pruebaftp@10.0.0.10` → usuario y servidor al que se conecta (puerto 22 por defecto); pide la contraseña.
   - `put subida.txt` → sube el archivo local al servidor.
   - `bye` → sale de `sftp`.

5. Para la captura (Ctrl-C en la terminal de tshark) y busca las credenciales del FTP en claro.
   ```bash
   tshark -r ~/captura.pcapng -Y 'ftp.request.command == "USER" || ftp.request.command == "PASS"' \
     -T fields -e ftp.request.command -e ftp.request.arg
   ```
   - `tshark` → (ver paso 2).
   - `-r ~/captura.pcapng` → lee los paquetes de ese archivo en vez de capturar.
   - `-Y '...'` → filtro de visualización (sintaxis de Wireshark): solo los paquetes cuyo comando FTP sea `USER` o `PASS`; `||` es "o".
   - `-T fields` → la salida son solo los campos que se pidan con `-e`, separados por tabulador.
   - `-e ftp.request.command` → imprime el comando FTP (`USER`, `PASS`).
   - `-e ftp.request.arg` → imprime su argumento (el usuario o la contraseña).

   Esperado: dos líneas, `USER pruebaftp` y `PASS Lab12345`. La contraseña viaja legible.

6. Comprueba que el tráfico SFTP no enseña nada: filtra por SSH y verás solo paquetes cifrados.
   ```bash
   tshark -r ~/captura.pcapng -Y 'ssh' -T fields -e tcp.dstport -e ssh.message_code | head
   ```
   - `tshark -r ~/captura.pcapng -T fields` → (ver paso 5).
   - `-Y 'ssh'` → muestra solo paquetes que Wireshark reconoce como SSH.
   - `-e tcp.dstport` → puerto TCP de destino.
   - `-e ssh.message_code` → código del mensaje SSH; solo se ve en los paquetes de la negociación sin cifrar, después queda vacío.
   - `| head` → muestra solo las 10 primeras líneas.

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
- `userdel` → borra un usuario.
- `-r` → borra también su directorio personal y su buzón de correo.
- `pruebaftp` → el usuario de prueba.
- `# en el servidor` → comentario: la shell ignora desde `#` hasta el final de la línea.
- `systemctl disable --now vsftpd` → deshabilita el arranque automático y, por `--now`, para el servicio ya.
- `rm -f` → (ver ejercicio 1); aquí borra la captura y el archivo subido en el cliente.

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
   - `connections { }` → bloque con las conexiones IKE definidas.
   - `lab { }` → nombre de esta conexión (el que se usa para referirse a ella).
   - `version = 2` → usa IKEv2 (`1` sería el IKEv1 obsoleto).
   - `local_addrs = 10.0.0.1` → dirección local desde la que se negocia IKE.
   - `remote_addrs = 10.0.0.2` → dirección del otro extremo.
   - `local { auth = psk }` → este equipo se autentica con clave precompartida (PSK).
   - `remote { auth = psk }` → se exige que el otro extremo también se autentique con PSK.
   - `children { }` → bloque con las CHILD_SA, las asociaciones que protegen el tráfico de verdad.
   - `net { }` → nombre de la CHILD_SA (el que usa `--child net` más abajo).
   - `local_ts = 10.0.0.1/32` → selector de tráfico local: qué tráfico propio entra en el túnel (solo esta IP).
   - `remote_ts = 10.0.0.2/32` → selector de tráfico remoto: hacia qué destino se cifra.
   - `esp_proposals = aes256gcm16-x25519` → algoritmos de ESP: AES-256 en modo GCM con ICV de 16 bytes (cifra y autentica a la vez) y el grupo X25519 para el Diffie-Hellman de las renegociaciones (secreto perfecto hacia adelante).
   - `start_action = trap` → instala una política "trampa": el túnel se levanta solo en cuanto aparece tráfico que encaja con los selectores.
   - `secrets { }` → bloque de credenciales.
   - `ike-lab { secret = "SecretoDeLaboratorio" }` → una PSK para IKE (el prefijo `ike` es obligatorio, `-lab` es un nombre libre); como no lleva `id`, se usa para cualquier identidad si no hay otra mejor.

   En la VM `10.0.0.2`, intercambia `local_addrs`/`remote_addrs` y `local_ts`/`remote_ts`.

2. Carga la configuración y arranca strongSwan en ambas VMs.
   ```bash
   sudo systemctl enable --now strongswan
   sudo swanctl --load-all
   ```
   - `systemctl enable --now strongswan` → (ver ejercicio 3); habilita y arranca el servicio `strongswan`, el demonio IKE (`charon-systemd`) al que habla `swanctl`.
   - `swanctl` → herramienta de control de strongSwan.
   - `--load-all` → (re)carga credenciales, pools, autoridades y conexiones desde `/etc/swanctl/`.

3. Antes del túnel, captura un ping en claro para tener la referencia. En la VM `10.0.0.1`:
   ```bash
   sudo tshark -i eth1 -f "host 10.0.0.2" -w ~/sin-tunel.pcapng &
   ping -c 3 10.0.0.2
   kill %1
   tshark -r ~/sin-tunel.pcapng -Y icmp -T fields -e ip.proto -e icmp.type | head
   ```
   - `sudo tshark -i eth1 -f "host 10.0.0.2" -w ~/sin-tunel.pcapng` → (ver ejercicio 3, paso 2); captura el tráfico con la otra VM.
   - `&` → deja la captura corriendo en segundo plano, como trabajo número 1 de la shell.
   - `ping` → manda ICMP echo request.
   - `-c 3` → solo 3 paquetes y termina.
   - `10.0.0.2` → la otra VM.
   - `kill %1` → termina el trabajo en segundo plano número 1 (la captura).
   - `tshark -r ... -T fields` → (ver ejercicio 3, paso 5).
   - `-Y icmp` → solo paquetes ICMP.
   - `-e ip.proto` → número de protocolo IP del paquete.
   - `-e icmp.type` → tipo ICMP (8 echo request, 0 echo reply).
   - `| head` → (ver ejercicio 3, paso 6).

   Verás protocolo `1` (ICMP) y los tipos de echo request/reply: el ping es legible.

4. Fuerza el túnel y confirma la SA como en la nota.
   ```bash
   sudo swanctl --initiate --child net
   sudo swanctl --list-sas
   ```
   - `swanctl --initiate` → inicia una conexión.
   - `--child net` → la CHILD_SA que se inicia, por su nombre (`net`, del paso 1); strongSwan levanta también la IKE_SA padre.
   - `swanctl --list-sas` → lista las IKE_SA activas y sus CHILD_SA.

   Esperado: una línea `ESTABLISHED, IKEv2` y debajo el hijo `INSTALLED, TUNNEL,
   ESP:AES_GCM_16-256`, igual que el ejemplo del README.

5. Con el túnel arriba, captura otra vez el mismo ping y mira el protocolo.
   ```bash
   sudo tshark -i eth1 -f "host 10.0.0.2" -w ~/con-tunel.pcapng &
   ping -c 3 10.0.0.2
   kill %1
   tshark -r ~/con-tunel.pcapng -Y "esp || icmp" -T fields -e ip.proto -e icmp.type | head
   ```
   - `tshark`, `&`, `ping -c 3`, `kill %1`, `-T fields`, `-e ip.proto`, `-e icmp.type` y `head` → (ver paso 3).
   - `-w ~/con-tunel.pcapng` → archivo de esta segunda captura.
   - `-Y "esp || icmp"` → muestra los paquetes ESP o ICMP.

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
- `swanctl --terminate` → cierra una conexión.
- `--child net` → la CHILD_SA que se cierra, por su nombre.
- `systemctl disable --now strongswan` → (ver ejercicio 3).
- `rm -f` → (ver ejercicio 1); borra las dos capturas.

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
   - `mkdir -p ~/bcrypt-lab && cd ~/bcrypt-lab` → (ver ejercicio 2, paso 1); crea el directorio de trabajo y entra en él.
   - `cat > passwords.txt` → `cat` copia su entrada estándar al archivo `passwords.txt` (`>` lo crea o lo sobrescribe).
   - `<<'EOF' ... EOF` → here-document (ver ejercicio 1, paso 4): las líneas entre las marcas son la entrada de `cat`.
   - `wc -l passwords.txt` → `wc` cuenta; `-l` cuenta solo líneas.
   - `# debe decir 20` → comentario, la shell lo ignora.

2. Genera los hashes MD5 de esas contraseñas con hashlib (formato que hashcat espera en modo 0:
   solo el hash hex).
   ```bash
   python3 -c 'import hashlib,sys; [print(hashlib.md5(l.strip().encode()).hexdigest()) for l in open("passwords.txt")]' > md5.txt
   head -3 md5.txt
   ```
   - `python3 -c` → (ver ejercicio 1, paso 1).
   - `open("passwords.txt")` → abre el archivo y lo recorre línea a línea (`for l in ...`).
   - `l.strip()` → quita el salto de línea y los espacios de los extremos.
   - `.encode()` → convierte el texto a bytes (UTF-8), que es lo que acepta la función de hash.
   - `hashlib.md5(...).hexdigest()` → calcula el MD5 y lo devuelve en hexadecimal (32 caracteres), sin sal.
   - `> md5.txt` → guarda la salida en `md5.txt`, un hash por línea.
   - `head -3 md5.txt` → (ver ejercicio 1, paso 3); muestra los 3 primeros hashes.

3. Genera los hashes bcrypt con coste 10 (el mínimo que pide OWASP). Con el módulo `bcrypt`:
   ```bash
   python3 -c 'import bcrypt; [print(bcrypt.hashpw(l.strip().encode(), bcrypt.gensalt(10)).decode()) for l in open("passwords.txt")]' > bcrypt.txt
   head -2 bcrypt.txt
   ```
   - `python3 -c`, `open(...)`, `l.strip().encode()` y `> archivo` → (ver paso 2).
   - `bcrypt.gensalt(10)` → genera una sal aleatoria con factor de coste 10 (2^10 iteraciones internas).
   - `bcrypt.hashpw(contraseña, sal)` → calcula el hash bcrypt; el resultado lleva dentro el algoritmo, el coste y la sal.
   - `.decode()` → pasa el resultado de bytes a texto para imprimirlo.
   - `head -2 bcrypt.txt` → muestra los 2 primeros hashes.

   Alternativa sin el módulo, con htpasswd:
   ```bash
   : > bcrypt.txt
   while read -r p; do htpasswd -bnBC 10 "" "$p" | cut -d: -f2 >> bcrypt.txt; done < passwords.txt
   ```
   - `: > bcrypt.txt` → `:` es un comando que no hace nada; con `>` el efecto es crear o vaciar el archivo.
   - `while read -r p; do ...; done < passwords.txt` → lee el archivo línea a línea y guarda cada una en la variable `p`; `-r` evita que las barras invertidas se interpreten.
   - `htpasswd` → herramienta de Apache para generar contraseñas cifradas.
   - `-b` → modo batch: toma la contraseña de la línea de comandos en vez de pedirla.
   - `-n` → muestra el resultado en pantalla en lugar de escribir en un archivo de contraseñas (por eso no se le pasa archivo).
   - `-B` → usa bcrypt.
   - `-C 10` → coste de bcrypt (solo con `-B`; por defecto 5, válido de 4 a 17).
   - `""` → nombre de usuario vacío; la salida queda `:hash`.
   - `"$p"` → la contraseña de la línea actual.
   - `| cut -d: -f2` → `cut` corta cada línea usando `:` como delimitador (`-d:`) y se queda con el segundo campo (`-f2`): el hash.
   - `>> bcrypt.txt` → añade al final del archivo sin borrar lo anterior.
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
   - `cat > dict.txt <<'EOF' ... EOF` → (ver paso 1); escribe el diccionario en `dict.txt`, una palabra por línea.

5. Mide la velocidad bruta de cada algoritmo con el benchmark de hashcat. Modo 0 es MD5, modo
   3200 es bcrypt (confirmado en la [wiki de hashcat](https://hashcat.net/wiki/doku.php?id=example_hashes)).
   ```bash
   hashcat --benchmark -m 0
   hashcat --benchmark -m 3200
   ```
   - `hashcat` → herramienta de recuperación de contraseñas por fuerza bruta y diccionario.
   - `--benchmark` → (forma larga de `-b`) mide la velocidad del algoritmo elegido sin atacar nada.
   - `-m 0` → tipo de hash 0: MD5.
   - `-m 3200` → tipo de hash 3200: bcrypt (`$2*$`).

   Apunta el `Speed` de cada uno. En MD5 lo normal son miles de millones (GH/s) o cientos de
   millones; en bcrypt coste 10, apenas cientos o unos miles por segundo. Esa diferencia de
   varios órdenes de magnitud es el objetivo del ejercicio.

6. Lanza el ataque de diccionario contra tus propios hashes y compara cuánto tarda en romper
   los que están en el diccionario.
   ```bash
   hashcat -m 0    -a 0 md5.txt    dict.txt --potfile-disable
   hashcat -m 3200 -a 0 bcrypt.txt dict.txt --potfile-disable
   ```
   - `-m 0` y `-m 3200` → (ver paso 5).
   - `-a 0` → modo de ataque 0, "straight": prueba cada palabra del diccionario tal cual.
   - `md5.txt` / `bcrypt.txt` → archivo con los hashes a atacar (va primero).
   - `dict.txt` → el diccionario (va después del archivo de hashes).
   - `--potfile-disable` → no escribe las contraseñas encontradas en el potfile (`hashcat.potfile`), el registro donde hashcat guarda lo ya roto.

   Con `--show` ves qué contraseñas cayeron:
   ```bash
   hashcat -m 0    md5.txt    --show
   hashcat -m 3200 bcrypt.txt --show
   ```
   - `-m 0` / `-m 3200` y los archivos de hashes → (ver arriba).
   - `--show` → no ataca: cruza la lista de hashes con el potfile y muestra los que ya están rotos, como `hash:contraseña`.

   Ojo: `--show` lee el potfile, y el ataque anterior se lanzó con `--potfile-disable`, así que no guardó nada en él. Para que `--show` muestre resultados, repite el ataque sin `--potfile-disable`.
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
- `rm -rf` → (ver ejercicio 2); borra el directorio del laboratorio con todo su contenido.
