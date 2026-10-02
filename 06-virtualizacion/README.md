# Virtualización

## Conceptos previos

- Hardware: las piezas físicas de una computadora: CPU, memoria RAM, disco y tarjeta de red.
- Sistema operativo: el programa que administra el hardware y reparte CPU, memoria y disco entre los programas (ver [`01-fundamentos-it`](../01-fundamentos-it/)).
- Kernel: el núcleo del sistema operativo; es la única parte que habla directamente con el hardware (ver [`02-linux`](../02-linux/)).
- Modo privilegiado (ring 0): el nivel de la CPU donde corre el kernel y desde el que se puede tocar cualquier cosa; las aplicaciones corren en modo usuario (ring 3), con permisos limitados.
- Imagen de disco: un archivo grande (`.vmdk`, `.vdi`, `.qcow2`) que hace de disco duro completo para una máquina virtual.
- ISO: archivo con el contenido de un disco de instalación; se "mete" en la máquina virtual como si fuera un DVD.
- NAT: técnica que deja salir a internet a varias máquinas detrás de una sola dirección IP (ver [`03-fundamentos-de-redes`](../03-fundamentos-de-redes/)).
- Malware: software diseñado para dañar, espiar o tomar control de un sistema (ver [`11-ataques-y-amenazas`](../11-ataques-y-amenazas/)).
- Exploit: código que aprovecha un fallo concreto de un programa para hacer algo que el programa no debería permitir.

## Basics of Virtualization

La **virtualización** es una técnica que permite que una sola computadora física se comporte como varias computadoras independientes, cada una con su propio sistema operativo, su propia memoria y su propio disco. Existe porque los servidores físicos pasaban la mayor parte del tiempo ociosos (un servidor típico usaba el 10-15 % de su CPU) y porque instalar una máquina por cada servicio era caro, lento y difícil de mover. Para seguridad importa por dos razones: permite aislar lo peligroso (abrir malware sin arriesgar la máquina real) y es la base de toda la nube (ver [`18-cloud`](../18-cloud/)).

Analogía: un edificio de apartamentos. El terreno y los cimientos (el hardware) son uno solo, pero dentro hay diez apartamentos con su propia puerta, su propia cocina y su propia llave. El administrador del edificio (el hypervisor) reparte agua y electricidad y se asegura de que nadie entre al apartamento del vecino.

```
        +-----------+  +-----------+  +-----------+
        |   App     |  |   App     |  |   App     |
        | Guest OS  |  | Guest OS  |  | Guest OS  |
        |  (VM 1)   |  |  (VM 2)   |  |  (VM 3)   |
        +-----------+  +-----------+  +-----------+
        +-------------------------------------------+
        |               Hypervisor                  |
        +-------------------------------------------+
        |     Hardware físico (CPU, RAM, disco)     |
        +-------------------------------------------+
```

### Hypervisor

Un **hypervisor** (o VMM, virtual machine monitor) es un programa que crea y ejecuta máquinas virtuales repartiendo entre ellas el hardware real y evitando que una toque los recursos de otra. Resuelve el problema de que un sistema operativo está escrito para creerse dueño absoluto de la máquina: si pones dos en el mismo hardware sin árbitro, ambos intentarán controlar la misma memoria y los mismos discos.

Cómo funciona por dentro. El sistema operativo invitado intenta ejecutar instrucciones privilegiadas (cambiar tablas de memoria, hablar con el disco). Las CPU modernas tienen extensiones de virtualización por hardware, Intel VT-x y AMD-V, que crean un nivel aún más privilegiado que ring 0 (a veces llamado "ring -1"): cuando el invitado ejecuta una de esas instrucciones, la CPU lo detiene y le pasa el control al hypervisor (un "VM exit"), que hace la operación de forma segura y devuelve el control. Para la memoria se usan tablas de páginas anidadas (Intel EPT, AMD NPT): el invitado cree que tiene las direcciones 0 a 4 GB, y la CPU traduce esas direcciones a otras distintas de la RAM real que el hypervisor eligió. Así cada VM ve "su" memoria y no puede ni nombrar la de la vecina. Los dispositivos (tarjeta de red, disco) se emulan por software o se aceleran con drivers paravirtualizados (virtio en KVM, VMware Tools, Guest Additions en VirtualBox).

En Linux se comprueba si la CPU soporta virtualización por hardware así (el número es la cantidad de núcleos con el flag; 0 significa que está apagada en la BIOS o no existe):

```
$ grep -Ec '(vmx|svm)' /proc/cpuinfo
16
$ lscpu | grep Virtualization
Virtualization:                       AMD-V
```

Hay dos tipos, y la diferencia es qué hay debajo del hypervisor:

```
   Tipo 1 (bare-metal)                 Tipo 2 (hosted)
   +------+ +------+                   +------+ +------+
   | VM 1 | | VM 2 |                   | VM 1 | | VM 2 |
   +------+ +------+                   +------+ +------+
   +---------------+                   +---------------+
   |  Hypervisor   |                   |  Hypervisor   |  <- una app más
   +---------------+                   +---------------+
   |   Hardware    |                   |   Host OS     |  <- Windows, macOS, Linux
   +---------------+                   +---------------+
                                       |   Hardware    |
                                       +---------------+
```

- Tipo 1 (bare-metal): se instala directamente sobre el hardware, sin sistema operativo de por medio; él mismo es el sistema. Ejemplos: VMware ESXi, Proxmox VE (Linux con KVM), Microsoft Hyper-V, Xen. Se usa en servidores y centros de datos: menos capas significan más rendimiento y menos código que atacar.
- Tipo 2 (hosted): se instala como una aplicación más encima de un sistema operativo normal. Ejemplos: Oracle VirtualBox, VMware Workstation, VMware Fusion (Mac), Parallels. Se usa en el escritorio: lo instalas en tu laptop y en cinco minutos tienes una VM con Kali.

