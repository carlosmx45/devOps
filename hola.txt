# Resumen Memoria Compartida

## Contexto Sobre Memoria Compartida
La memoria compartida (SMEM) es uno de los componentes clave de la GPU.

Físicamente, cada SM contiene un pequeño grupo de memoria de baja latencia compartido por todos los subprocesos en el bloque de subprocesos que se ejecuta actualmente en ese SM.

La memoria compartida permite que los subprocesos dentro del mismo bloque de subprocesos cooperen, facilita la reutilización de datos en el chip y puede reducir en gran medida el ancho de banda de la memoria global que necesitan los núcleos.

Debido a que la aplicación administra explícitamente el contenido de la memoria compartida, a menudo se la describe como una caché administrada por el programa.


## Volatile Qualifier
Declarar una variable en la memoria global o compartida usando el *calificador volátil* evita la optimización del compilador, que podría almacenar datos temporalmente en registros o en la memoria local.

Con el *calificador volátil*, el compilador asume que el valor de la variable puede ser cambiado o utilizado en cualquier momento por cualquier hilo.

Por lo tanto, cualquier referencia a esta variable se compila en una lectura de memoria global o escritura de memoria global que omite la caché.

---
## SHARED MEMORY VERSUS GLOBAL MEMORY
La memoria global de la GPU se encuentra en el dispositivo de memoria (DRAM), por lo que es más lento de acceder que la memoria compartida de la GPU.

Comparada con la DRAM, la memoria compartida tiene:
- De 20 a 30 veces menor latencia que la DRAM
- Más de 10 veces mayor ancho de banda que la DRAM

La granularidad de acceso a la memoria compartida también es menor.

Mientras que la granularidad de acceso de la DRAM es de 32 bytes o 128 bytes, la granularidad de acceso de la memoria compartida es la siguiente:
- Fermi: ancho de banco de 4 bytes
- Kepler: ancho de banco de 8 bytes

---
## LAYOUT OF SHARED MEMORY
Para comprender completamente cómo utilizar la memoria compartida de forma eficaz hay que tener en cuenta:
- Arreglos Cuadrados vs Arreglos Rectangulares
- Acceso a la Fila Mayor vs Acceso a la Columna Mayor
- Declaración de Memoria Compartida Estática vs Dinámica
- Alcance del Archivo vs Alcance del Kernel
- Padding de Memoria vs No Padding de Memoria

Al diseñar un kernel que utilice la memoria compartida, tu enfoque debe de seguir los siguientes conceptos:
- Mapeo de datos entre bancos de memoria
- Mapeo del indice del hilo al desplazamiento de la memoria compartida

Con estos conceptos claros en mente, puede diseñar un núcleo eficiente para evitar conflictos bancarios y utilizar plenamente los beneficios de la memoria compartida.

---
## Square Shared Memory
Puede utilizar la memoria compartida para almacenar en caché datos globales con dimensiones cuadradas de forma sencilla.

La dimensionalidad simple de una matriz cuadrada facilita el cálculo de compensaciones de memoria 1D a partir de índices de subprocesos 2D.

Para declarar una variable estatica 2D en memoria compartida, lo haces de la siguiente forma:

      __shared__ int tile [N][N];

Como esta variable en memoria compartida es cuadrada, podrás acceder a ella con un bloque de hilos 2D de la siguiente manera:

        tile[threadIdx.y][threadIdx.x]
Esta forma de acceder es mucho más eficiente y de menor conflicto de bancos que esta:

        tile[threadIdx.x][threadIdx.y]
Esto debido a que los hilos vecinos están accediendo a las celdas de la matriz vecina a lo largo de la dimensión más interna de la matriz.

---
## Accessing Row-Major versus Column-Major
Considere un ejemplo en el que se utiliza una cuadrícula con un bloque 2D que contiene 32 hilos en cada dimensión.

Puede definir las dimensiones del bloque usando la siguiente macro:

        #define BDIMX 32
        #define BDIMY 32
También puedes usar esa macro para definir la configuración de ejecución para el kernel:

        dim3 block (BDIMX,BDIMY);
        dim3 grid (1,1);
El kernel tiene dos operaciones simples:
- Escriba índices de subprocesos globales en una matriz de memoria compartida 2D en orden de fila principal.
- Lea esos valores de la memoria compartida en orden de fila principal y guárdelos en la memoria global.

