# Guía paso a paso — TP #3.a: UEFI

Esta guía es para **hacer el práctico siguiéndola en orden**. Cada parte dice qué
comandos correr, qué se tiene que ver y **qué capturar** (📸). Las capturas van en
`capturas/` con el prefijo de su parte, y el resultado de cada parte se registra en
[`PROCEDIMIENTO.md`](PROCEDIMIENTO.md), igual que en el TP #2.

La consigna está desglosada en [`CONSIGNA.md`](CONSIGNA.md). La teoría sale del documento
de la cátedra *"Arquitectura y Fundamentos del Firmware UEFI"* y del repositorio de
referencia [javierbrk/UEFI-Lessons](https://github.com/javierbrk/UEFI-Lessons)
(lecciones 00 a 04 y 42).

> **Cómo leer los bloques de comandos:** los `#` son comentarios y no se escriben. En
> GDB se escribe **un comando por línea** y se espera la respuesta antes del siguiente.

---

## Parte 0 — Qué estamos haciendo, en criollo

### El problema de fondo

En el TP #2 el programa arrancaba con un sistema operativo abajo: Linux lo cargaba en
memoria, le armaba la pila y le daba `printf` y las syscalls. Acá **no hay sistema
operativo**. Vamos a escribir un programa que corre *antes* del SO, dentro del firmware
de la máquina, y que para imprimir en pantalla usa los servicios que le ofrece ese
firmware.

Ese firmware es **UEFI**, el reemplazo del BIOS de toda la vida. El BIOS dependía de
hardware heredado (el timer 8254 y el controlador de interrupciones 8259), arrancaba en
modo real de 16 bits y estaba limitado a 1 MB de memoria. UEFI, en cambio, es modular,
corre en 32 o 64 bits y es multiplataforma (x64, ARM, etc.).

### UEFI no es lo mismo que PI

El documento de la cátedra separa dos conceptos que se suelen mezclar:

| | Qué es | Pregunta que responde |
|---|---|---|
| **UEFI** | Una **especificación de interfaz**: APIs, estructuras de datos y el entorno pre-SO | ¿Cómo le habla un programa (o un bootloader) al firmware? |
| **PI** (*Platform Initialization*) | La **arquitectura interna** del firmware | ¿Cómo se construye la plataforma desde el reset hasta tener ese entorno UEFI? |

Nuestro Hello World es un **consumidor de la interfaz UEFI**: no le importa cómo se
inicializó la memoria, solo usa lo que la especificación le promete.

### Las fases de arranque (PI)

```
 reset
   │
   ▼
┌───────┐  ┌───────┐  ┌──────────────────────┐  ┌───────────────┐  ┌────────────┐
│  SEC  │→ │  PEI  │→ │         DXE          │→ │      BDS      │→ │ OS loader  │→ RT
│ caché │  │ RAM,  │  │ DXE Dispatcher carga │  │ elige qué     │  │ llama a    │  (solo
│ como  │  │ chip- │  │ drivers, instala     │  │ arrancar      │  │ ExitBoot-  │  Runtime
│ RAM   │  │ set,  │  │ protocolos y la      │  │ (UEFI Shell)  │  │ Services() │  Services)
│       │  │ HOBs  │  │ System Table         │  │       │       │  └────────────┘
└───────┘  └───────┘  └──────────────────────┘  │       ▼       │
                                                │ HelloWorld.efi│ ← acá corre lo nuestro
                                                └───────────────┘
```

1. **SEC (Security):** no hay RAM todavía. Usa la caché del procesador como RAM
   temporal (*Cache-as-RAM*) y es la raíz de confianza.
2. **PEI (Pre-EFI Initialization):** inicializa lo mínimo indispensable (el controlador
   de memoria y parte del chipset), determina el modo de arranque (normal o
   despertar de S3) y le pasa lo que descubrió a DXE en los **HOBs** (*Hand-Off Blocks*).
3. **DXE (Driver Execution Environment):** el núcleo. El *DXE Dispatcher* carga drivers
   según sus dependencias, se instalan los buses (PCI, USB) y aparecen los servicios
   UEFI: la **System Table**, los **Boot Services** y los **Runtime Services**.
4. **BDS (Boot Device Selection):** lee las variables de arranque de la NVRAM, conecta
   consolas y discos, y le pasa el control a lo que haya que arrancar. En nuestro caso
   arranca la **UEFI Shell**, y desde la Shell lanzamos nuestra app.
5. **RT (Runtime):** cuando el cargador del SO llama a `ExitBootServices()`, se termina
   el entorno de pre-arranque. Solo sobreviven los *Runtime Services* (variables, reloj).

Nuestra app vive en el mundo de BDS: los Boot Services **siguen disponibles**, así que
podemos usar la consola. En la Parte 2 vamos a ver estas fases en el log de OVMF.

Aparte de estas fases está el **SMM** (*System Management Mode*), un modo de altísimo
privilegio, invisible para el SO y activado por interrupciones SMI. No lo usamos.

### Las estructuras que vamos a tocar

- **System Table (`EFI_SYSTEM_TABLE`).** Es la puerta de entrada: el firmware le pasa
  un puntero a esta tabla a toda aplicación. Tiene, entre otras cosas, la consola de
  salida (`ConOut`), los `BootServices` y los `RuntimeServices`.
- **Protocolos.** Un protocolo es un `struct` de punteros a función (más algunos datos)
  identificado por un **GUID**. `ConOut` apunta a un `EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL`,
  y una de sus funciones es `OutputString`. Es "orientación a objetos en C": cada
  función recibe como primer argumento el propio protocolo (`This`), como un `self`.
- **Handles.** Agrupan protocolos y representan algo concreto: un disco, una
  partición, un driver o **nuestra propia imagen** (el `ImageHandle` que recibe
  `UefiMain`).
- **Imágenes PE/COFF.** Todo ejecutable UEFI (apps, drivers, bootloaders) es un
  PE/COFF, el mismo formato que los `.exe` de Windows. Trae **información de
  reubicación**: `LoadImage()` lo puede poner en **cualquier** dirección y le corrige
  las direcciones absolutas. Esto es justamente lo que complica la depuración (Parte 5).

### La convención de llamada: no es la del TP #2

En el TP #2 usamos **System V AMD64**, la ABI de Linux. UEFI, en cambio, usa la
convención de **Microsoft x64**. En EDK2 esto lo marca la palabra `EFIAPI`, que en
x64 con GCC se define en `tools_def.txt` como:

```
-DEFIAPI=__attribute__((ms_abi))
```

| | System V (TP #2) | Microsoft x64 (UEFI) |
|---|---|---|
| Argumentos enteros 1 a 4 | `rdi`, `rsi`, `rdx`, `rcx` | **`rcx`, `rdx`, `r8`, `r9`** |
| Argumentos 5 y 6 | `r8`, `r9` | por la pila |
| Valor de retorno | `rax` | `rax` |
| Espacio reservado por el llamador | ninguno | **32 bytes de *shadow space*** justo arriba de la dirección de retorno |
| Red zone (128 bytes debajo de `rsp`) | sí | **no** (EDK2 compila con `-mno-red-zone`) |

El *shadow space* es la novedad: el llamador reserva 32 bytes en su marco para que la
función llamada guarde ahí sus cuatro argumentos de registro si quiere. En la Parte 5
lo vamos a ver en vivo.

> ¿Por qué no hay red zone? En el firmware puede llegar una interrupción en cualquier
> momento y usa **la misma pila**. Si una función guardara datos debajo de `rsp`, la
> interrupción los pisaría.

### Las herramientas

| Herramienta | Qué es |
|---|---|
| **EDK2** | La implementación de referencia de UEFI/PI (TianoCore). Trae las bibliotecas, los headers y el sistema de compilación (`build`). |
| **OVMF** | *Open Virtual Machine Firmware*: un firmware UEFI hecho con EDK2 para máquinas virtuales. Lo compilamos nosotros en modo DEBUG. |
| **QEMU** | El emulador de PC donde corre OVMF. Monta una carpeta nuestra como disco FAT y expone un *stub* de GDB. |
| **UEFI Shell** | Una línea de comandos que corre sobre UEFI. BDS la arranca y desde ahí ejecutamos los `.efi`. |
| **GDB** | El mismo depurador del TP #2, conectado de forma remota a QEMU. |

### Los archivos de un paquete EDK2

| Archivo | Qué describe | En el TP #2 sería… |
|---|---|---|
| `.inf` | Un **módulo** (una app o un driver): fuentes, punto de entrada, bibliotecas que necesita | una regla del Makefile |
| `.dsc` | Una **plataforma**: qué módulos se compilan y qué implementación de cada biblioteca se usa | el Makefile entero |
| `.dec` | Un **paquete**: qué headers, GUIDs y PCDs exporta para otros | — (usamos el de `MdePkg`) |

---

## Parte 1 — Preparar el entorno

### 1.1 Instalar las herramientas

Debian/Ubuntu:

```bash
sudo apt install build-essential git uuid-dev iasl nasm python3 qemu-system-x86 gdb
```

Arch:

```bash
sudo pacman -S base-devel git util-linux-libs acpica nasm python qemu-system-x86 gdb
```

- `uuid-dev` y `python3`: los usan las herramientas de compilación de EDK2 (BaseTools).
- `nasm` e `iasl`: hacen falta para compilar OVMF (partes en ensamblador y tablas ACPI).
- `qemu-system-x86`: el emulador.

El **gdb-dashboard** ya quedó instalado en `~/.gdbinit` desde el TP #2. Si no, ver la
Parte 1.2 de `PROCEDIMIENTO.md` del TP #2.

### 1.2 Clonar EDK2 en una versión estable

```bash
git clone --branch edk2-stable202608 --depth 1 https://github.com/tianocore/edk2 ~/edk2
cd ~/edk2
git submodule update --init --depth 1
```

- Usamos un **tag estable** y no `master`, para que lo que está en esta guía coincida
  con lo que ves.
- Los **submódulos son obligatorios**: sin ellos ni siquiera compila `MdePkg`, porque su
  `.dec` incluye headers de `MdePkg/Library/MipiSysTLib/mipisyst`, que es un submódulo.

### 1.3 Compilar las BaseTools

```bash
make -C ~/edk2/BaseTools -j$(nproc)
```

Esto compila las herramientas en C de EDK2: `GenFw`, que convierte el ELF que genera
gcc en un PE/COFF; `GenFv` y `GenFfs`, que arman el firmware, y otras. Al final corre
sus propios tests y termina con `OK`.

### 1.4 El entorno de EDK2

```bash
cd ~/edk2
source edksetup.sh
```

`edksetup.sh` define `WORKSPACE`, `EDK_TOOLS_PATH` y `CONF_PATH`, agrega las
herramientas al `PATH` y, la primera vez, crea `Conf/` copiando las plantillas
(`target.txt`, `tools_def.txt`, `build_rule.txt`).

> **Diferencia con UEFI-Lessons:** las lecciones usan `--tagname=GCC5`. En las
> versiones actuales de EDK2 ese toolchain **ya no existe**: se llama `GCC`. Por eso los
> directorios de salida terminan en `_GCC` (`DEBUG_GCC`, `NOOPT_GCC`) y no en `_GCC5`.

No hace falta tener esto cargado a mano: el `Makefile` del repo lo hace solo antes de
cada `build`.

### 1.5 Clonar este repositorio

```bash
git clone https://github.com/nicoborsottib/practico-3 ~/practico-3
cd ~/practico-3
```

Si EDK2 no está en `~/edk2`, se le pasa la ruta al `Makefile` en cada comando:
`make EDK2=/otra/ruta/edk2 ...`

### 1.6 Compilar OVMF (una sola vez)

```bash
make ovmf
```

que equivale a:

```bash
build -p OvmfPkg/OvmfPkgX64.dsc -a X64 -t GCC -b DEBUG
```

Tarda unos minutos. Al terminar:

```bash
ls -lh ~/edk2/Build/OvmfX64/DEBUG_GCC/FV/OVMF*.fd
```

- `OVMF_CODE.fd`: el firmware. Lo montamos **solo lectura**.
- `OVMF_VARS.fd`: la plantilla de la NVRAM. Cada VM usa **una copia propia y
  escribible**; el `Makefile` la copia a `build/OVMF_VARS.fd`.

**¿Por qué DEBUG y no el OVMF que viene con la distro?** Porque el OVMF de DEBUG
escribe un log detallado por el puerto de E/S `0x402`, y en ese log está **la
dirección donde carga cada imagen**. Sin ese dato no podemos depurar (Parte 5).

### 1.7 Verificación

```bash
uname -m
gcc --version | head -1
qemu-system-x86_64 --version | head -1
gdb --version | head -1
git -C ~/edk2 describe --tags
ls -lh ~/edk2/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd
```

📸 **`P1-01-entorno`** — la salida de todos esos comandos.

---

## Parte 2 — La aplicación UEFI más chica posible

**Código:** [`HelloPkg/SimplestApp/`](HelloPkg/SimplestApp) y [`HelloPkg/HelloPkg.dsc`](HelloPkg/HelloPkg.dsc)
**Base:** lecciones 01 y 02 de UEFI-Lessons

### 2.1 Leer el código

`SimplestApp.c` es esto y nada más:

```c
EFI_STATUS
EFIAPI
UefiMain (
  IN EFI_HANDLE        ImageHandle,
  IN EFI_SYSTEM_TABLE  *SystemTable
  )
{
  return EFI_SUCCESS;
}
```

- **No hay `main`.** El nombre del punto de entrada lo decide el `.inf` con
  `ENTRY_POINT = UefiMain`.
- `UefiMain` **tampoco es el verdadero punto de entrada**. La biblioteca
  `UefiApplicationEntryPoint` define `_ModuleEntryPoint`, que primero corre los
  **constructores** de las bibliotecas, después llama a `UefiMain` y por último corre
  los destructores. Es el equivalente al `_start` de la libc del TP #2.
- Recibe dos cosas: el **handle de su propia imagen** y el **puntero a la System Table**.
- Devuelve un `EFI_STATUS` (`EFI_SUCCESS` vale 0).

Mirar también `SimplestApp.inf`:

| Campo | Valor | Para qué |
|---|---|---|
| `BASE_NAME` | `SimplestApp` | nombre del `.efi` |
| `FILE_GUID` | un GUID aleatorio | todo componente UEFI se identifica por un GUID |
| `MODULE_TYPE` | `UEFI_APPLICATION` | se carga, corre, devuelve y **se descarga** (un `UEFI_DRIVER` queda residente) |
| `ENTRY_POINT` | `UefiMain` | la función que llama `_ModuleEntryPoint` |
| `[LibraryClasses]` | `UefiApplicationEntryPoint` | la biblioteca que aporta `_ModuleEntryPoint` |

Y `HelloPkg.dsc`, que en `[LibraryClasses]` dice **qué implementación** usar para cada
clase de biblioteca (por ejemplo `DebugLib|MdePkg/Library/UefiDebugLibConOut/...`).

### 2.2 Compilar

```bash
cd ~/practico-3
make
```

El `Makefile` hace:

```bash
cd ~/edk2 && export PACKAGES_PATH=~/edk2:~/practico-3 && source edksetup.sh
build -p HelloPkg/HelloPkg.dsc -a X64 -t GCC -b NOOPT
cp ~/edk2/Build/HelloPkg/NOOPT_GCC/X64/*.efi disco/
```

- **`PACKAGES_PATH`** permite que `HelloPkg` viva en *nuestro* repo y no adentro de
  `edk2` (las lecciones lo copian adentro).
- **`-b NOOPT`** compila con `-O0` y símbolos de depuración. Es el equivalente al
  `-g -O0` del TP #2 y lo necesitamos para la Parte 5.
- Al final aparecen unos `Error 1 (ignored)`: son de un `cp *.pdb` que solo aplica en
  Windows. **No son errores reales**; lo que importa es que termine en `- Done -`.

📸 **`P2-01-compilacion`** — el final del `make`, con el `- Done -` y el `ls -l disco`.

### 2.3 Es un PE/COFF, no un ELF

```bash
file disco/SimplestApp.efi
objdump -h disco/SimplestApp.efi
```

Tiene que decir `PE32+ executable (EFI application) x86-64`. Aunque lo compiló `gcc`
en Linux, `GenFw` convirtió el ELF al formato que exige la especificación UEFI.

📸 **`P2-02-pe-coff`**

### 2.4 Arrancar OVMF y la UEFI Shell

```bash
make run
```

que equivale a:

```bash
qemu-system-x86_64 -m 256M \
  -drive if=pflash,format=raw,unit=0,readonly=on,file=~/edk2/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd \
  -drive if=pflash,format=raw,unit=1,file=build/OVMF_VARS.fd \
  -drive format=raw,file=fat:rw:disco \
  -net none -nographic \
  -global isa-debugcon.iobase=0x402 -debugcon file:build/debug.log
```

| Opción | Para qué |
|---|---|
| `-drive if=pflash,...` | el firmware en la flash emulada: código de solo lectura + NVRAM escribible |
| `-drive format=raw,file=fat:rw:disco` | expone la carpeta `disco/` como un disco FAT; en la Shell aparece como `FS0:` |
| `-nographic` | la consola de la VM sale en la terminal |
| `-net none` | sin red, así OVMF no pierde tiempo intentando arrancar por PXE |
| `-debugcon file:build/debug.log` | todo lo que OVMF escribe en el puerto `0x402` va a ese archivo |
| `-m 256M` | memoria **fija**: con la misma configuración, OVMF carga la app siempre en la misma dirección (Parte 5) |

Antes de arrancar QEMU, `make run` copia la UEFI Shell (que se compiló junto con OVMF,
en `~/edk2/Build/OvmfX64/DEBUG_GCC/X64/Shell.efi`) a **`disco/EFI/BOOT/BOOTX64.EFI`**.

**¿Por qué hace falta?** Es la fase **BDS** en acción. OVMF registra su Shell interna
como opción de arranque de categoría *aplicación* (`LOAD_OPTION_CATEGORY_APP`, en
`OvmfPkg/Library/PlatformBootManagerLib/BdsPlatform.c`), y BDS **solo arranca solas las
opciones de categoría *boot***. Así que BDS prueba nuestro disco, que es la única opción
*boot*, y busca un cargador en `\EFI\BOOT\BOOTX64.EFI`, que es la ruta por defecto que
define la especificación UEFI para medios removibles. Si no lo encuentra, se queda en:

```
BdsDxe: failed to load Boot0002 "UEFI QEMU HARDDISK QM00001 " from PciRoot(0x0)/Pci(0x1,0x1)/Ata(Primary,Master,0x0): Not Found
BdsDxe: No bootable option or device was found.
BdsDxe: Press any key to enter the Boot Manager Menu.
```

Con la Shell en esa ruta, el mismo `Boot0002` arranca bien. (Sin la copia, la salida es
apretar una tecla y elegir **EFI Internal Shell** en el Boot Manager.) En la lección
00 esto no pasaba porque en 2021 OVMF todavía arrancaba la Shell sin pasar por el menú.

Cuando aparece la Shell, `disco/startup.nsh` ejecuta `fs0:` solo, así que el prompt
queda en `FS0:\>`.

```
FS0:\> ls
FS0:\> map
```

`map` muestra el *Mapping table*: `FS0:` es nuestro disco, con su *device path*
(`PciRoot(0x0)/Pci(...)/...`). El device path es justamente cómo UEFI identifica un
dispositivo a través de sus handles y protocolos.

📸 **`P2-03-shell`** — el arranque de la Shell y la salida de `ls` y `map`.

### 2.5 Ejecutar la app mínima

```
FS0:\> SimplestApp.efi
FS0:\> echo %lasterror%
```

No imprime nada, que es lo esperado. `%lasterror%` es el `echo $?` de la UEFI Shell:
muestra el `EFI_STATUS` que devolvió el último comando. Tiene que ser `0x0`
(`EFI_SUCCESS`).

📸 **`P2-04-simplestapp`**

Para salir de QEMU: **`Ctrl+A` y después `X`**.

### 2.6 Las fases de arranque, en el log

Mientras OVMF arrancaba, todo su log fue a `build/debug.log`:

```bash
wc -l build/debug.log
grep -n -m1 "SecCoreStartupWithStack" build/debug.log      # SEC
grep -n -m1 "Loading PEIM at"         build/debug.log      # PEI
grep -n -m1 "DXE IPL Entry"           build/debug.log      # PEI → DXE
grep -n -m1 "Loading DXE CORE at"     build/debug.log      # DXE
grep -n -m1 "\[Bds\] Entry"           build/debug.log      # BDS
grep -n "SimplestApp.efi"             build/debug.log      # nuestra app
```

Los números de línea tienen que ir **en aumento**: es la secuencia SEC → PEI → DXE →
BDS de la Parte 0, y nuestra app aparece al final, cargada como cualquier otra imagen.
Esa última línea (`Loading driver at 0x... EntryPoint=0x... SimplestApp.efi`) es la
que vamos a usar en la Parte 5.

📸 **`P2-05-fases-en-el-log`**

---

## Parte 3 — Hello World "a mano"

**Código:** [`HelloPkg/HelloWorld/HelloWorld.c`](HelloPkg/HelloWorld/HelloWorld.c)
**Base:** lección 03

### 3.1 Leer el código

```c
EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL  *ConOut;
EFI_STATUS                       Status;

ConOut = SystemTable->ConOut;
Status = ConOut->OutputString (ConOut, L"Hello World!\r\n");
return Status;
```

- No hay `printf` porque **no hay libc**. Lo único que tenemos es lo que nos dio el
  firmware: la System Table.
- `SystemTable->ConOut` es un puntero al protocolo `EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL` de
  la consola.
- `ConOut->OutputString` es **un puntero a función** que está adentro del firmware.
  Lo llamamos pasándole el propio protocolo (`This`) y la cadena.
- `L"..."`: las cadenas UEFI son **`CHAR16`** (UTF-16), dos bytes por carácter. EDK2
  compila con `-fshort-wchar` para que `L""` sea de 16 bits y no de 32, como en Linux.
- `\r\n`: la consola UEFI necesita el retorno de carro explícito.

Las dos estructuras, como las define la especificación:

```c
typedef struct {
  EFI_TABLE_HEADER                 Hdr;               // 0x00  (24 bytes)
  CHAR16                           *FirmwareVendor;   // 0x18
  UINT32                           FirmwareRevision;  // 0x20  (+4 de relleno)
  EFI_HANDLE                       ConsoleInHandle;   // 0x28
  EFI_SIMPLE_TEXT_INPUT_PROTOCOL   *ConIn;            // 0x30
  EFI_HANDLE                       ConsoleOutHandle;  // 0x38
  EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL  *ConOut;           // 0x40  ← este
  ...
} EFI_SYSTEM_TABLE;

typedef struct _EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL {
  EFI_TEXT_RESET                   Reset;             // 0x00
  EFI_TEXT_STRING                  OutputString;      // 0x08  ← este
  ...
} EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL;
```

Hacé las cuentas de los offsets vos mismo: los vamos a encontrar en el desensamblado.

### 3.2 Ejecutar

```bash
make            # si ya compilaste en la Parte 2, no cambia nada
make run
```

```
FS0:\> HelloWorld.efi
Hello World!
FS0:\> echo %lasterror%
```

📸 **`P3-01-hello-world`**

### 3.3 Lo que generó el compilador

```bash
objdump -d -S --no-show-raw-insn ~/edk2/Build/HelloPkg/NOOPT_GCC/X64/HelloWorld.debug \
  | awk '/<UefiMain>:/,/ret/'
```

El `.debug` es el ELF con los símbolos DWARF, antes de convertirlo a PE. Lo que hay
que encontrar:

```
sub    $0x30,%rsp              # 0x20 de shadow space para lo que llamemos + 0x10 de locales
mov    %rcx,0x10(%rbp)         # ImageHandle llegó en RCX  → lo guarda en SU shadow space
mov    %rdx,0x18(%rbp)         # SystemTable llegó en RDX  → ídem
mov    0x18(%rbp),%rax         # rax = SystemTable
mov    0x40(%rax),%rax         # rax = SystemTable->ConOut          (offset 0x40)
mov    %rax,-0x10(%rbp)        # ConOut = ...
mov    0x8(%rax),%r8           # r8  = ConOut->OutputString         (offset 0x08)
lea    ...(%rip),%rdx          # rdx = L"Hello World!\r\n"          (2do argumento)
mov    %rax,%rcx               # rcx = ConOut                       (1er argumento: This)
call   *%r8                    # llamada indirecta, a una función del firmware
```

Esto ya confirma tres cosas de la Parte 0, **antes de ejecutar nada**:

1. Los argumentos entran por **`rcx` y `rdx`** (Microsoft x64), no por `rdi` y `rsi`.
2. La función guarda sus argumentos en `0x10(%rbp)` y `0x18(%rbp)`, es decir **arriba
   de la dirección de retorno**. Esa memoria la reservó el que la llamó: es el
   **shadow space**.
3. Los offsets `0x40` y `0x8` son los de las estructuras de 3.1.

📸 **`P3-02-desensamblado`**

### 3.4 Pregunta

**¿Por qué `call *%r8` y no `call OutputString`?** Porque `OutputString` no está en
nuestro binario: la implementa el driver de consola del firmware, y la única forma de
llegar a ella es el puntero que está en el protocolo. Los protocolos son, en esencia,
**tablas de punteros a función**: en tiempo de compilación nuestra app no conoce
ninguna dirección del firmware.

---

## Parte 4 — El mismo Hello World con las bibliotecas de EDK2

**Código:** [`HelloPkg/HelloPrint/HelloPrint.c`](HelloPkg/HelloPrint/HelloPrint.c)
**Base:** lección 04

### 4.1 Leer el código

```c
gST->ConOut->OutputString (gST->ConOut, L"Hello World! (via gST)\r\n");
Print (L"Hello World! (via Print)\r\n");
Print (L"Firmware: %s, revision 0x%08x\r\n", gST->FirmwareVendor, gST->FirmwareRevision);
Print (L"UEFI %d.%d\r\n", gST->Hdr.Revision >> 16, gST->Hdr.Revision & 0xFFFF);
```

- **`gST`** es una variable global que define `UefiBootServicesTableLib`. ¿Quién la
  carga? El **constructor** de esa biblioteca (`UefiBootServicesTableLibConstructor`),
  que copia `SystemTable` en `gST` y `SystemTable->BootServices` en `gBS`.
  `_ModuleEntryPoint` lo ejecuta **antes** de llamar a `UefiMain`.
- **`Print`** sale de `UefiLib`: formatea como `printf` y termina llamando a
  `gST->ConOut->OutputString`. `%s` espera una cadena `CHAR16`.
- Para poder usarlas hubo que agregar `UefiBootServicesTableLib` y `UefiLib` en el
  `.inf`, y sus instancias (más las que ellas necesitan: `MemoryAllocationLib`,
  `DevicePathLib`, etc.) en el `.dsc`.

### 4.2 Ejecutar

```bash
make run
```

```
FS0:\> HelloPrint.efi
```

Tiene que imprimir las cuatro líneas. `FirmwareVendor` es `EDK II`, y la versión de
UEFI sale de la cabecera de la System Table (la revisión 2.70 se codifica como
`(2 << 16) | 70`).

📸 **`P4-01-hello-print`**

### 4.3 El precio de las bibliotecas

```bash
ls -l disco/*.efi
make TARGET=RELEASE app
ls -l ~/edk2/Build/HelloPkg/RELEASE_GCC/X64/*.efi
make            # volver a dejar los .efi de NOOPT en disco/ para la Parte 5
```

Comparar `HelloWorld.efi` con `HelloPrint.efi`, y NOOPT (`-O0`) con RELEASE (`-Os` +
LTO).

📸 **`P4-02-tamanos`**

---

## Parte 5 — Depurar el Hello World con GDB

**Base:** lección 42 de UEFI-Lessons

### 5.1 El problema

En el TP #2 hacíamos `gdb ./programa` y listo: el binario se cargaba siempre en la
misma dirección (`-no-pie`) y GDB lo sabía. Acá hay tres diferencias:

1. GDB no ejecuta el programa: se conecta a **QEMU**, que expone un *stub* de GDB en
   el puerto TCP 1234 (opción `-s`). Lo que depuramos es la **máquina virtual entera**.
2. El `.efi` es un **PE/COFF reubicable**. El firmware elige en tiempo de ejecución
   dónde ponerlo (`LoadImage`), así que la dirección de `UefiMain` **no se conoce hasta
   que la app se carga**.
3. El `.efi` no tiene símbolos. Los símbolos están en el `.debug` (ELF + DWARF).

La solución es decirle a GDB: "los símbolos de este `.debug` corresponden a código que
está cargado a partir de *tal* dirección". Eso es `add-symbol-file`.

### 5.2 ¿Dónde quedó cargada la app?

Si en la Parte 3 ya corriste `HelloWorld.efi` con `make run`, el dato ya está en el
log. Si no, corré `make run`, ejecutá `HelloWorld.efi` y salí con `Ctrl+A X`.

```bash
grep "Loading driver at" build/debug.log | tail -3
```

```
Loading driver at 0x0000XXXXXXX EntryPoint=0x0000YYYYYYY HelloWorld.efi
```

La imprime el DXE Core (`MdeModulePkg/Core/Dxe/Image/Image.c`) cada vez que carga una
imagen. Es la **base de carga**: dónde quedó el byte 0 del PE en memoria.

📸 **`P5-01-direccion-de-carga`**

### 5.3 ¿Dónde está cada sección dentro de la imagen?

```bash
objdump -h disco/HelloWorld.efi
objdump -h ~/edk2/Build/HelloPkg/NOOPT_GCC/X64/HelloWorld.debug | head -12
```

La columna **VMA** del `.efi` es el **RVA** de cada sección, es decir su distancia a la
base de carga. Hay que fijarse en dos cosas:

- `.text` empieza en **`0x1000`** y `.data` en **`0x5000`**. En la lección 42 era
  `0x240`: las versiones nuevas de EDK2 alinean las secciones a 4 KiB. **No copies los
  números de la lección**: sacalos de tu `objdump`.
- En el `.debug`, `.text` y `.data` tienen **las mismas VMA** que en el `.efi`. El
  linker script de EDK2 (`BaseTools/Scripts/GccBase.lds`) lo garantiza a propósito,
  para que los símbolos del ELF sirvan para el PE.

Entonces:

```
dirección de .text en memoria = base + RVA(.text)
dirección de .data en memoria = base + RVA(.data)
```

📸 **`P5-02-secciones`**

### 5.4 Calcular las direcciones

```bash
make simbolos
```

[`scripts/simbolos.sh`](scripts/simbolos.sh) hace la cuenta de 5.3 con la última línea
del log y escribe `build/simbolos.gdb`, con el `add-symbol-file` listo. Además hace un
**control cruzado**: `EntryPoint − base` tiene que dar la dirección de
`_ModuleEntryPoint` en el `.debug` (por ejemplo `0x3658`). Si coinciden, la base es
correcta. Y confirma lo de la Parte 2: el verdadero punto de entrada es
`_ModuleEntryPoint`, no `UefiMain`.

Verificá la cuenta a mano con los números de 5.2 y 5.3.

📸 **`P5-03-simbolos`**

### 5.5 Conectar GDB y llegar a `UefiMain`

Hacen falta **dos terminales**, las dos en `~/practico-3`.

**Terminal 1** — QEMU esperando a GDB:

```bash
make debug          # es 'make run' + la opción -s (stub de GDB en tcp::1234)
```

**Terminal 2** — GDB:

```bash
make gdb            # gdb -x scripts/conectar.gdb
```

[`scripts/conectar.gdb`](scripts/conectar.gdb) carga `build/simbolos.gdb`, pone
`break UefiMain` y hace `target remote :1234`. En ese momento **QEMU se congela**:
GDB lo detuvo donde estuviera (en algún lugar del firmware, sin símbolos).

```
>>> continue
```

En la **terminal 1**, en la Shell:

```
FS0:\> HelloWorld.efi
```

GDB tiene que frenar en `UefiMain`, con el dashboard mostrando la línea
`ConOut = SystemTable->ConOut;`.

📸 **`P5-04-breakpoint-uefimain`**

> **Si no frena:** la app se cargó en otra dirección. Revisá con
> `grep "Loading driver" build/debug.log | tail -1` (el log se reescribe en cada
> arranque) y compará con `build/simbolos.gdb`. Si cambió, salí de los dos lados,
> `make simbolos` y repetí. Para que no cambie, hay que arrancar QEMU siempre con las
> mismas opciones (por eso `-m 256M` está fijo en el `Makefile`).
>
> **Detalle:** el breakpoint se pone *antes* de que la app esté en memoria. Funciona
> porque QEMU, en modo emulación (TCG, sin `-enable-kvm`), implementa los
> breakpoints sin escribir en la memoria del invitado. Con KVM habría que usar
> `hbreak UefiMain`.

### 5.6 Los argumentos llegaron por RCX y RDX

```
>>> info registers rcx rdx
>>> p ImageHandle
>>> p SystemTable
>>> p/x SystemTable->Hdr.Signature
>>> x/8cb SystemTable
>>> x/sh SystemTable->FirmwareVendor
>>> p/x SystemTable->FirmwareRevision
>>> p SystemTable->ConOut
```

- `rcx` tiene que ser igual a `ImageHandle`, y `rdx` igual a `SystemTable`.
- La firma de la System Table es `0x5453595320494249`, que en memoria se lee
  `I B I   S Y S T` (`EFI_SYSTEM_TABLE_SIGNATURE`). Así sabemos que el puntero
  realmente apunta a una System Table.
- `FirmwareVendor` es una cadena `CHAR16`: por eso `x/sh` (*s*tring de *h*alfwords)
  y no `x/s`. Tiene que mostrar `u"EDK II"`.

📸 **`P5-05-system-table`**

### 5.7 El shadow space y quién nos llamó

```
>>> x/2gx $rbp+0x10
>>> x/gx $rbp+8
>>> info symbol *(long *)($rbp+8)
>>> bt
```

- `x/2gx $rbp+0x10`: los mismos valores que `rcx` y `rdx`. El prólogo los copió al
  **shadow space**, los 32 bytes que el llamador reservó arriba de la dirección de
  retorno. En System V esto no existe: allá los argumentos se guardaban **debajo** de
  `rbp`, como locales.
- `$rbp+8`: la **dirección de retorno**, como en el TP #2. `info symbol` muestra que
  vuelve a `ProcessModuleEntryPointList`: es la función de `AutoGen.c` que llama a
  `UefiMain`.
- `bt`: `UefiMain` ← `ProcessModuleEntryPointList` ← `_ModuleEntryPoint` ← (el
  firmware, sin símbolos). Es la cadena que describimos en la Parte 2.

Para ver el lado del llamador:

```
>>> disassemble /s ProcessModuleEntryPointList
```

Hace `sub $0x20,%rsp` antes del `call UefiMain`: reserva **exactamente 32 bytes**, que
son los cuatro slots del shadow space. No tiene variables locales, así que no reserva
nada más.

📸 **`P5-06-shadow-space-y-backtrace`**

### 5.8 Justo antes de llamar a `OutputString`

```
>>> next
>>> p ConOut
>>> p *ConOut
>>> disassemble /s UefiMain
```

En el desensamblado, ubicá la dirección del `call *%r8` y avanzá hasta ahí:

```
>>> until *0x...           # la dirección del call *%r8
>>> info registers rcx rdx r8 rsp
>>> p $rcx == (long)ConOut
>>> p $r8 == (long)ConOut->OutputString
>>> x/sh $rdx
>>> x/4gx $rsp
```

(`stepi` repetido hasta llegar al `call` también sirve.)

- `rcx` = `ConOut` → primer argumento (`This`).
- `rdx` → `u"Hello World!\r\n"` → segundo argumento.
- `r8` = `ConOut->OutputString`, una dirección **del firmware**, fuera de nuestra
  imagen (comparala con la base de 5.2).
- `x/4gx $rsp`: los 32 bytes de shadow space que **nosotros** reservamos (con el
  `sub $0x30,%rsp`) para la función que vamos a llamar.

📸 **`P5-07-antes-del-call`**

### 5.9 La llamada y el retorno

```
>>> nexti
>>> p/x $rax
```

`nexti` ejecuta el `call` entero: en la **terminal 1** aparece `Hello World!`. En `rax`
vuelve el `EFI_STATUS` de `OutputString`: `0x0`, o sea `EFI_SUCCESS`.

📸 **`P5-08-retorno`** — GDB con `rax = 0x0` y QEMU mostrando el `Hello World!`.

```
>>> continue
```

La Shell vuelve al prompt. Para terminar: `quit` en GDB y `Ctrl+A X` en QEMU.

### 5.10 Machete de GDB para esta parte

| Comando | Qué hace |
|---|---|
| `target remote :1234` | conectarse al stub de QEMU |
| `add-symbol-file F.debug ADDR -s .data ADDR2` | cargar símbolos reubicados |
| `break UefiMain` / `hbreak UefiMain` | breakpoint de software / de hardware |
| `continue`, `next`, `step`, `nexti`, `stepi`, `finish` | igual que en el TP #2 |
| `until *ADDR` | correr hasta una dirección |
| `x/sh ADDR` | cadena `CHAR16` |
| `x/Ngx ADDR` | N palabras de 8 bytes en hexa |
| `info symbol ADDR` | qué símbolo hay en esa dirección |
| `disassemble /s FUNC` | desensamblado intercalado con el código fuente |

---

## Parte 6 — El Hello World en hardware real (bare metal)

Hasta acá todo corrió sobre **OVMF dentro de QEMU**, un firmware emulado. En esta parte
llevamos **los mismos `.efi`, sin recompilar**, a una notebook real: arrancamos la UEFI
Shell desde un pendrive y ejecutamos la app antes de que exista ningún sistema
operativo.

Qué se demuestra:

- **El binario es portable entre firmwares.** La app no toca hardware: solo usa la
  System Table y sus protocolos. Por eso corre igual en OVMF (EDK II) que en el firmware
  del fabricante (en nuestro caso, **Insyde**, en una Lenovo Legion).
- **La ruta por defecto de medios removibles.** Es el mismo mecanismo de la Parte 2.4:
  BDS busca `\EFI\BOOT\BOOTX64.EFI` en el pendrive sin necesitar ninguna entrada
  `Boot####` en la NVRAM.
- **Secure Boot.** El firmware solo ejecuta imágenes PE/COFF firmadas con una clave de su
  base `db`. Nuestra Shell no está firmada, así que hay que desactivarlo (pregunta 7 de
  la Parte 7).

> **Riesgo para la PC:** prácticamente nulo. La app solo escribe en la consola, y la
> configuración de Secure Boot se vuelve atrás al final. Lo único a cuidar: BitLocker
> (6.4) y no usar en la Shell comandos que no estén en esta guía.

### 6.1 Qué hace falta

- **Linux con EDK2 y BaseTools**: Partes 1.1 a 1.5. Puede ser una máquina virtual (lo
  hicimos en una VM de VirtualBox). **No** hace falta `make ovmf`.
- Un **pendrive que se pueda borrar entero**.
- Una **PC con firmware UEFI** (cualquier notebook de los últimos años). No se toca su
  disco.

```bash
sudo apt install -y dosfstools     # trae mkfs.vfat, para formatear el pendrive
```

### 6.2 Compilar las apps y la Shell

Las apps son las mismas de las Partes 2 a 4:

```bash
cd ~/practico-3
make app
ls ~/edk2/Build/HelloPkg/NOOPT_GCC/X64/*.efi
```

La Shell **no** la sacamos de OVMF ni del link `UDK2018` del documento de la cátedra
(es de 2018 y en hardware nuevo puede fallar). La compilamos desde `ShellPkg` en
**`RELEASE`**:

```bash
cd ~/edk2
source edksetup.sh
build -p ShellPkg/ShellPkg.dsc -a X64 -t GCC -b RELEASE
find ~/edk2/Build/Shell -name Shell.efi
```

`find` devuelve cuatro rutas. La que usamos es:

```
~/edk2/Build/Shell/RELEASE_GCC/X64/ShellPkg/Application/Shell/Shell/OUTPUT/Shell.efi
```

- Las carpetas `DEBUG/` son copias intermedias con información de depuración.
- La carpeta con el GUID `EA4BB293-...` es una variante de la Shell que `ShellPkg.dsc`
  compila como ejemplo.

**¿Por qué RELEASE y no la Shell de OVMF?** La de OVMF es DEBUG y escribe su log en el
puerto de E/S `0x402`. En QEMU ahí está el debugcon (Parte 2.6), pero en una PC real no
sabemos qué hay en ese puerto. La RELEASE no escribe nada.

### 6.3 Preparar el pendrive

**1. Pasarle el pendrive a la VM.** En VirtualBox: *Dispositivos → USB* y tildar el
pendrive. Aparece con un nombre genérico (en nuestro caso, *USB Disk 20*). Desde ese
momento Windows deja de verlo.

**2. Identificarlo.** Este es el paso delicado: si te equivocás de disco, formateás el
de la VM.

```bash
lsblk
```

El pendrive es el disco con su tamaño y con `RM = 1` (removible). En nuestro caso,
`sdb` de 29,3 GB con la partición `sdb1`. `sda` es el disco de la VM y **no se toca**.
En lo que sigue, reemplazar `sdb1` por la partición que corresponda.

**3. Formatear en FAT32.** Es el sistema de archivos que todo firmware UEFI está obligado
a leer. Ubuntu monta el pendrive solo, así que primero hay que desmontarlo:

```bash
sudo umount /dev/sdb1
sudo mkfs.vfat -F 32 -n UEFI /dev/sdb1
```

Si te olvidás del `umount`, `mkfs.vfat` se niega con
`/dev/sdb1 contains a mounted filesystem`.

> **Ojo al pegar comandos:** van **de a uno**. Si se pegan varias líneas juntas, la
> terminal puede tomarlas como un único bloque y ejecutar solo la primera.

**4. Copiar los archivos con la estructura de la ruta por defecto:**

```bash
sudo mount /dev/sdb1 /mnt
sudo mkdir -p /mnt/EFI/BOOT
sudo cp ~/edk2/Build/Shell/RELEASE_GCC/X64/ShellPkg/Application/Shell/Shell/OUTPUT/Shell.efi /mnt/EFI/BOOT/BOOTX64.EFI
sudo cp ~/edk2/Build/HelloPkg/NOOPT_GCC/X64/{HelloWorld,HelloPrint,SimplestApp}.efi /mnt/
ls -R /mnt
```

Tiene que quedar así:

```
/mnt
├── EFI/BOOT/BOOTX64.EFI     ← la Shell, renombrada
├── HelloPrint.efi
├── HelloWorld.efi
└── SimplestApp.efi
```

**5. Desmontar y devolver el pendrive:**

```bash
sync
sudo umount /mnt
```

Destildarlo en *Dispositivos → USB* y recién ahí sacarlo.

### 6.4 Antes de tocar el firmware: BitLocker

Si el disco de Windows está cifrado con BitLocker, cambiar Secure Boot hace que, al
volver, Windows pida la **clave de recuperación**. Hay que fijarse antes, en una
terminal de Windows **como administrador**:

```
manage-bde -status C:
```

- Si dice `Estado de protección: Protección desactivada`, se puede seguir.
- Si dice `Protección activada`, primero hay que anotar la clave (<https://aka.ms/myrecoverykey>)
  y suspender la protección: *Panel de control → Cifrado de unidad BitLocker →
  Suspender protección*.

### 6.5 Desactivar Secure Boot

Para entrar al setup del firmware sin adivinar la tecla, desde la terminal de
administrador:

```
shutdown /r /fw /t 0
```

La PC reinicia directo en el setup. En la Lenovo Legion es gráfico y acepta mouse:

1. **More Settings → Security → Secure Boot → Disabled.**
2. No tocar *Reset to Setup Mode* ni *Restore Factory Keys*: borran o reemplazan las
   claves `PK`, `KEK`, `db` y `dbx`.
3. **F10 → Yes** para guardar y salir.

En otras marcas la opción suele estar en *Security*, *Boot* o *Authentication*. Si
aparece gris, a veces hay que definir primero una *Supervisor Password*. El modo de
arranque tiene que ser **UEFI** (no *Legacy* ni *CSM*).

> **Detalle:** en la misma pantalla, *Secure Boot Status* puede seguir
> diciendo *Enabled*. Ese campo muestra con qué configuración arrancó la máquina esta
> vez. El cambio se aplica recién después de guardar y reiniciar.

**¿Por qué hay que desactivarlo?** Antes de ejecutar una imagen, el firmware verifica su
firma Authenticode contra los certificados de la variable `db` (Microsoft y el
fabricante) y contra la lista negra `dbx`. Nuestra Shell y nuestras apps no tienen
firma, así que con Secure Boot activo el firmware se niega a ejecutarlas.

### 6.6 Arrancar desde el pendrive

Con el pendrive enchufado, al encender se aprieta repetidamente la tecla del **boot
menu**. En Lenovo es **F12** (o **Fn+F12**). En otras marcas suele ser F11, F9, F8 o Esc.

Se elige **EFI USB Device**. El menú muestra las opciones de arranque que armó BDS:
*Windows Boot Manager* (una entrada `Boot####` de la NVRAM), el pendrive (por la ruta
por defecto `\EFI\BOOT\BOOTX64.EFI`) y PXE por red.

### 6.7 En la UEFI Shell

Al arrancar, la Shell imprime su versión, la del firmware y la tabla de mapeo. En el
encabezado ya se ve **`UEFI v2.80 (INSYDE Corp., 0x77066035)`**: es el firmware real, no
OVMF.

**1. Encontrar el pendrive:**

```
Shell> map -r
```

Hay dos sistemas de archivos y se distinguen por su *device path*:

| Mapeo | Device path | Qué es |
|---|---|---|
| `FS0:` | `PciRoot(0x0)/Pci(0x14,0x0)/USB(...)/HD(1,MBR,...)` | el **pendrive**: USB y tabla MBR, como lo formateamos |
| `FS1:` | `PciRoot(0x0)/Pci(0x6,0x0)/.../NVMe(...)/HD(1,GPT,...)` | la **partición EFI de Windows**, en el disco NVMe |

Los `BLKx:` son dispositivos de bloque sin sistema de archivos que la Shell pueda leer
(el disco entero y las otras particiones de Windows).

> **No entrar a `FS1:`**. Desde la Shell se puede modificar el disco de Windows sin
> confirmación. Tampoco usar `rm`, `del`, `format`, `mv`, `edit`, `bcfg` ni `setvar`.

**2. Ir al pendrive y listar:**

```
Shell> FS0:
FS0:\> ls
```

**3. Ver el firmware y ejecutar el Hello World:**

```
FS0:\> ver
UEFI Interactive Shell v2.2
EDK II
UEFI v2.80 (INSYDE Corp., 0x77066035)
FS0:\> HelloWorld.efi
Hello World!
FS0:\> echo %lasterror%
0x0
```

- `EDK II` es quien compiló **la Shell**. La línea siguiente es **el firmware**: Insyde,
  que implementa UEFI 2.80. En QEMU, en cambio, el firmware también era EDK II (OVMF).
- `0x0` es `EFI_SUCCESS`, el valor que devolvió `UefiMain`.

**4. El `HelloPrint`, que además lee la System Table:**

```
FS0:\> HelloPrint.efi
Hello World! (via gST)
Hello World! (via Print)
Firmware: INSYDE Corp., revision 0x77066035
UEFI 2.80
```

Los datos del firmware salen de `gST->FirmwareVendor`, `gST->FirmwareRevision` y
`gST->Hdr.Revision`. Es la misma System Table que en QEMU, completada ahora por otro
firmware: el mismo binario la lee sin cambios.

### 6.8 Dejar todo como estaba

1. En la Shell, `reset` (reinicia) y sacar el pendrive.
2. Entrar al setup (**F2** al encender o `shutdown /r /fw /t 0` desde Windows) y volver
   a poner **Secure Boot → Enabled**. **F10 → Yes.**
3. Si se suspendió BitLocker, reanudarlo: *Panel de control → Cifrado de unidad
   BitLocker → Reanudar protección*.

### 6.9 Si algo no anda

| Síntoma | Causa | Solución |
|---|---|---|
| `mkfs.vfat: ... contains a mounted filesystem` | Ubuntu montó el pendrive solo | `sudo umount /dev/sdb1` antes de formatear |
| `cp: no se puede efectuar 'stat' sobre '.../X64/Shell.efi'` | La Shell de `ShellPkg` no queda directo en `X64/` | Usar la ruta `.../Shell/Shell/OUTPUT/Shell.efi` (6.2) |
| El pendrive no aparece en el boot menu | No es FAT32 o falta `\EFI\BOOT\BOOTX64.EFI` | Revisar 6.3 |
| *Secure Boot Violation* / el pendrive no arranca | Secure Boot sigue activo | 6.5 |
| `'HelloWorld.efi' is not recognized` | Estás en el `FSx:` equivocado | `map -r` y buscar el que tenga `USB(...)` |

### 6.10 Preguntas

Las respuestas van en `PROCEDIMIENTO.md`, junto con las de la Parte 7:

1. ¿Por qué el mismo `HelloWorld.efi` corre sin cambios en OVMF y en el firmware de
   Insyde? (Pista: PE/COFF reubicable, System Table, protocolos, convención Microsoft
   x64.)
2. ¿Cómo encontró BDS la Shell en el pendrive si no había ninguna entrada `Boot####`
   para él? Relacionalo con la Parte 2.4.
3. ¿Por qué el firmware rechazaría nuestra Shell con Secure Boot activo, y qué
   variables de la NVRAM intervienen en esa decisión?
4. En `map -r`, ¿cómo se distingue el pendrive del disco de Windows solo mirando el
   *device path*?

---

## Parte 7 — Preguntas para cerrar

Las respuestas van al final de `PROCEDIMIENTO.md`, justificadas con lo que vimos:

1. ¿Qué diferencia hay entre UEFI y PI? ¿En qué fase corre nuestra app y por qué todavía
   puede usar `ConOut`?
2. ¿Por qué el `.efi` es un PE/COFF y qué consecuencia tiene eso para depurar?
3. ¿Qué convención de llamada usa UEFI en x64? ¿Qué cambia respecto de System V?
   Mostralo con los registros y la pila de la Parte 5.
4. ¿Qué es un protocolo y por qué la llamada a `OutputString` es indirecta?
5. ¿Quién llama a `UefiMain`? ¿Qué pasa antes de que se ejecute?
6. ¿Para qué sirve cada archivo del paquete (`.inf`, `.dsc`) y qué hace `PACKAGES_PATH`?
7. Seguridad: ¿qué implicaría que una app como esta pudiera ejecutarse sin
   **Secure Boot**? (Pista: corre con los privilegios del firmware, antes que el SO y
   sin ninguna de sus protecciones.)