> [!TIP]
> Criterio rápido: si para usarlo primero tienes que arrancar Windows, macOS o Linux de escritorio, es tipo 2; si lo instalas en un servidor vacío desde una ISO y lo administras por navegador web, es tipo 1.

Ejemplo concreto: un servidor con 64 GB de RAM y 16 núcleos con ESXi (tipo 1) puede correr 8 VMs de 6 GB cada una (48 GB) y dejar margen para el propio hypervisor. La misma laptop con 16 GB y VirtualBox (tipo 2) aguanta cómodamente 2 VMs de 4 GB, porque Windows y el navegador del host ya se comen 6-8 GB.

La frontera entre tipos se difumina: KVM es un módulo del kernel de Linux, así que un Linux de escritorio con KVM se comporta como tipo 1 (el kernel mismo es el hypervisor) aunque tenga escritorio encima. Hyper-V, al activarse, mete el Windows que usabas dentro de una partición privilegiada. El examen pregunta la versión simple: tipo 1 sobre hardware, tipo 2 sobre un sistema operativo.

```
Hypervisor tipo 1 → corre directo sobre el hardware; servidores (ESXi, Proxmox, Hyper-V, Xen).
Hypervisor tipo 2 → corre como app sobre un Host OS; escritorio (VirtualBox, Workstation, Fusion).
VT-x / AMD-V → extensiones de CPU que permiten al hypervisor atrapar instrucciones privilegiadas del invitado.
```

### GuestOS

El **Guest OS** (sistema operativo invitado) es el sistema operativo que se instala y corre dentro de una máquina virtual, creyendo que tiene una computadora entera para él. Existe para poder correr un sistema distinto al de la máquina real sin reinstalar nada: un Windows Server dentro de un Linux, un Kali dentro de un Windows.

El invitado no sabe (o no necesita saber) que está virtualizado: ve una CPU, una tarjeta de red "Intel e1000" o "virtio", un disco de 40 GB, y arranca como en hardware real. Para ir más rápido se le instalan herramientas de integración (VirtualBox Guest Additions, VMware Tools, `qemu-guest-agent`) que añaden drivers optimizados, carpetas compartidas, portapapeles compartido y ajuste de resolución.

Analogía: un pez en una pecera dentro de una casa. Para el pez, la pecera es todo el mundo; no sabe que hay una sala alrededor.

Ejemplo: desde dentro de una VM Linux se puede detectar que es invitado, y el malware usa exactamente esto para comportarse "bien" cuando sospecha que está en un laboratorio (técnica de evasión MITRE T1497):

```
$ systemd-detect-virt
oracle
$ sudo dmidecode -s system-product-name
VirtualBox
```

> [!WARNING]
> Las herramientas de integración (carpetas compartidas, portapapeles, arrastrar y soltar) son puentes entre invitado y anfitrión. En una VM donde se analiza malware hay que desactivarlas: cada puente es un camino de salida.

### HostOS

El **Host OS** (sistema operativo anfitrión) es el sistema operativo instalado directamente en la máquina física, sobre el que corre un hypervisor de tipo 2. Es el "dueño de casa": controla el hardware real, y el hypervisor tipo 2 le pide recursos como cualquier otra aplicación.

Por qué importa en seguridad: el host es el punto único de fallo. Si el host está comprometido, todas sus VMs lo están (el atacante puede leer su memoria y sus discos, que son simples archivos). Por eso el host se mantiene mínimo y actualizado, y en un homelab nunca se navega por sitios dudosos desde el host "porque total, lo peligroso está en la VM".

Analogía: el dueño de un hotel. Los huéspedes (VMs) tienen su habitación con llave, pero el dueño tiene llave maestra de todas.

Ejemplo: en una laptop con Arch Linux (host) y VirtualBox, una VM de Windows 10 con disco de 50 GB es literalmente este archivo del host; quien pueda leerlo, puede montar el disco y leer todo:

```
$ ls -lh ~/VirtualBox\ VMs/Win10/
-rw------- 1 jony jony  21G Win10.vdi
-rw------- 1 jony jony  12K Win10.vbox
```

En un hypervisor tipo 1 no hay "host OS" en el sentido de escritorio: el propio hypervisor ocupa ese lugar (en Proxmox es un Debian recortado; en ESXi, el VMkernel propio de VMware).

```
Guest OS → el sistema dentro de la VM; cree que tiene hardware propio.
Host OS → el sistema en el hardware real que aloja al hypervisor tipo 2; llave maestra de todas las VMs.
```

### VM

Una **VM** (máquina virtual) es una computadora completa hecha de software, con CPU, memoria, disco y red virtuales, que ejecuta su propio sistema operativo aislado de las demás. Resuelve tres problemas a la vez: aprovechar mejor el hardware (consolidación), poder crear y destruir máquinas en minutos, y aislar cargas para que el fallo o el compromiso de una no arrastre a las otras.

Una VM, vista desde el host, es un puñado de archivos: la configuración (cuántas CPU, cuánta RAM, qué red), uno o más discos virtuales y, si existen, sus snapshots. Por eso se puede copiar, mover a otro servidor, exportar (formato OVA/OVF) y respaldar como cualquier archivo.

Analogía: un videojuego guardado. Puedes copiar la partida a un USB, abrirla en otra consola y seguir exactamente donde ibas.

