# Procedimiento

Este documento registra paso a paso lo que fuimos haciendo, con las capturas que verifican cada resultado. Está pensado para leerse en orden.

Las partes siguen la numeración de [`GUIA.md`](GUIA.md), donde están la teoría y los comandos. Los requisitos del trabajo están en [`CONSIGNA.md`](CONSIGNA.md).

| Parte | Tema | Código |
|---|---|---|
| 1 | Preparar el entorno | — |
| 2 | La app mínima | `HelloPkg/SimplestApp/` |
| 3 | Hello World con la System Table | `HelloPkg/HelloWorld/` |
| 4 | Hello World con las bibliotecas de EDK2 | `HelloPkg/HelloPrint/` |
| 5 | Depuración con GDB | `HelloPkg/HelloWorld/` + `scripts/` |

Todas las sesiones de depuración se hicieron con **gdb-dashboard**.

> **Para completar:** cada `> ✏️` marca un dato que depende de la corrida (direcciones,
> versiones) y que hay que copiar de la captura. Una vez completado, se borra el marcador.

---

# Parte 1: preparar el entorno

## 1.1 Herramientas y EDK2

```bash
sudo apt install build-essential git uuid-dev iasl nasm python3 qemu-system-x86 gdb
git clone --branch edk2-stable202608 --depth 1 https://github.com/tianocore/edk2 ~/edk2
cd ~/edk2 && git submodule update --init --depth 1
make -C ~/edk2/BaseTools -j$(nproc)
```

## 1.2 OVMF en DEBUG

```bash
cd ~/practico-3
make ovmf
```

Compilamos OVMF nosotros en DEBUG, en lugar de usar el de la distro, porque esa versión escribe por el puerto `0x402` un log con la dirección donde carga cada imagen, y ese dato es imprescindible para la Parte 5.

## 1.3 Verificación

![Entorno](capturas/P1-01-entorno.jpg)

> ✏️ Arquitectura, versiones de gcc, QEMU y GDB, tag de EDK2 y tamaño de `OVMF_CODE.fd`.

---

# Parte 2: la app mínima

## 2.1 Compilar

```bash
make
```

![Compilación](capturas/P2-01-compilacion.jpg)

`build` compila `HelloPkg.dsc` con el target `NOOPT` (`-O0` + símbolos) y el `Makefile` copia los tres `.efi` a `disco/`. Los `Error 1 (ignored)` corresponden a la copia de archivos `.pdb`, que solo existen en Windows: no afectan el resultado.

## 2.2 El ejecutable es PE/COFF

![PE/COFF](capturas/P2-02-pe-coff.jpg)

`file` reconoce `PE32+ executable (EFI application) x86-64`. Aunque lo compiló gcc en Linux, `GenFw` convirtió el ELF al formato PE/COFF que exige la especificación UEFI. Ese formato trae información de reubicación, y por eso el firmware puede cargar la imagen en cualquier dirección.

## 2.3 La UEFI Shell

```bash
make run
```

![Shell](capturas/P2-03-shell.jpg)

La primera vez, OVMF se quedó en `BdsDxe: No bootable option or device was found`. La Shell interna está registrada como opción de categoría *aplicación* y BDS solo arranca solas las de categoría *boot*. Por eso probó nuestro disco, y el disco no tenía un cargador en la ruta por defecto `\EFI\BOOT\BOOTX64.EFI`. Lo resolvimos con `make run`, que copia `Shell.efi` a esa ruta. Así BDS arrancó la UEFI Shell desde nuestro disco, y `disco/` aparece como `FS0:`. El *device path* que muestra `map` es la forma en que UEFI identifica el dispositivo.

## 2.4 Ejecutar `SimplestApp.efi`

![SimplestApp](capturas/P2-04-simplestapp.jpg)

No imprime nada, que es lo esperado. `echo %lasterror%` (el `echo $?` de la Shell) muestra `0x0`, o sea el `EFI_SUCCESS` que devolvió `UefiMain`.

## 2.5 Las fases de arranque en el log

![Fases](capturas/P2-05-fases-en-el-log.jpg)

> ✏️ Números de línea de cada marcador.

Los marcadores aparecen en orden creciente: `SecCoreStartupWithStack` (SEC), `Loading PEIM` (PEI), `DXE IPL Entry` y `Loading DXE CORE` (paso a DXE), `[Bds] Entry` (BDS) y, al final, `Loading driver at ... SimplestApp.efi`. Es la secuencia SEC → PEI → DXE → BDS del documento de la cátedra, y muestra que nuestra app se carga con el mismo mecanismo que cualquier driver.

