# TP #3.a — UEFI

Hoja de ruta por partes. Sistemas de Computación (UNC) — Ing. Javier Jorge.

Este documento no es la consigna oficial: es la consigna traducida a pasos ejecutables
y ordenada de forma que cada parte dependa solo de la anterior.

**Integrantes:** Nicolás Borsotti Bosco · Santiago Valentín Ciacci · Ignacio Ariel Leguizamón

---

## La consigna

> **Práctico #3.a — UEFI**
> - Revisar el repositorio <https://github.com/javierbrk/UEFI-Lessons>
> - Crear un hello world de EFI que pueda correr en QEMU y depurarlo
> - Más info en el documento adjunto (*Arquitectura y Fundamentos del Firmware UEFI*)

## Qué pide, en una frase

Escribir una **aplicación UEFI** que imprima "Hello World", hacerla correr sobre un
firmware UEFI real (**OVMF**) dentro de **QEMU** y **depurarla con GDB**: llegar con un
breakpoint a su punto de entrada y mostrar cómo recibe los datos del firmware y cómo lo
llama de vuelta.

Igual que en el TP #2, no alcanza con que el programa ande: lo que se entrega es el
programa **más la evidencia de que se entiende qué pasa**. Esa evidencia es la sesión de
GDB de la Parte 5 y la conexión de cada resultado con la teoría del documento adjunto.

## Requisitos y dónde se cumplen

| # | Requisito | Dónde se cumple |
|---|---|---|
| R1 | Revisar UEFI-Lessons | Las Partes 1 a 5 siguen las lecciones 00, 01, 02, 03, 04 y 42, adaptadas a la versión actual de EDK2 |
| R2 | Crear un Hello World de EFI | Parte 3 (`HelloWorld`, usando la System Table directo) y Parte 4 (`HelloPrint`, con las bibliotecas de EDK2) |
| R3 | Que corra en QEMU | Partes 2 a 4: OVMF + UEFI Shell en QEMU, con `disco/` montado como `FS0:` |
| R4 | Depurarlo | Parte 5: GDB conectado al stub de QEMU, símbolos reubicados, breakpoint en `UefiMain` y seguimiento de la llamada a `OutputString` |
| R5 | Justificar con la teoría del documento | Parte 0 de `GUIA.md` + las respuestas y conclusiones de `PROCEDIMIENTO.md` |

## Plan por partes

| Parte | Qué se hace | Qué se demuestra | Lección base |
|---|---|---|---|
| 0 | Teoría | UEFI vs. PI, fases SEC/PEI/DXE/BDS/RT, System Table, protocolos y handles, PE/COFF, convención de llamada Microsoft x64 | documento adjunto |
| 1 | Entorno: EDK2, BaseTools, OVMF DEBUG, QEMU, GDB | — | 00 |
| 2 | `SimplestApp`: la app mínima | Ciclo compilar → copiar → ejecutar; el `.efi` es PE/COFF; las fases de arranque en el log | 01, 02 |
| 3 | `HelloWorld` con `SystemTable->ConOut->OutputString` | Cómo se llega a un servicio del firmware sin libc; offsets de las estructuras en el desensamblado | 03 |
| 4 | `HelloPrint` con `gST` y `Print` | Constructores de bibliotecas; el costo en tamaño | 04 |
| 5 | Depuración con GDB | Dirección de carga, `add-symbol-file`, argumentos en `rcx`/`rdx`, shadow space, llamada indirecta, `EFI_STATUS` en `rax` | 42 |
| 6 | Hello World en hardware real (bare metal) | El mismo `.efi` corre sin cambios sobre un firmware real (Insyde); ruta por defecto `\EFI\BOOT\BOOTX64.EFI`; Secure Boot | documento adjunto (TP 3) |
| 7 | Preguntas y conclusiones | — | — |

## Decisiones tomadas

- **EDK2 y no gnu-efi.** Es lo que usa UEFI-Lessons, y además permite compilar OVMF en
  DEBUG. Sin el log de OVMF no sabríamos dónde se carga la app (Parte 5).
- **Versión fija de EDK2: `edk2-stable202608`.** Las lecciones son de 2021 y desde
  entonces cambiaron cosas: el toolchain `GCC5` pasó a llamarse `GCC`, ahora es
  obligatorio declarar una `StackCheckLib` y las secciones se alinean a 4 KiB. Fijar el
  tag hace que lo que dice la guía coincida con lo que se ve.
- **`HelloPkg` vive en nuestro repo**, no adentro de `edk2`, gracias a `PACKAGES_PATH`.
- **Target `NOOPT`** (`-O0`) para la app que depuramos, por lo mismo que en el TP #2
  usábamos `-O0`: el código generado sigue al fuente línea por línea.