Ejemplo concreto con VirtualBox desde la línea de comandos: crear una VM de 4 GB de RAM, 2 CPU y disco de 40 GB:

```
$ VBoxManage createvm --name kali-lab --ostype Debian_64 --register
Virtual machine 'kali-lab' is created and registered.
$ VBoxManage modifyvm kali-lab --memory 4096 --cpus 2 --nic1 intnet --intnet1 labnet
$ VBoxManage createmedium disk --filename kali-lab.vdi --size 40960
Medium created. UUID: 6c1e...
$ VBoxManage list runningvms
"kali-lab" {9b2f4c1e-...}
```

Los modos de red de la VM deciden con quién puede hablar, y son la primera decisión de seguridad de un laboratorio:

- NAT: la VM sale a internet a través del host; nadie de fuera puede iniciar conexión hacia ella. Bueno para instalar paquetes.
- Bridged (puente): la VM aparece en tu red física como un equipo más, con IP propia del router de la casa. Peligroso para malware: queda al lado de tu teléfono y tu tele.
- Host-only: la VM solo ve al host y a otras VMs host-only; sin internet.
- Internal network: las VMs solo se ven entre sí; ni siquiera el host. Es la red ideal para atacante + víctima.

```
NAT → VM sale a internet, nadie entra; para actualizar.
Bridged → VM es un equipo más de la LAN real; evitar con malware.
Host-only → VM y host, sin internet.
Internal → solo VMs entre sí; red de laboratorio ideal.
```

## Common Virtualization Technologies

### VMWare

**VMware** es una empresa (propiedad de Broadcom desde noviembre de 2023) que fabrica la familia de productos de virtualización más usada en empresas: hypervisors de escritorio (Workstation para Windows/Linux, Fusion para Mac) y de servidor (ESXi, administrado en conjunto con vCenter dentro de la suite vSphere). Existe porque fue la que llevó la virtualización x86 al mercado a finales de los 90 y se volvió el estándar de los centros de datos.

En seguridad aparece en dos lados: como herramienta de laboratorio (Workstation Pro es gratis para uso personal desde mayo de 2024) y como objetivo, porque un ESXi comprometido entrega decenas de servidores de golpe. Los grupos de ransomware tienen variantes específicas para ESXi que apagan las VMs y cifran directamente los archivos `.vmdk` del datastore.

Analogía: VMware es como la marca de ascensores que está en casi todos los edificios corporativos; si conoces su panel de control, te mueves por la mitad de las empresas.

Ejemplo: tras importar un OVA vulnerable (por ejemplo Metasploitable) en Workstation, la VM queda como archivos `.vmx` (configuración) y `.vmdk` (disco). Antes de explotar nada se toma un snapshot desde el menú VM → Snapshot → Take Snapshot, para volver al estado limpio en segundos.

```
Workstation → tipo 2 para Windows/Linux de escritorio.
Fusion → tipo 2 para macOS.
ESXi → tipo 1 de servidor; se administra por web o vCenter.
vCenter / vSphere → gestión centralizada de muchos ESXi.
```

### VirtualBox

**VirtualBox** es un hypervisor de tipo 2, gratuito y de código abierto (licencia GPLv3, mantenido por Oracle), que corre sobre Windows, Linux y macOS con CPU Intel/AMD. Es la puerta de entrada típica a los laboratorios: se instala en minutos, importa OVAs y casi todo tutorial de "monta tu lab" lo usa.

Cómo se usa: interfaz gráfica para lo cotidiano y `VBoxManage` para automatizar (ver el ejemplo de la sección VM). El Extension Pack, que añade USB 3 y cifrado de disco, tiene licencia distinta (PUEL): gratis para uso personal y educativo, de pago en empresas.

Analogía: es la bicicleta con rueditas de la virtualización: no es la más rápida, pero cualquiera puede empezar ese mismo día.

Ejemplo: tomar y restaurar un snapshot antes de ejecutar un ejecutable sospechoso:

```
$ VBoxManage snapshot win10-analisis take limpio --description "antes del sample"
0%...10%...100%
$ VBoxManage snapshot win10-analisis restore limpio
Restoring snapshot 'limpio' (2f0c...)
```

Ojo práctico: VirtualBox y KVM compiten por las extensiones VT-x/AMD-V. En Linux, si el módulo `kvm` está cargado, VirtualBox puede negarse a arrancar VMs o ir muy lento; hay que usar uno u otro a la vez.

### esxi

**ESXi** es el hypervisor de tipo 1 de VMware: un sistema mínimo (el VMkernel) que se instala directamente en un servidor físico y no tiene escritorio, solo una consola de texto y una interfaz web (Host Client) para crear y administrar VMs. Existe para exprimir servidores en producción: casi no hay capas entre la VM y el hardware, y gestiona memoria con trucos como compartir páginas idénticas entre VMs.

Detalles que importan en seguridad:

- La gestión va por HTTPS (puerto 443) y SSH (22, apagado por defecto). Nunca debe exponerse a internet: las campañas como ESXiArgs (2023) cifraron miles de ESXi expuestos a través del servicio OpenSLP (puerto 427).
- Los discos de las VMs viven en datastores como archivos `.vmdk`; quien obtiene root en ESXi tiene todos los discos.
- La edición gratuita fue retirada por Broadcom en 2024 y reintroducida en 2025 con la versión 8.0 Update 3e; su disponibilidad cambia, así que para un homelab Proxmox es la opción más estable.

Analogía: ESXi es el administrador de un edificio que vive en el sótano y no tiene apartamento propio: su única función es el edificio.

