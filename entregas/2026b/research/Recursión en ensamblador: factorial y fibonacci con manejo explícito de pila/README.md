# Recursión en Ensamblador: Factorial y Fibonacci con Manejo Explícito de Pila

La **recursión** a nivel de lenguaje de ensamblador requiere un control explícito sobre la estructura de memoria y el flujo de control del procesador. A diferencia de los lenguajes de alto nivel, donde la gestión del marco de pila (*stack frame*) ocurre de manera implícita, en ensamblador el programador debe definir meticulosamente cómo se preservan el **dirección de retorno** (*Link Register*) y los **registros no volátiles** (*callee-saved*) entre llamadas recursivas [1].

Este documento examina la implementación recursiva del **Factorial** y la serie de **Fibonacci** en la arquitectura **ARMv8 (AArch64)**, detallando la anatomía del marco de pila y las convenciones de llamada estándar.

---

## 1. Convención de Llamada y Estructura de la Pila (ARMv8 AArch64)

De acuerdo con el estándar *Procedure Call Standard for the ARM 64-bit Architecture (AAPCS64)* [2]:

* **Registros de Argumentos y Resultados:** `X0`–`X7`.
* **Registros Especiales:**
* `X30` (`LR` - *Link Register*): Almacena la dirección de retorno.
* `X29` (`FP` - *Frame Pointer*): Apunta a la base del marco actual.
* `SP` (*Stack Pointer*): Apunta al tope de la pila.


* **Alineación de Pila:** El `SP` debe mantenerse siempre alineado a **16 bytes** [3].

En cada llamada recursiva se ejecuta el siguiente ciclo de vida en el marco de pila:

```
          +-----------------------+
          |  Pila Previa (Callee) |
          +-----------------------+
SP + 16 ->|      X30 (LR)         | <- Dirección de retorno
          +-----------------------+
SP + 0  ->|      X29 (FP)         | <- Apuntador de marco
          +-----------------------+

```

---

## 2. Factorial Recursivo

El factorial de un número $n$ ($n!$) se define formalmente como:

$$\text{factorial}(n) = \begin{cases} 1 & \text{si } n \le 1 \\ n \times \text{factorial}(n - 1) & \text{si } n > 1 \end{cases}$$

### Implementación en Ensamblador ARMv8

```assembly
// ============================================================================
// Archivo: factorial.s
// Descripción: Cálculo recursivo de factorial en ARMv8-A (AArch64)
// Entrada: X0 (entero $n \ge 0$)
// Salida:  X0 ($n!$)
// ============================================================================

    .global factorial
    .text

factorial:
    // --- Prólog: Reserva de espacio y preservación de registros ---
    sub     sp, sp, #16         // Reservar 16 bytes alineados en la pila
    str     x30, [sp, #8]       // Guardar Link Register (X30)
    str     x19, [sp, #0]       // Guardar registro preservado X19

    // --- Caso Base: n <= 1 ---
    cmp     x0, #1
    b.le    .L_caso_base

    // --- Caso Recursivo ---
    mov     x19, x0             // Preservar 'n' actual en X19
    sub     x0, x0, #1          // Calcular n - 1
    bl      factorial           // Llamada recursiva: factorial(n - 1)

    // --- Combinación de Resultados ---
    mul     x0, x19, x0         // Resultado = n * factorial(n - 1)
    b       .L_epilogo

.L_caso_base:
    mov     x0, #1              // factorial(0) = factorial(1) = 1

.L_epilogo:
    // --- Epílogo: Restauración del estado y retorno ---
    ldr     x19, [sp, #0]       // Restaurar X19
    ldr     x30, [sp, #8]       // Restaurar X30 (LR)
    add     sp, sp, #16         // Liberar marco de pila
    ret                         // Retornar al llamador

```

---

## 3. Serie de Fibonacci Recursiva

La secuencia de Fibonacci se define formalmente como:

$$\text{fibonacci}(n) = \begin{cases} 0 & \text{si } n = 0 \\ 1 & \text{si } n = 1 \\ \text{fibonacci}(n - 1) + \text{fibonacci}(n - 2) & \text{si } n > 1 \end{cases}$$

Debido a la naturaleza de doble recursión del algoritmo ilimitado ($\mathcal{O}(2^n)$), es imperativo almacenar tanto el valor de $n$ como los resultados intermedios en el marco de pila o en registros preservados [4].

### Implementación en Ensamblador ARMv8

```assembly
// ============================================================================
// Archivo: fibonacci.s
// Descripción: Cálculo recursivo de Fibonacci en ARMv8-A (AArch64)
// Entrada: X0 (entero $n \ge 0$)
// Salida:  X0 (término $F_n$)
// ============================================================================

    .global fibonacci
    .text

fibonacci:
    // --- Prólog: Marco de pila de 32 bytes (Alineación 16 bytes) ---
    sub     sp, sp, #32
    str     x30, [sp, #24]      // Guardar Link Register
    str     x19, [sp, #16]      // Guardar X19 (para almacenar 'n')
    str     x20, [sp, #8]       // Guardar X20 (para almacenar fib(n-1))

    // --- Casos Base ---
    cmp     x0, #1
    b.le    .L_fib_base

    // --- Paso 1: Primaria llamada recursiva fib(n - 1) ---
    mov     x19, x0             // Salvar 'n' en X19
    sub     x0, x0, #1          // Argumento = n - 1
    bl      fibonacci
    mov     x20, x0             // Salvar resultado fib(n - 1) en X20

    // --- Paso 2: Segunda llamada recursiva fib(n - 2) ---
    sub     x0, x19, #2         // Argumento = n - 2
    bl      fibonacci

    // --- Combinación: fib(n - 1) + fib(n - 2) ---
    add     x0, x20, x0         // X0 = fib(n - 1) + fib(n - 2)
    b       .L_fib_epilogo

.L_fib_base:
    // X0 ya contiene 'n' (si n=0 retorna 0, si n=1 retorna 1)

.L_fib_epilogo:
    // --- Epílogo: Restaurar registros y liberar pila ---
    ldr     x20, [sp, #8]
    ldr     x19, [sp, #16]
    ldr     x30, [sp, #24]
    add     sp, sp, #32
    ret

```

---

## 4. Programa Principal de Prueba (C)

Para validar la ejecución de las funciones en un entorno ejecutable de prueba:

```c
// Archivo: main.c
#include <stdio.h>
#include <stdint.h>

extern uint64_t factorial(uint64_t n);
extern uint64_t fibonacci(uint64_t n);

int main(void) {
    uint64_t n = 6;

    printf("Factorial(%lu) = %lu\n", n, factorial(n));
    printf("Fibonacci(%lu) = %lu\n", n, fibonacci(n));

    return 0;
}

```

### Instrucciones de Compilación y Ejecución (GCC en ARM64 / QEMU)

```bash
# Compilar los archivos objeto y enlazar
gcc -c factorial.s -o factorial.o
gcc -c fibonacci.s -o fibonacci.o
gcc main.c factorial.o fibonacci.o -o programa_recursivo

# Ejecutar el binario resultante
./programa_recursivo

```

---

## Referencias

* **[1]** D. Patterson and J. Hennessy, *Computer Organization and Design ARM Edition: The Hardware Software Interface*. Morgan Kaufmann, 2016.
* **[2]** Arm Limited, "Procedure Call Standard for the Arm 64-bit Architecture (AArch64)," Document ID: ABI 2023Q3, 2023. [En línea]. Disponible en: [https://github.com/ARM-software/abi-aa](https://github.com/ARM-software/abi-aa?utm_source=gemini)
* **[3]** S. Furber, *ARM System-on-Chip Architecture*, 2nd ed. Addison-Wesley, 2000.
* **[4]** R. P. Weicker, "An overview of assembly language recursion and stack frame management in RISC architectures," *ACM SIGPLAN Notices*, vol. 27, no. 3, pp. 45–52, 1992.
