# Ejercicios: Ataques web y de red

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo se hace sobre equipos, VMs, topologías de laboratorio y dominios públicos consultados de
forma pasiva. El código vulnerable se compila y se ejecuta en tu propia máquina para observar
el fallo y corregirlo; no se ataca nada de terceros.

## Ejercicio 1: Reproduce un desbordamiento y una fuga de memoria con AddressSanitizer

Nodo: Buffer Overflow ([README](README.md#buffer-overflow)) y Memory Leak ([README](README.md#memory-leak)).

Objetivo: compilar con AddressSanitizer los ejemplos vulnerables de `greet` (desbordamiento de
pila) y de `handle` (fuga de memoria) de la nota, provocar el fallo con una entrada larga o
inválida, leer el informe de ASan hasta la línea exacta culpable, y comprobar que las versiones
corregidas de la nota salen limpias.

Necesitas: Linux con `gcc` (o `clang`). Es ejercicio defensivo: el objetivo es ver el informe y
arreglar el bug, no explotarlo. 40 minutos.

- Debian/Ubuntu: `sudo apt install gcc` (ASan viene con el compilador)
- Arch: `sudo pacman -S gcc`

### Pasos (desbordamiento de pila: greet)

1. Guarda la versión vulnerable de la nota dentro de un programa completo que puedas ejecutar.
   Crea `greet_vuln.c`:
   ```c
   #include <stdio.h>
   #include <string.h>

   /* Vulnerable: no bounds check. */
   void greet(const char *name) {
       char buf[64];
       strcpy(buf, name);
       printf("Hello %s\n", buf);
   }

   int main(int argc, char **argv) {
       if (argc < 2) { fprintf(stderr, "uso: %s <nombre>\n", argv[0]); return 2; }
       greet(argv[1]);
       return 0;
   }
   ```

2. Compila con AddressSanitizer y símbolos de depuración, y ejecútalo con un nombre de más de
   63 caracteres para desbordar `buf`.
   ```bash
   gcc -fsanitize=address -g -o greet_vuln greet_vuln.c
   ./greet_vuln "$(python3 -c 'print("A"*200)')"
   ```
   ASan aborta el programa y escribe un informe que empieza así:
   ```
   ==NNNNN==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x...
   WRITE of size 201 at 0x... thread T0
       #1 ... in greet greet_vuln.c:7
       #2 ... in main greet_vuln.c:13
   ```
   La línea 7 (`strcpy`) es la culpable: escribió 201 bytes en un búfer de 64.

3. Guarda la versión corregida de la nota en `greet_fixed.c` (copia acotada y rechazo si no cabe)
   y comprueba que con la misma entrada larga ya no desborda, sino que rechaza.
   ```c
   #include <stdio.h>
   #include <stddef.h>
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

   int main(int argc, char **argv) {
       if (argc < 2) { fprintf(stderr, "uso: %s <nombre>\n", argv[0]); return 2; }
       if (greet(argv[1]) != 0) { fprintf(stderr, "nombre demasiado largo, rechazado\n"); return 1; }
       return 0;
   }
   ```
   ```bash
   gcc -fsanitize=address -g -o greet_fixed greet_fixed.c
   ./greet_fixed "$(python3 -c 'print("A"*200)')"; echo "codigo de salida: $?"
   ```
   Esperado: `nombre demasiado largo, rechazado` y código de salida 1. ASan no dice nada porque
   no hay escritura fuera de límites.

### Pasos (fuga de memoria: handle)

4. Guarda la versión con fuga de la nota en un programa completo. `handle` sale por el `return`
   del tamaño inválido sin liberar; un `main` que lo llama 100 veces acumula la fuga. Crea
   `handle_vuln.c`:
   ```c
   #include <stdio.h>
   #include <stdlib.h>
   #include <string.h>

   struct request { size_t len; };

   static int is_valid_size(size_t len) { return len <= 4096; }
   static void process(char *body, const struct request *req) { memset(body, 0, req->len); }

   /* Leaks on the error path. */
   int handle(const struct request *req) {
       char *body = malloc(req->len);
       if (body == NULL) return -1;
       if (!is_valid_size(req->len)) return -1;   /* body never freed */
       process(body, req);
       free(body);
       return 0;
   }

   int main(void) {
       for (int i = 0; i < 100; i++) {
           struct request req = { .len = 8192 };   /* invalido: > 4096 */
           handle(&req);
       }
       printf("procesadas 100 peticiones\n");
       return 0;
   }
   ```

5. Compila y ejecuta. ASan incluye LeakSanitizer, que al terminar el programa reporta la memoria
   que nunca se liberó.
   ```bash
   gcc -fsanitize=address -g -o handle_vuln handle_vuln.c
   ./handle_vuln
   ```
   Al final del programa aparece:
   ```
   ==NNNNN==ERROR: LeakSanitizer: detected memory leaks
   Direct leak of 819200 byte(s) in 100 object(s) allocated from:
       #1 ... in handle handle_vuln.c:12
       #2 ... in main handle_vuln.c:24
   SUMMARY: AddressSanitizer: 819200 byte(s) leaked in 100 allocation(s).
   ```
   100 peticiones de 8192 bytes cada una = 819200 bytes perdidos, señalando el `malloc` de la
   línea 12 en la ruta de error.

6. Guarda la versión corregida de la nota en `handle_fixed.c` (un único punto de salida que
   siempre libera) y comprueba que sale sin fugas.
   ```c
   #include <stdio.h>
   #include <stdlib.h>
   #include <string.h>

   struct request { size_t len; };

   static int is_valid_size(size_t len) { return len <= 4096; }
   static void process(char *body, const struct request *req) { memset(body, 0, req->len); }

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

   int main(void) {
       for (int i = 0; i < 100; i++) {
           struct request req = { .len = 8192 };
           handle(&req);
       }
       printf("procesadas 100 peticiones\n");
       return 0;
   }
   ```
   ```bash
   gcc -fsanitize=address -g -o handle_fixed handle_fixed.c
   ./handle_fixed; echo "codigo de salida: $?"
   ```
   Esperado: `procesadas 100 peticiones` y código de salida 0, sin informe de fugas.

### Resultado esperado

Cuatro binarios compilados con ASan. Los dos vulnerables producen informes que apuntan a la
línea exacta del fallo (el `strcpy` del desbordamiento y el `malloc` de la fuga); los dos
corregidos terminan sin que ASan diga nada.

### Comprueba que lo lograste

- ¿Qué tipo de error reporta ASan en cada caso? En `greet`, `stack-buffer-overflow` (escritura
  fuera de un búfer de pila); en `handle`, `detected memory leaks` al terminar.
- ¿A qué línea te lleva el informe y por qué esa? Al `strcpy` y al `malloc`: ASan da el número de
  línea de la operación culpable, que es justo donde hay que corregir.
- ¿Por qué la versión corregida ya no dispara ASan? Porque `snprintf` acota la copia y rechaza lo
  que no cabe, y porque el único `free` se ejecuta en todas las rutas de salida.

### Limpieza

```bash
rm -f greet_vuln greet_vuln.c greet_fixed greet_fixed.c handle_vuln handle_vuln.c handle_fixed handle_fixed.c
```

## Ejercicio 2: Endurece los trunks contra VLAN hopping en Packet Tracer

Nodo: VLAN Hopping ([README](README.md#vlan-hopping)).

Objetivo: montar en Packet Tracer dos switches unidos por un trunk con dos VLAN, ver con
`show interfaces trunk` el estado inicial, aplicar la configuración endurecida de la nota
(puertos de acceso forzados, sin DTP, native VLAN dedicada, VLAN permitidas limitadas) y
verificar que ningún puerto de acceso negocia trunk. Es un ejercicio defensivo: el objetivo es
cerrar el vector de switch spoofing, no ejecutarlo.

Necesitas: Cisco Packet Tracer. Dos switches 2960 y al menos dos PCs. 1 hora. Nota: los nombres
de puerto dependen del modelo; en el 2960 los de acceso son `FastEthernet0/1` a `0/24` y los de
enlace `GigabitEthernet0/1` a `0/2`. Si usas un 3650, serían `GigabitEthernet1/0/x` como en la
nota.

### Pasos

1. En Packet Tracer coloca dos switches 2960 (SW1 y SW2) y cuatro PCs (dos por switch). Conecta
   SW1 y SW2 entre sí por `GigabitEthernet0/1` con un cable de cobre cruzado, y cada PC a un
   `FastEthernet` de su switch con cable directo.

2. Crea las VLAN 10 y 20 en ambos switches. Abre la CLI de SW1 (pestaña CLI del switch) y escribe:
   ```
   enable
   configure terminal
   vlan 10
    name usuarios
   vlan 20
    name servidores
   exit
   ```
   Repite lo mismo en SW2.

3. Antes de endurecer, deja el enlace entre switches negociando trunk por DTP (el estado por
   defecto peligroso) y míralo. En SW1:
   ```
   configure terminal
   interface GigabitEthernet0/1
    switchport mode dynamic desirable
   end
   show interfaces trunk
   ```
   Verás `Gi0/1` como `trunking` y, en `show dtp interface gi0/1`, que negocia. Un equipo que
   hable DTP en un puerto de acceso podría convertirlo en trunk: ese es el riesgo.

4. Aplica la configuración endurecida de la nota. Fuerza los puertos de PC a acceso y sin
   negociación, y haz el trunk estático con native VLAN dedicada y VLAN permitidas limitadas. En
   SW1:
   ```
   configure terminal
   interface range FastEthernet0/1 - 24
    switchport mode access
    switchport access vlan 10
    switchport nonegotiate
   exit
   interface GigabitEthernet0/1
    switchport mode trunk
    switchport nonegotiate
    switchport trunk native vlan 999
    switchport trunk allowed vlan 10,20
   end
   ```
   Repite en SW2. Crea también la VLAN 999 muerta (`vlan 999` / `name no-usar`) y no pongas
   ningún host en ella.

5. Verifica que el trunk quedó estático y con la native VLAN 999, como en el ejemplo de la nota.
   ```
   show interfaces trunk
   ```
   Esperado (parecido al README):
   ```
   Port        Mode         Encapsulation  Status        Native vlan
   Gi0/1       on           802.1q         trunking      999
   ```
   `Mode on` significa trunk estático, no negociado.

6. Comprueba que ningún puerto de acceso negocia trunk ni aparece como trunk.
   ```
   show interfaces switchport | include Name|Administrative Mode|Negotiation
   show dtp
   ```
   En cada `FastEthernet` de PC debe verse `Administrative Mode: static access` y
   `Negotiation of Trunking: Off`. Ningún puerto de acceso debe salir en `show interfaces trunk`.

### Resultado esperado

Una topología de dos switches con VLAN 10 y 20, un único trunk estático en Gi0/1 con native VLAN
999 y solo las VLAN 10 y 20 permitidas, y todos los puertos de PC en acceso estático con la
negociación DTP apagada. El `show interfaces trunk` lista solo el enlace entre switches, nunca
un puerto de usuario.

### Comprueba que lo lograste

- ¿Qué pasa si conectas un PC a un `FastEthernet` endurecido y ese PC intentara negociar un trunk?
  No puede: con `switchport mode access` y `nonegotiate`, el puerto no responde a DTP, así que el
  switch spoofing falla.
- ¿Por qué una native VLAN 999 dedicada y sin hosts frena el double tagging? Porque la native ya
  no es la VLAN 1 que usan los equipos, así que no hay tráfico legítimo sin etiqueta que abusar.
- ¿Aparece algún puerto de acceso en `show interfaces trunk`? No debe; si aparece, ese puerto
  quedó negociando y hay que forzarlo a acceso.

### Limpieza

Guarda el `.pkt` si quieres conservarlo o ciérralo sin guardar. No toca ningún equipo real.

## Ejercicio 3: Compara DNSSEC en un dominio firmado y en uno que falla

Nodo: DNS Poisoning ([README](README.md#dns-poisoning)).

Objetivo: usar `dig +dnssec` contra un dominio firmado correctamente y contra `dnssec-failed.org`
(un dominio de prueba que falla la validación a propósito) y comparar los dos resultados: la
bandera `ad` (datos autenticados) en el firmado frente al `SERVFAIL` del que falla, que es lo que
un resolver validador hace en vez de devolver una IP falsa.

Necesitas: Linux con `dig` y un resolver que valide DNSSEC (por ejemplo el `1.1.1.1` de
Cloudflare, que valida). Solo son consultas DNS de lectura a dominios públicos de prueba; no se
ataca nada. 20 minutos.

- Debian/Ubuntu: `sudo apt install dnsutils`
- Arch: `sudo pacman -S bind` (trae `dig`)

### Pasos

1. Consulta un dominio firmado con DNSSEC a través de un resolver validador y pide los datos
   DNSSEC. Fíjate en las banderas de la cabecera.
   ```bash
   dig @1.1.1.1 +dnssec cloudflare.com A
   ```
   En `;; flags:` debe aparecer `ad` (Authenticated Data): el resolver validó las firmas. En la
   sección de respuesta verás, junto al registro `A`, un registro `RRSIG` con la firma, igual que
   en el ejemplo de la nota.

2. Confirma de un vistazo que la bandera `ad` está presente.
   ```bash
   dig @1.1.1.1 +dnssec cloudflare.com A | grep -E "flags:|RRSIG" | head
   ```
   Esperado: una línea `;; flags: qr rd ra ad;` y al menos un `RRSIG`.

3. Consulta ahora `dnssec-failed.org`, un dominio que Verisign mantiene con firmas inválidas a
   propósito, a través del mismo resolver validador.
   ```bash
   dig @1.1.1.1 +dnssec dnssec-failed.org A
   ```
   Esperado: `;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL` y ninguna IP en la respuesta. El
   resolver, al no poder validar las firmas, prefiere no contestar antes que entregar datos que
   podrían ser falsos.

4. Para ver la diferencia con un resolver que NO valida, repite contra uno sin validación (por
   ejemplo el `+cd`, "checking disabled", que le pide al resolver que no valide):
   ```bash
   dig @1.1.1.1 +dnssec +cd dnssec-failed.org A | grep -E "status:|[0-9]+\s+IN\s+A"
   ```
   Con `+cd` el resolver sí devuelve la IP (status `NOERROR`) porque le dijiste que no validara:
   eso ilustra que la protección depende de que el resolver valide.

### Resultado esperado

Dos respuestas que se comparan directas: `cloudflare.com` con bandera `ad` y su `RRSIG`, y
`dnssec-failed.org` con `SERVFAIL` y sin IP. La conclusión: un resolver validador convierte una
firma que no cuadra en un `SERVFAIL`, cortando el envenenamiento de caché en lugar de servir la
dirección falsa.

### Comprueba que lo lograste

- ¿Qué significa la bandera `ad` en la cabecera? Que el resolver validó la cadena de firmas DNSSEC
  de esa respuesta (Authenticated Data).
- ¿Por qué `dnssec-failed.org` devuelve `SERVFAIL` y no una IP? Porque sus firmas no validan y un
  resolver validador prefiere fallar antes que entregar datos potencialmente manipulados.
- ¿Qué cambió al añadir `+cd`? Le pediste al resolver que no validara, así que devolvió la IP: sin
  validación, DNSSEC no te protege.

### Limpieza

No aplica: solo se hicieron consultas DNS de lectura.