Ejemplo: desde la consola SSH de un ESXi de laboratorio:

```
[root@esxi:~] vim-cmd vmsvc/getallvms
Vmid   Name          File                               Guest OS
1      dc01          [datastore1] dc01/dc01.vmx         windows2019srv_64Guest
2      kali          [datastore1] kali/kali.vmx         debian11_64Guest
[root@esxi:~] vim-cmd vmsvc/power.getstate 2
Powered on
```

### proxmox

**Proxmox VE** (Virtual Environment) es una plataforma de virtualización de tipo 1, gratuita y de código abierto, basada en Debian, que combina dos tecnologías: KVM/QEMU para máquinas virtuales completas y LXC para contenedores de sistema. Se administra por navegador en el puerto 8006 y es el estándar de facto de los homelabs.

Por qué existe: ofrece lo que ESXi da en empresas (clúster, migración en vivo, backups, almacenamiento ZFS/Ceph, firewall por VM) sin licencia. La suscripción de pago solo da acceso al repositorio "enterprise" y soporte; el repositorio "no-subscription" funciona igual para un laboratorio.

Analogía: es un taller comunitario con herramientas industriales: las mismas máquinas que una fábrica, abiertas a quien quiera aprender.

Ejemplo: desde la shell de Proxmox, listar VMs (`qm`) y contenedores (`pct`), y crear un snapshot:

```
root@pve:~# qm list
      VMID NAME        STATUS     MEM(MB)    BOOTDISK(GB) PID
       100 pfsense     running    2048              16.00 1432
       101 kali        running    4096              60.00 1520
       102 win10-vict  stopped    4096              50.00 0
root@pve:~# pct list
VMID       Status     Lock         Name
200        running                 wazuh
root@pve:~# qm snapshot 102 limpio --description "recien instalada"
```

> [!TIP]
> En Proxmox, una VM (KVM) es la opción para todo lo que deba estar realmente aislado o no sea Linux (Windows, un firewall, malware); un contenedor LXC es la opción ligera para servicios Linux de confianza (un SIEM, un DNS), porque comparte el kernel del host.

```
VMware → empresa/familia comercial dominante en empresas (Workstation, Fusion, ESXi, vSphere).
VirtualBox → tipo 2 gratis y open source; entrada típica al laboratorio.
ESXi → tipo 1 de VMware; servidor sin escritorio, gestión web; objetivo de ransomware.
Proxmox VE → tipo 1 open source sobre Debian; KVM para VMs + LXC para contenedores; web en :8006.
```

## Understand Concept of Isolation

El **aislamiento** es la propiedad de un entorno de ejecución que impide que lo que ocurre dentro de él (un fallo, un programa malicioso, un usuario) lea, modifique o afecte lo que está fuera. Es el motivo por el que la virtualización importa en seguridad: permite ejecutar algo en lo que no confías sabiendo que, si sale mal, el daño queda encerrado. Se relaciona con el principio de mínimo privilegio y la defensa en profundidad (ver [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/)).

El aislamiento no es binario: hay niveles, y cada uno pone la frontera en un sitio distinto.

```
  Más aislamiento, más costo                       Menos aislamiento, más ligero
  <---------------------------------------------------------------------------->
  Máquinas físicas    VM (hypervisor)    Contenedor (kernel     Sandbox de proceso
  separadas           frontera: CPU+     compartido)            (navegador, app)
                      hypervisor         frontera: namespaces   frontera: permisos
                                         + cgroups del kernel   del propio proceso
```

### VMs

El aislamiento de una **VM** es la separación que impone el hypervisor apoyado en el hardware: cada VM tiene su propio kernel, su propia memoria traducida por la CPU (EPT/NPT) y dispositivos virtuales, de modo que para salir de ella hay que romper el hypervisor. Es el nivel de aislamiento más fuerte que se tiene en una sola máquina física, porque la superficie de ataque entre VM y host es pequeña (unas decenas de dispositivos virtuales y llamadas al hypervisor) comparada con las más de 300 llamadas al sistema que expone un kernel Linux.

Analogía: casas separadas en el mismo terreno. Comparten el terreno, pero cada una tiene sus paredes, su techo y su cerradura.

Ejemplo: abrir un adjunto sospechoso de un correo en una VM de Windows con red "internal" y sin carpetas compartidas. Si el adjunto cifra el disco, se cifra el `.vdi` de la VM, no los documentos del host.

### Contenedores

Un **contenedor** es un proceso (o grupo de procesos) del sistema anfitrión al que el kernel le muestra una vista recortada del sistema: su propio árbol de archivos, sus propios procesos y su propia red, sin un kernel propio. Existe porque una VM completa es pesada (arranca un sistema entero, ocupa GB) mientras que un contenedor arranca en milisegundos y ocupa MB; Docker lo popularizó en 2013 para empaquetar aplicaciones con todas sus dependencias.

Cómo funciona por dentro (Linux, ver [`02-linux`](../02-linux/)):

- Namespaces: le dan al proceso su propia vista de PIDs, red, puntos de montaje, usuarios y hostname. Dentro del contenedor, la aplicación se cree el PID 1.
- cgroups: limitan cuánta CPU, memoria y E/S puede usar (por ejemplo, máximo 512 MB).
- Capabilities, seccomp y AppArmor/SELinux: recortan qué llamadas al sistema y qué privilegios de root puede usar.

```
$ docker run --rm -it --memory 512m alpine sh
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    7 root      0:00 ps
/ # uname -r
6.10.3-arch1-1        <- el kernel del HOST: no hay kernel propio
```