---

# Parte 3: Hello World con la System Table

## 3.1 Ejecución

![Hello World](capturas/P3-01-hello-world.jpg)

## 3.2 El desensamblado de `UefiMain`

![Desensamblado](capturas/P3-02-desensamblado.jpg)

| Instrucción | Qué muestra |
|---|---|
| `mov %rcx,0x10(%rbp)` / `mov %rdx,0x18(%rbp)` | `ImageHandle` y `SystemTable` llegan en **`rcx` y `rdx`** (Microsoft x64) y se guardan en el shadow space, arriba de la dirección de retorno |
| `mov 0x40(%rax),%rax` | `SystemTable->ConOut`: `ConOut` está en el offset `0x40` de `EFI_SYSTEM_TABLE` |
| `mov 0x8(%rax),%r8` | `ConOut->OutputString`: segundo puntero del protocolo, offset `0x8` |
| `mov %rax,%rcx` + `lea ...,%rdx` | primer argumento `This`, segundo argumento la cadena |
| `call *%r8` | **llamada indirecta**: la función está en el firmware y solo la conocemos por el puntero del protocolo |
| `sub $0x30,%rsp` | 32 bytes de shadow space para la función que llamamos + 16 de variables locales |

---

# Parte 4: Hello World con las bibliotecas de EDK2

## 4.1 Ejecución

![HelloPrint](capturas/P4-01-hello-print.jpg)

`gST` existe y vale lo mismo que `SystemTable` porque el constructor de `UefiBootServicesTableLib` la cargó antes de que se ejecutara `UefiMain`. `Print` formatea la cadena y termina llamando al mismo `OutputString`.

> ✏️ `FirmwareVendor`, `FirmwareRevision` y versión de UEFI que se imprimieron.

## 4.2 Tamaños

![Tamaños](capturas/P4-02-tamanos.jpg)

> ✏️ Tabla con los tamaños de `HelloWorld.efi` y `HelloPrint.efi` en NOOPT y en RELEASE.

---

# Parte 5: depuración con GDB

## 5.1 Dirección de carga

![Dirección de carga](capturas/P5-01-direccion-de-carga.jpg)

> ✏️ `Loading driver at 0x... EntryPoint=0x... HelloWorld.efi` → base = `0x...`

## 5.2 Secciones

![Secciones](capturas/P5-02-secciones.jpg)

`.text` empieza en `0x1000` y `.data` en `0x5000`, con las mismas VMA en el `.efi` y en el `.debug`.

## 5.3 Cálculo de las direcciones

![Símbolos](capturas/P5-03-simbolos.jpg)

> ✏️ Cuenta a mano: `base + 0x1000 = ...`, `base + 0x5000 = ...`, y control cruzado `EntryPoint − base = 0x3658 = _ModuleEntryPoint`.

## 5.4 Breakpoint en `UefiMain`

![Breakpoint](capturas/P5-04-breakpoint-uefimain.jpg)

## 5.5 Los argumentos y la System Table

![System Table](capturas/P5-05-system-table.jpg)

> ✏️ Valores de `rcx`/`ImageHandle` y `rdx`/`SystemTable`.

La firma `0x5453595320494249` (`IBI SYST`) confirma que `rdx` apunta a una `EFI_SYSTEM_TABLE`, y `FirmwareVendor` es `u"EDK II"`.

## 5.6 Shadow space y backtrace

![Shadow space](capturas/P5-06-shadow-space-y-backtrace.jpg)

`$rbp+0x10` y `$rbp+0x18` contienen los mismos valores que `rcx` y `rdx`: el prólogo los copió al shadow space que reservó el llamador. La dirección de retorno en `$rbp+8` pertenece a `ProcessModuleEntryPointList`, y el backtrace es `UefiMain` ← `ProcessModuleEntryPointList` ← `_ModuleEntryPoint`.

## 5.7 Antes del `call *%r8`

![Antes del call](capturas/P5-07-antes-del-call.jpg)

> ✏️ `rcx`, `rdx`, `r8` y `rsp`. Comparar `r8` con la base de 5.1: la función está fuera de nuestra imagen.

## 5.8 Retorno

![Retorno](capturas/P5-08-retorno.jpg)

`OutputString` imprimió `Hello World!` en QEMU y devolvió `rax = 0x0` (`EFI_SUCCESS`).

---

# Respuestas

> ✏️ Responder las preguntas de la Parte 7 de `GUIA.md`, citando las capturas.

---

# Conclusiones

> ✏️ Completar al terminar.