---
## Writing Row-Major and Reading Column-Major
Para implementar la escritura en filas de la memoria compartida, debemos poner el dimensión más interna del índice del hilo como índice de columna del tile de la memeria compartida 2D:
    
        tile[threadIdx.y][threadIdx.x] = idx;
Asignar valores a la memoria global desde la memoria compartida en orden de columna se hace al intercambiar dos índices de hilos cuando se referencian en memoria compartida:

        out[idx] = tile[threadIdx.x][threadIdx.y];

Con esto, la operación de almacenar se encuentra libre de conflictos, pero la operación de cargar reporta 16 conflictos.

---
## Dynamic Shared Memory
Puedes implementar los mismos kernels al declarar la memoria compartida de forma dinámica.

Esto lo puedes hacer tanto fuera del kernel para hacerlo global al alcance del archivo, o dentro del kernel para restringirlo a un alcance de kernel.

La memoria compartida dinámica debe ser declarada como un arreglo de una dimensión sin tamaño, Por lo tanto, debes calcular los índices de acceso a la memoria en base a los índices de hilos 2D.

- Como vas a escribir en filas y leer en columnas en este kernel, debes mantener dos índices de la siguiente manera:
    - *row_idx*: Compensación de memoria en fila 1D calculado desde índices de hilos 2D
    - *col_idx*: Compensación de memoria de columnas 1D calculado desde índices de hilos 2D

Para escribir en la memoria compartida en filas utilizamos *row_idx*:

        tile[row_idx] = row_idx;

Una vez que todo este sincronizado y que la memoria compartida este llena, podemos leerla en orden de columna y asignar a la memoria global:

        out[row_idx] = tile[col_idx];

Esto se debe a que *out* esta almacenada en la memoria global y los hilos dentro de un bloque de hilos están organizados en orden de fila.

Es por ello que para escribir en *out* en forma de filas debemos coordinar los hilos para garantizar almacenes fusionados.

El tamaño de la memoria compartida debe de ser especificada cuando se lance el kernel:

        setRowReadColDyn<<<grid, block, BDIMX * BDIMY * sizeof(int)>>>(d_C);

---
## Padding Statically Declared Shared Memory
Para resolver conflictos de bancos de memeria de una memoria compartida rectangular, podemos utilizar el padding. Sin embargo, para dispositivos Kepler debes calcular cuantos elementos de padding son necesarios. 

Para facilitar el código, utilizaremos una macro que defina el número de columnas de padding agregadas a cada fila:

        #define NPAD 2

Para declarar el padding estático en la memoria compartida se realiza así:

        __shared__ int tile[BDIMY][BDIMX + NPAD];
        
Cambiar el número de elementos de padding resulta en un reporte de que las operaciones de carga de la memoria compartida son atendidos por dos transacciones, lo que significa, un conflicto de banco de memoria de dos direcciones.

---
## Padding Dynamically Declared Shared Memory
El padding también puede ser utilizado por kernels que utilicen memoria compartida dinámica rectangular. Como el padding de la memoria compartida y la memoria global tendrán diferentes tamaños,
Se deben mantener tres índices por cada hilo del kernel:
- *row_idx*: Índice de fila para el padding de la memoria compartida. Al usar este índice, el warp puede acceder a una fila de la matriz.
- *col_idx*: Índice de columna para el padding de la memoria compartida. Al usar este índice, el warp puede acceder a una columna de la matriz.
- *g_idx*: Índice a la memoria global lineal. Al usar este índice, un warp puede fucionar accesos a la memoria global.

Estos índices se pueden calcular de la siguiente manera:

        unsigned int g_idx = threadIdx.y * blockDim.x + threadIdx.x;
        unsigned int irow = g_idx / blockDim.y;
        unsigned int icol = g_idx % blockDim.y;
        unsigned int row_idx = threadIdx.y * (blockDim.x + IPAD) + threadIdx.x;
        unsigned int col_idx = icol * (blockDim.x + IPAD) + irow;

---
## Performance of the Rectangular Shared Memory Kernels
Los kernels que utilizan padding de la memoria compartida ganan rendimiento al remover conflictos de bancos de memoria y los kernels con memoria compartida dinámica reportan una menor cantidad de gastos de recursos.

---
## REDUCING GLOBAL MEMORY ACCESS
Una de las principales razones para utilizar la memoria compartida, es para almacenar datos en la caché del chip. De esta forma reducimos la cantidad de accesos a la memoria global en nuestro kernel. 