Analogía: apartamentos dentro de un mismo edificio con paredes de tablaroca. Cada uno tiene su puerta, pero comparten estructura, tuberías y cimientos (el kernel); un problema en los cimientos afecta a todos.

> [!WARNING]
> Un contenedor ejecutado con `--privileged`, con el socket `/var/run/docker.sock` montado o como root sin user namespaces prácticamente no está aislado: desde dentro se puede tomar el host. Es la técnica MITRE ATT&CK T1611, Escape to Host.

### El aislamiento es tan fuerte como la frontera que comparten

> [!IMPORTANT]
> Lo que define cuán aislado está algo no es la herramienta, sino qué comparte con el resto: una VM comparte el hypervisor; un contenedor comparte el kernel entero. Un fallo en lo compartido rompe el aislamiento de todos a la vez.

El aislamiento es una propiedad que se mide por la frontera compartida, y por eso no hay que confundir "está en un contenedor" con "está en una caja segura".

```
                         VM                      Contenedor            Sandbox de proceso
Kernel propio            sí                      no                    no
Qué comparte             hypervisor + CPU        kernel del host       proceso y SO
Superficie frontera      pequeña (dispositivos   grande (~300+         la del SO + reglas
                         virtuales, hypercalls)  syscalls)             del sandbox
Arranque / tamaño        minutos / GB            ms / MB               instantáneo
Para malware             sí                      no                    solo como capa extra
```

La fila "Qué comparte" es la que decide todo lo demás: cuanto más grande es lo compartido, más fácil es saltar al vecino. Por eso los proveedores cloud no ponen contenedores de clientes distintos sobre el mismo kernel: AWS los mete en microVMs Firecracker (frontera de hypervisor con arranque de contenedor) y Google usa gVisor (un kernel falso en espacio de usuario que filtra las llamadas al sistema antes de que lleguen al kernel real). Límite de la idea: una VM tampoco es invulnerable (ver VM escape), solo tiene una frontera más estrecha.

```
Aislamiento por VM → frontera = hypervisor + CPU; fuerte.
Aislamiento por contenedor → frontera = kernel compartido; ligero, más débil.
Firecracker / gVisor → microVM / kernel en espacio de usuario; contenedor con frontera reforzada.
```

### Sandboxing

El **sandboxing** es una técnica que ejecuta un programa en un entorno restringido, con permisos mínimos y acceso controlado a archivos, red y sistema, para que su comportamiento pueda observarse o contenerse sin afectar al resto del equipo. Existe porque muchas veces no se puede decidir de antemano si algo es malicioso: es más fácil dejarlo correr encerrado y mirar qué hace.

Formas habituales:

- Sandbox de aplicación: el navegador ejecuta cada pestaña en un proceso sin permisos para tocar el disco; si una página explota un fallo del motor JavaScript, todavía necesita un segundo fallo para salir del sandbox.
- Sandbox de sistema: Windows Sandbox, Firejail o Flatpak en Linux, que dan a un programa una vista recortada del sistema.
- Sandbox de análisis de malware: un entorno (Cuckoo/CAPE, Any.Run, el sandbox de un antivirus o de un gateway de correo) que ejecuta el archivo en una VM instrumentada y reporta qué archivos creó, a qué IP se conectó y qué claves de registro tocó (ver [`16-respuesta-a-incidentes-y-forense`](../16-respuesta-a-incidentes-y-forense/)).

Analogía: el arenero de un parque (de ahí el nombre "sandbox"): el niño puede hacer lo que quiera con la arena, pero la arena no sale del cajón.

Ejemplo: ejecutar un PDF sospechoso con Firejail sin red y con un home vacío y temporal:

```
$ firejail --net=none --private evince factura.pdf
Parent pid 41210, child pid 41211
Child process initialized in 38.12 ms
```

> [!NOTE]
> El malware moderno busca señales de sandbox (pocos núcleos, 2 GB de RAM, nombres de dispositivo "VBOX", ningún movimiento de ratón) y se queda quieto si las encuentra (MITRE T1497). Un sandbox que no detecta nada no prueba que el archivo sea inofensivo.

### VM escape

Un **VM escape** es un ataque que, partiendo de código que corre dentro de una máquina virtual, explota un fallo del hypervisor o de un dispositivo virtual para ejecutar código en el host, rompiendo el aislamiento. Es el peor caso de la virtualización: el atacante pasa de controlar una VM a controlar todas las del mismo host.

Cómo ocurre: el invitado puede hablar con el hypervisor a través de los dispositivos emulados (tarjeta gráfica, controlador de disquete, USB, red). Si el código que emula ese dispositivo tiene un fallo de memoria (desbordamiento de búfer, desbordamiento de entero), el invitado le manda datos manipulados y consigue escribir fuera del búfer en la memoria del proceso del hypervisor en el host.

```
   VM (atacante con root en el invitado)
      |  1. envía datos manipulados al dispositivo virtual
      v
   Código de emulación del dispositivo (QEMU, VBox) ---- fallo de memoria
      |  2. escribe fuera de su búfer
      v
   Proceso del hypervisor en el host  -> 3. ejecución de código en el host
      v
   Todas las VMs del host + el host
```

Casos reales que conviene conocer:

- VENOM (CVE-2015-3456, 2015): desbordamiento de búfer en el controlador virtual de disquete de QEMU, presente en KVM y Xen; afectaba incluso a VMs que nunca usaban disquete porque el dispositivo existía por defecto.
- Fallos en la tarjeta gráfica virtual de VirtualBox y en VMware (por ejemplo en el controlador USB) demostrados en Pwn2Own, la competencia donde se pagan cientos de miles de dólares por un escape.

