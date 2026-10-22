# Trabajo Práctico N°3.a

## UEFI: Hello World en QEMU y depuración con GDB

**Una aplicación que corre dentro del firmware, antes que el sistema operativo**

---

**Universidad Nacional de Córdoba**
Facultad de Ciencias Exactas, Físicas y Naturales
Cátedra de Sistemas de Computación

**Profesor:** Ing. Javier Jorge

**Integrantes:**

- Nicolás Borsotti Bosco
- Santiago Valentín Ciacci
- Ignacio Ariel Leguizamón

---

## De qué se trata

El trabajo consiste en escribir una **aplicación UEFI** que imprima "Hello World",
ejecutarla sobre **OVMF** (un firmware UEFI construido con EDK2) dentro de **QEMU**, y
**depurarla con GDB**.

La diferencia de fondo con el TP #2 es que acá **no hay sistema operativo**: no hay
libc, ni `printf`, ni syscalls. Lo único que tiene la aplicación es un puntero a la
**System Table** que le pasa el firmware, y todo lo que hace (como escribir en la
consola) lo hace a través de los **protocolos** que cuelgan de esa tabla. Además, el
firmware usa otra convención de llamada (**Microsoft x64**, no System V), y la imagen es
un **PE/COFF reubicable**: su dirección se decide recién al cargarla, lo que obliga a
calcularla para poder depurar.

Seguimos el repositorio de referencia [UEFI-Lessons](https://github.com/javierbrk/UEFI-Lessons)
(lecciones 00 a 04 y 42), adaptado a la versión actual de EDK2, y el documento de la
cátedra *Arquitectura y Fundamentos del Firmware UEFI*.

## Organización del repositorio

```
practico-3/
├── README.md              este documento
├── CONSIGNA.md            qué pide el TP y en qué orden abordarlo
├── GUIA.md                la teoría y el paso a paso, con lo que hay que capturar
├── PROCEDIMIENTO.md       el registro de lo que fuimos haciendo, con capturas
├── capturas/              las imágenes referenciadas por PROCEDIMIENTO.md
├── Makefile               compilar, correr en QEMU y depurar
├── HelloPkg/              nuestro paquete EDK2
│   ├── HelloPkg.dsc       plataforma: módulos y bibliotecas
│   ├── SimplestApp/       la app mínima (Parte 2)
│   ├── HelloWorld/        Hello World con la System Table (Partes 3 y 5)
│   └── HelloPrint/        Hello World con gST y Print (Parte 4)
├── scripts/
│   ├── simbolos.sh        calcula dónde quedó cargada la app y arma el add-symbol-file
│   └── conectar.gdb       carga los símbolos, pone el breakpoint y se conecta a QEMU
└── disco/                 se monta en QEMU como FS0: (los .efi y la Shell como EFI/BOOT/BOOTX64.EFI)
```

| Documento | Para qué sirve |
|---|---|
| [`CONSIGNA.md`](CONSIGNA.md) | Los requisitos desglosados, el plan por partes y las decisiones tomadas |
| [`GUIA.md`](GUIA.md) | La teoría (Parte 0) y los pasos de cada parte, con los comandos y las capturas a sacar |
| [`PROCEDIMIENTO.md`](PROCEDIMIENTO.md) | El registro del trabajo realizado, parte por parte, con las capturas que verifican cada resultado |

| Parte | Tema | Capturas | Estado |
|---|---|---|---|
| Parte 1 | Entorno: EDK2, BaseTools, OVMF DEBUG, QEMU y GDB | 1 | pendiente |
| Parte 2 | La app mínima, el formato PE/COFF y las fases de arranque en el log | 5 | pendiente |
| Parte 3 | Hello World con `SystemTable->ConOut->OutputString` | 2 | pendiente |
| Parte 4 | Hello World con `gST` y `Print` | 2 | pendiente |
| Parte 5 | Depuración con GDB: dirección de carga, símbolos, convención Microsoft x64 y shadow space | 8 | pendiente |
| Parte 6 | Hello World en hardware real (bare metal): pendrive, Secure Boot y firmware Insyde | 11 | hecha |
| Parte 7 | Preguntas y conclusiones | — | pendiente |

## Cómo compilar y usar

Requiere Linux x86-64 con EDK2 (`edk2-stable202608`) clonado en `~/edk2` y sus BaseTools
compiladas, QEMU y GDB. Los pasos para preparar todo están en la Parte 1 de
[`GUIA.md`](GUIA.md).

```bash
make ovmf          # una sola vez: compila OVMF en DEBUG
make               # compila HelloPkg y copia los .efi a disco/
make run           # QEMU + OVMF; en la UEFI Shell: HelloWorld.efi
```

```
FS0:\> HelloWorld.efi
Hello World!
```

Para depurar (detalle en la Parte 5 de la guía):

```bash
make simbolos      # lee build/debug.log y calcula dónde se cargó la app
make debug         # terminal 1: QEMU esperando a GDB en :1234
make gdb           # terminal 2: GDB con los símbolos y un breakpoint en UefiMain
```

Si EDK2 está en otro lado: `make EDK2=/ruta/a/edk2 ...`
