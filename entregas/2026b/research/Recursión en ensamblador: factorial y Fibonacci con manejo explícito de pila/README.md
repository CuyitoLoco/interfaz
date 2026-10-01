Recursión en ensamblador: factorial y Fibonacci con manejo explícito de pila
Introducción

La recursión es una técnica de programación en la que una función se llama a sí misma para resolver un problema dividiéndolo en casos cada vez más pequeños. Aunque es común encontrarla en lenguajes de alto nivel como C, C++, Java o Python, también puede implementarse directamente en lenguaje ensamblador. Sin embargo, hacerlo requiere comprender con mayor detalle cómo funciona la memoria, los registros y, principalmente, la pila de ejecución.

En lenguajes de alto nivel, el compilador normalmente se encarga de administrar de manera automática la información necesaria para realizar llamadas recursivas. Cuando una función se ejecuta, se crea un marco de pila que puede contener parámetros, variables locales, direcciones de retorno y registros que deben conservarse. En ensamblador, gran parte de este proceso debe realizarse explícitamente mediante instrucciones que manipulan el registro de la pila.

Dos ejemplos clásicos para estudiar la recursión son el cálculo del factorial y la sucesión de Fibonacci. El factorial de un número entero positivo n se define como n! = n × (n-1)!, teniendo como caso base 0! = 1. Por otro lado, la sucesión de Fibonacci se define mediante F(n) = F(n-1) + F(n-2), con los casos base F(0) = 0 y F(1) = 1.

El objetivo de este trabajo es explicar cómo implementar ambos algoritmos utilizando ensamblador y un manejo explícito de la pila. Se analizará cómo se realizan las llamadas recursivas, cómo se almacenan temporalmente los valores y cómo se recupera la información cuando cada llamada termina. Esto permite comprender la relación existente entre los conceptos de recursión estudiados en programación y el funcionamiento interno de un procesador.

Desarrollo técnico
1. Concepto de pila en ensamblador

La pila, conocida como stack, es una región de memoria utilizada para almacenar información temporal durante la ejecución de un programa. Su funcionamiento se basa normalmente en el principio LIFO (Last In, First Out), lo que significa que el último elemento almacenado es el primero en recuperarse.

En arquitecturas como x86-64, el registro RSP (Stack Pointer) apunta a la posición actual de la pila. Las instrucciones PUSH y POP permiten introducir y retirar información de ella.

Por ejemplo:

push rax
pop rbx

La primera instrucción almacena el contenido de RAX en la pila, mientras que la segunda recupera ese valor y lo coloca en RBX.

La pila es especialmente importante durante una función recursiva porque cada llamada necesita conservar información correspondiente a su propio nivel de ejecución. Si una función se llama varias veces a sí misma, cada llamada debe mantener sus parámetros y valores temporales sin sobrescribir los utilizados por las llamadas anteriores.

Una llamada recursiva puede representarse conceptualmente de la siguiente manera:

función(n)
    guardar información de la llamada
    si n es caso base:
        regresar resultado
    llamar función(n-1)
    recuperar información
    calcular resultado
    regresar

Cada nueva llamada genera otro nivel de información en la pila.

2. Factorial recursivo

El factorial de un número entero n se define matemáticamente como:

n! = n × (n-1)!

El caso base es:

0! = 1

Por ejemplo:

5! = 5 × 4 × 3 × 2 × 1
5! = 120

Una implementación recursiva en ensamblador puede realizarse de la siguiente manera:

factorial:
    cmp rdi, 1
    jle base

    push rdi
    dec rdi
    call factorial
    pop rdi

    imul rax, rdi
    ret

base:
    mov rax, 1
    ret

En este ejemplo se utiliza una convención típica de llamadas de x86-64, donde RDI contiene el primer argumento y RAX contiene el valor de retorno.

La instrucción:

cmp rdi, 1
jle base

comprueba si el valor recibido es menor o igual que uno. Si se cumple esta condición, se llega al caso base y se coloca 1 en RAX.

Cuando el número es mayor que uno, primero se almacena el valor actual mediante:

push rdi

Después se decrementa RDI y se realiza la llamada recursiva:

dec rdi
call factorial

Cuando la llamada termina, el valor original se recupera utilizando:

pop rdi

Finalmente se multiplica el resultado obtenido por el valor original:

imul rax, rdi

Por ejemplo, al calcular factorial(4), las llamadas pueden visualizarse de la siguiente manera:

factorial(4)
    |
    +-- factorial(3)
            |
            +-- factorial(2)
                    |
                    +-- factorial(1)
                            |
                            +-- retorna 1
                    retorna 2
            retorna 6
    retorna 24

La pila permite conservar los valores 4, 3 y 2 mientras las llamadas más profundas continúan ejecutándose.

3. Fibonacci recursivo

La sucesión de Fibonacci se define de la siguiente manera:

F(0) = 0
F(1) = 1
F(n) = F(n-1) + F(n-2)

Los primeros valores de la sucesión son:

0, 1, 1, 2, 3, 5, 8, 13, 21, 34...

Una implementación recursiva básica en ensamblador puede ser:

fibonacci:
    cmp rdi, 0
    je fib_zero

    cmp rdi, 1
    je fib_one

    push rdi

    dec rdi
    call fibonacci

    push rax

    pop rax
    pop rdi

    ; En una implementación completa se deben
    ; conservar ambos resultados antes de la segunda llamada.

fib_zero:
    xor rax, rax
    ret

fib_one:
    mov rax, 1
    ret

Sin embargo, Fibonacci requiere un manejo de pila más complejo que factorial porque cada llamada genera dos llamadas recursivas:

F(n) = F(n-1) + F(n-2)

Por ejemplo:

F(4)
├── F(3)
│   ├── F(2)
│   │   ├── F(1)
│   │   └── F(0)
│   └── F(1)
└── F(2)
    ├── F(1)
    └── F(0)

Esto demuestra que el número de llamadas crece rápidamente. La implementación recursiva de Fibonacci tiene una complejidad temporal aproximada de O(2^n), mientras que el factorial recursivo requiere O(n) llamadas.

Una implementación más completa de Fibonacci requiere conservar el resultado de F(n-1) en la pila mientras se calcula F(n-2). Conceptualmente, el procedimiento es:

1. Guardar n.
2. Calcular F(n-1).
3. Guardar el resultado.
4. Recuperar n.
5. Calcular F(n-2).
6. Recuperar F(n-1).
7. Sumar ambos resultados.
8. Regresar el resultado.

Una posible implementación x86-64 es:

fibonacci:
    cmp rdi, 0
    je .zero

    cmp rdi, 1
    je .one

    push rdi

    dec rdi
    call fibonacci

    push rax

    pop rax
    pop rdi

    dec rdi
    push rax
    call fibonacci
    pop rbx

    add rax, rbx
    ret

.zero:
    xor rax, rax
    ret

.one:
    mov rax, 1
    ret

El ejemplo anterior sirve para mostrar la idea del manejo explícito de la pila, aunque una implementación real debe diseñarse cuidadosamente de acuerdo con la convención de llamadas utilizada y con la preservación de registros.

4. Importancia del manejo explícito de la pila

El manejo explícito de la pila permite observar directamente lo que ocurre durante una llamada recursiva. Cada nivel de la función puede almacenar sus propios datos y recuperarlos cuando la ejecución regresa.

En factorial, la pila puede almacenar los valores pendientes de multiplicar:

Antes de llegar al caso base:

[4]
[3]
[2]

Cuando se alcanza el caso base, las llamadas comienzan a regresar:

1 × 2 = 2
2 × 3 = 6
6 × 4 = 24

En Fibonacci, el proceso es más complejo porque cada nivel necesita conservar resultados mientras se ejecuta otra llamada recursiva.

Este comportamiento demuestra una característica importante de los sistemas operativos y procesadores: la memoria utilizada por una función recursiva no desaparece inmediatamente cuando se realiza otra llamada. Cada llamada necesita conservar la información necesaria para continuar después del retorno.

Un error en el manejo de la pila puede provocar problemas como pérdida de datos, corrupción de registros o incluso un desbordamiento de pila (stack overflow) cuando el número de llamadas recursivas es demasiado grande.

5. Comparación entre factorial y Fibonacci
Característica	Factorial	Fibonacci
Caso base	0! = 1	F(0)=0, F(1)=1
Llamadas recursivas	Una por nivel	Dos por nivel
Complejidad aproximada	O(n)	O(2^n)
Uso de pila	Moderado	Mayor
Dificultad de implementación	Menor	Mayor
Operación principal	Multiplicación	Suma
Ejemplo	5! = 120	F(6) = 8

La comparación muestra que factorial es un ejemplo más sencillo para comenzar a estudiar la recursión en ensamblador. Fibonacci permite profundizar en el manejo de múltiples llamadas y en la conservación de resultados intermedios.

Conclusiones

La implementación de algoritmos recursivos en lenguaje ensamblador permite comprender de manera más detallada cómo funcionan las llamadas a funciones a nivel de máquina. A diferencia de los lenguajes de alto nivel, donde gran parte de la administración de memoria se realiza automáticamente, en ensamblador es necesario controlar directamente los registros y la pila.

El factorial representa un ejemplo sencillo de recursión porque cada llamada genera solamente una nueva llamada. El valor actual puede almacenarse en la pila mientras se calcula el factorial del número siguiente. Una vez alcanzado el caso base, los valores almacenados se recuperan progresivamente y se realizan las multiplicaciones correspondientes.

Fibonacci presenta una mayor complejidad debido a que cada llamada genera dos llamadas recursivas. Esto hace que el número de operaciones aumente considerablemente conforme crece el valor de n. Además, es necesario conservar resultados intermedios para poder realizar la suma final.

El estudio de ambos algoritmos demuestra que la pila es un componente fundamental para implementar correctamente funciones recursivas. Comprender su funcionamiento permite relacionar conceptos de programación con aspectos internos del procesador, como los registros, las direcciones de retorno, los parámetros y los marcos de ejecución.

Finalmente, el manejo explícito de la pila permite comprender por qué los algoritmos recursivos pueden consumir una cantidad considerable de memoria y por qué una profundidad excesiva de llamadas puede producir un desbordamiento de pila. Por esta razón, conocer tanto la implementación recursiva como la iterativa es importante para seleccionar una solución adecuada dependiendo del problema y de los recursos disponibles.

Bibliografía

[1] R. Hyde, The Art of Assembly Language, 2nd ed. San Francisco, CA, USA: No Starch Press, 2010.

[2] K. R. Irvine, Assembly Language for x86 Processors, 7th ed. Upper Saddle River, NJ, USA: Pearson, 2015.

[3] Intel Corporation, Intel 64 and IA-32 Architectures Software Developer's Manual, Intel Corporation, 2024.

[4] D. E. Knuth, The Art of Computer Programming, Vol. 1: Fundamental Algorithms, 3rd ed. Reading, MA, USA: Addison-Wesley, 1997.

[5] T. H. Cormen, C. E. Leiserson, R. L. Rivest, and C. Stein, Introduction to Algorithms, 4th ed. Cambridge, MA, USA: MIT Press, 2022.