Analogía: un preso que descubre que el sistema de ventilación de su celda conecta con la oficina del director. La celda era sólida; el ducto compartido no.

Defensas: mantener el hypervisor parcheado (es el componente más crítico), quitar a las VMs dispositivos que no usan (disquete, USB, audio, 3D), no dar root a nadie que no lo necesite dentro de VMs compartidas y separar en hosts distintos las cargas de distinto nivel de confianza.

```
VM escape → del invitado al host rompiendo el hypervisor; raro pero crítico.
Container escape (T1611) → del contenedor al host por kernel compartido o mala configuración; mucho más común.
Evasión de sandbox (T1497) → el malware detecta el entorno y se esconde; no sale, se camufla.
```

### Snapshots para labs

Un **snapshot** es una captura del estado completo de una máquina virtual en un instante (disco, y opcionalmente memoria RAM y configuración) a la que se puede volver en segundos. En un laboratorio de seguridad es la herramienta que hace posible romper cosas sin miedo: se toma un snapshot "limpio", se ataca, se infecta o se rompe la VM, y se restaura.

Cómo funciona por dentro: al tomar el snapshot, el disco original se congela en modo solo lectura y se crea un archivo diferencial (delta) donde a partir de ese momento se escriben todos los cambios (copy-on-write). Restaurar es simplemente tirar el delta. Por eso tomar un snapshot de una VM de 40 GB tarda un segundo, y por eso una cadena de 15 snapshots hace que cada lectura tenga que recorrer 15 archivos.

```
  disco base (40 GB, congelado)  <-  delta 1 "limpio"  <-  delta 2 "post-explotación"  <-  estado actual
```

Analogía: el "guardar partida" antes del jefe final. Si pierdes, cargas la partida y vuelves a intentar sin rehacer todo el juego.

Ejemplo de flujo en un lab de análisis:

1. Instalar Windows 10 en la VM, actualizar, instalar herramientas (Sysmon, Process Monitor, Wireshark).
2. Cambiar la red a "internal", desactivar carpetas compartidas y portapapeles.
3. Snapshot `limpio`.
4. Copiar el sample, ejecutarlo, observar 10 minutos.
5. Restaurar `limpio`. La VM vuelve a estar sin infectar en 5 segundos.

### Un snapshot no es un backup

> [!IMPORTANT]
> Un snapshot depende del disco base: si ese archivo se corrompe, se borra o se cifra, el snapshot desaparece con él. Sirve para volver atrás en un laboratorio, no para recuperarse de un desastre.

Un snapshot es un punto de retorno que vive pegado a la VM, mientras que un backup es una copia independiente guardada en otro lugar. Un ransomware que cifra los `.vmdk` de un ESXi se lleva por delante VMs y snapshots en la misma operación; el backup que estaba en otro equipo es lo único que queda.

```
                      Snapshot                     Backup
Depende del original  sí (delta sobre la base)     no (copia completa)
Dónde vive            mismo almacenamiento         otro disco/servidor/nube
Tiempo de creación    segundos                     minutos a horas
Uso                   volver atrás en el lab       recuperarse de un desastre
```

La primera fila es la que importa: todo lo demás se deriva de que el snapshot no es una copia. Además, los snapshots viejos degradan el rendimiento (VMware recomienda no mantenerlos más de 72 horas en producción); en un lab se pueden conservar, pero pocos y con nombre claro.

```
Snapshot → punto de retorno copy-on-write; depende del disco base; para labs.
Backup → copia independiente en otro lugar; para desastres.
Clon → VM nueva e independiente copiada de otra.
```

## Cómo montar un homelab de seguridad

Un **homelab de seguridad** es un conjunto de máquinas virtuales propias, aisladas de la red de la casa, donde se practican ataques y defensas legalmente y sin riesgo para terceros. Es el lugar donde se ejecuta todo lo que en este roadmap se lee: escaneos con Nmap, explotación, detección con un SIEM, análisis de malware.

Analogía: un simulador de vuelo. Puedes estrellar el avión cien veces, y lo único que pierdes es el tiempo de recargar.

Decisión de plataforma:

- Solo tienes tu laptop (16 GB de RAM o más): VirtualBox o VMware Workstation (tipo 2). Suficiente para 2-3 VMs a la vez.
- Tienes una PC vieja o un mini PC dedicado (32 GB de RAM o más es lo cómodo): Proxmox VE (tipo 1), administrado desde la laptop por navegador. Permite tener encendidos 5-8 equipos y dejarlos corriendo.

Topología mínima recomendada, con el firewall como única puerta entre el lab y la casa:

```
   Internet
      |
   Router de casa (192.168.1.0/24)
      |
   [ pfSense / OPNsense ]  <- VM firewall: WAN = bridged/NAT, LAN = red interna
      |
   Red interna del lab "labnet" (10.10.10.0/24), sin acceso a 192.168.1.0/24
      |-- Kali Linux            10.10.10.10   atacante
      |-- Metasploitable 2/3    10.10.10.20   víctima Linux vulnerable a propósito
      |-- DVWA (en Docker)      10.10.10.21   víctima web
      |-- Windows Server (DC)   10.10.10.30   Active Directory de evaluación
      |-- Windows 10/11         10.10.10.31   estación unida al dominio
      |-- Wazuh / Security Onion 10.10.10.50  SIEM: ve lo que hace el atacante
```

Recursos de referencia por VM: Kali 4 GB/2 CPU/60 GB; Metasploitable 1 GB; Windows Server 4 GB; Windows 10 4 GB; Wazuh 4-8 GB; pfSense 1-2 GB. Total holgado: unos 20-24 GB de RAM, de ahí los 32 GB recomendados.

Pasos:

1. Instalar el hypervisor y comprobar VT-x/AMD-V activo (sección Hypervisor).
2. Crear la red aislada: en VirtualBox, "Internal network" llamada `labnet`; en Proxmox, un bridge `vmbr1` sin puerto físico.
3. Montar el firewall (pfSense u OPNsense) con dos interfaces y una regla que bloquee todo tráfico del lab hacia la red de la casa.
4. Importar Kali (hay imágenes oficiales listas para VirtualBox y VMware) y las víctimas: Metasploitable, DVWA, y las ISOs de evaluación de Windows de Microsoft (180 días gratis).
5. Añadir la parte defensiva: un SIEM (Wazuh o Security Onion) que reciba logs de las víctimas (ver [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/)).
6. Snapshot `limpio` de cada víctima antes de la primera práctica.

> [!WARNING]
> Las máquinas vulnerables a propósito (Metasploitable, DVWA) nunca deben estar en modo bridged ni expuestas a internet: cualquiera que escanee tu red las tomaría en minutos. Y solo se ataca lo que es tuyo o lo que una plataforma autoriza (TryHackMe, Hack The Box).

Ejemplo de primera práctica dentro del lab, desde Kali hacia Metasploitable, que confirma que el aislamiento funciona en los dos sentidos:

```
┌──(kali㉿kali)-[~]
└─$ nmap -sV 10.10.10.20
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
22/tcp open  ssh     OpenSSH 4.7p1 Debian 8ubuntu1
80/tcp open  http    Apache httpd 2.2.8
┌──(kali㉿kali)-[~]
└─$ ping -c 2 192.168.1.1
2 packets transmitted, 0 received, 100% packet loss   <- la casa no es alcanzable: bien
```

```
Homelab → VMs propias aisladas para practicar ataque y defensa sin riesgo legal.
Red interna + firewall → el lab no puede tocar la red de la casa.
Atacante / víctimas / SIEM → los tres roles mínimos del lab.
```

## Recursos para aprender y practicar

### Videos

- [What is a Hypervisor?](https://www.youtube.com/watch?v=LMAEbB2a50M) — IBM Technology; qué hace un hypervisor y la diferencia tipo 1 / tipo 2 (Hypervisor).
- [Virtualization Explained](https://www.youtube.com/watch?v=FZR0rG3HKIk) — IBM Technology; visión general de la virtualización y sus beneficios (Basics of Virtualization).
- [Virtualization Concepts - CompTIA A+ 220-1201 - 4.1](https://www.youtube.com/watch?v=xXOIdDWUNGU) — Professor Messer; host, guest, hypervisor, tipos y requisitos (Hypervisor, GuestOS, HostOS, VM).
- [you need to learn Virtual Machines RIGHT NOW!! (Kali Linux VM, Ubuntu, Windows)](https://www.youtube.com/watch?v=wX75Z-4MEoM) — NetworkChuck; crear tu primera VM con VirtualBox (VM, VirtualBox).
- [Virtual Machines Pt. 2 (Proxmox install w/ Kali Linux)](https://www.youtube.com/watch?v=_u8qTN3cCnQ) — NetworkChuck; instalar Proxmox y una VM de Kali (proxmox, Hypervisor tipo 1).
- [Proxmox vs ESXi in 2024](https://www.youtube.com/watch?v=E-CEonr2sAA) — VirtualizationHowto; comparación práctica para homelab (esxi, proxmox).
- [Containers vs VMs: What's the difference?](https://www.youtube.com/watch?v=cjXI-yxqGTI) — IBM Technology; frontera de aislamiento de VMs frente a contenedores (Isolation).
- [Virtualization Vulnerabilities - CompTIA Security+ SY0-701 - 2.3](https://www.youtube.com/watch?v=t2JrPrzRDLA) — Professor Messer; VM escape y resource reuse (VM escape).
- [VirtualBox VM Escape: Integer Overflow Explained Clearly](https://www.youtube.com/watch?v=tya56SPQW78) — David Bombal; cómo funciona un escape real paso a paso (VM escape).
- [I'm Building A NEW Cybersecurity Homelab (5 Years Later) - Proxmox Setup](https://www.youtube.com/watch?v=gno9k9RxkQ8) — CYBERWOX; homelab de seguridad completo sobre Proxmox (homelab).
- [how to build a HACKING lab (to become a hacker)](https://www.youtube.com/watch?v=mvsiuLzpx2E) — NetworkChuck; laboratorio de hacking con VMs (homelab).

### Lectura y documentación

- [NIST SP 800-125, Guide to Security for Full Virtualization Technologies](https://csrc.nist.gov/pubs/sp/800/125/final) — riesgos y recomendaciones de seguridad para hypervisors (Hypervisor, Isolation).
- [NIST SP 800-190, Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final) — amenazas y controles de contenedores (Contenedores).
- [VirtualBox: Virtual Networking](https://www.virtualbox.org/manual/topics/networkingdetails.html) — manual oficial de los modos NAT, bridged, host-only e internal (VM, homelab).
- [Manual de VirtualBox](https://www.virtualbox.org/manual/) — referencia de `VBoxManage` y snapshots (VirtualBox).
- [Documentación de Proxmox VE](https://pve.proxmox.com/pve-docs/) y [Linux Container en Proxmox](https://pve.proxmox.com/wiki/Linux_Container) — VMs con `qm`, contenedores con `pct` (proxmox).
- [VMware vSphere (ESXi)](https://www.vmware.com/products/cloud-infrastructure/esxi-and-esx) y [VMware Workstation y Fusion](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion) — páginas oficiales de producto (VMWare, esxi).
- [Kali Linux: Virtualization](https://www.kali.org/docs/virtualization/) — guías oficiales para instalar Kali en cada hypervisor (homelab).
- [Docker Engine security](https://docs.docker.com/engine/security/) — namespaces, cgroups, capabilities y seccomp (Contenedores).
- [MITRE ATT&CK T1611, Escape to Host](https://attack.mitre.org/techniques/T1611/) y [T1497, Virtualization/Sandbox Evasion](https://attack.mitre.org/techniques/T1497/) — técnicas de escape y evasión (VM escape, Sandboxing).
- [NVD: CVE-2015-3456 (VENOM)](https://nvd.nist.gov/vuln/detail/CVE-2015-3456) y [análisis de CrowdStrike](https://www.crowdstrike.com/en-us/blog/venom-vulnerability-details/) — el escape clásico vía controlador de disquete (VM escape).
- [pfSense](https://www.pfsense.org/download/), [Wazuh](https://documentation.wazuh.com/current/index.html) y [Security Onion](https://securityonionsolutions.com/) — firewall y SIEM del lab (homelab).
- [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter) — ISOs de evaluación de Windows Server para montar un dominio de laboratorio (homelab).

### Práctica

- [TryHackMe: Hosted Hypervisors](https://tryhackme.com/room/hostedhypervisors) — gratis; hipervisores tipo 2, VMs y su superficie de ataque (Basics of Virtualization).
- [TryHackMe: Hypervisor Internals](https://tryhackme.com/room/hypervisorinternals) — gratis; cómo aísla un hipervisor por dentro (Isolation).
- [TryHackMe: Intro to Containerisation](https://tryhackme.com/room/introtocontainerisation) e [Intro to Docker](https://tryhackme.com/room/introtodockerk8pdqk) — gratis; contenedores frente a VMs y su aislamiento (Contenedores).
- [Metasploitable 3](https://github.com/rapid7/metasploitable3) y [DVWA](https://github.com/digininja/DVWA) — víctimas vulnerables para tu red interna (homelab).
- Ejercicio en casa: crea dos VMs en una red "internal", toma un snapshot `limpio` de la víctima, bórrale `/etc/passwd` desde la shell, y restaura. Mide cuánto tarda (Snapshots para labs).
- Ejercicio en casa: dentro de un contenedor `docker run --rm -it alpine sh` ejecuta `uname -r` y compáralo con el del host; luego en una VM haz lo mismo. Explica la diferencia (Isolation).

## Cuadro resumen

Todo lo visto, en una línea por término.

Basics of Virtualization

```
Virtualización → una máquina física se comporta como varias independientes.
Hypervisor → programa que crea VMs y les reparte el hardware sin que se toquen.
Hypervisor tipo 1 → corre directo sobre el hardware; servidores (ESXi, Proxmox, Hyper-V, Xen).
Hypervisor tipo 2 → corre como app sobre un Host OS; escritorio (VirtualBox, Workstation, Fusion).
VT-x / AMD-V → extensiones de CPU que permiten al hypervisor atrapar instrucciones privilegiadas del invitado.
Guest OS → el sistema dentro de la VM; cree que tiene hardware propio.
Host OS → el sistema en el hardware real que aloja al hypervisor tipo 2; llave maestra de todas las VMs.
VM → computadora hecha de software; desde el host es un puñado de archivos.
NAT → VM sale a internet, nadie entra; para actualizar.
Bridged → VM es un equipo más de la LAN real; evitar con malware.
Host-only → VM y host, sin internet.
Internal → solo VMs entre sí; red de laboratorio ideal.
```

Common Virtualization Technologies

```
VMware → empresa/familia comercial dominante en empresas (Workstation, Fusion, ESXi, vSphere).
VirtualBox → tipo 2 gratis y open source; entrada típica al laboratorio.
ESXi → tipo 1 de VMware; servidor sin escritorio, gestión web; objetivo de ransomware.
Proxmox VE → tipo 1 open source sobre Debian; KVM para VMs + LXC para contenedores; web en :8006.
```

Understand Concept of Isolation

```
Aislamiento → impedir que lo de dentro afecte lo de fuera; se mide por lo que se comparte.
Aislamiento por VM → frontera = hypervisor + CPU; fuerte.
Aislamiento por contenedor → frontera = kernel compartido; ligero, más débil.
Firecracker / gVisor → microVM / kernel en espacio de usuario; contenedor con frontera reforzada.
Namespaces / cgroups → vista propia del sistema / límite de recursos de un contenedor.
Sandboxing → ejecutar algo restringido para observarlo o contenerlo.
VM escape → del invitado al host rompiendo el hypervisor; raro pero crítico.
Container escape (T1611) → del contenedor al host por kernel compartido o mala configuración; mucho más común.
Evasión de sandbox (T1497) → el malware detecta el entorno y se esconde; no sale, se camufla.
Snapshot → punto de retorno copy-on-write; depende del disco base; para labs.
Backup → copia independiente en otro lugar; para desastres.
Clon → VM nueva e independiente copiada de otra.
```

Cómo montar un homelab de seguridad

```
Homelab → VMs propias aisladas para practicar ataque y defensa sin riesgo legal.
Red interna + firewall → el lab no puede tocar la red de la casa.
Atacante / víctimas / SIEM → los tres roles mínimos del lab.
```
