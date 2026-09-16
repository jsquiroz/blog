+++
date = '2026-09-15'
draft = false
title = 'Memory allocator en Go parte 1'
tags = ['go', 'memoria', 'performance']
series = ['memory-allocator-in-go']
+++

En este post vamos a hablar sobre qué estrategias usa Go para gestionar la memoria en tus aplicaciones.

Partamos de la definición de una variable; básicamente, es un espacio en memoria que almacena un dato. Sin embargo, el espacio en memoria que se reserva no es cualquiera; existen 2 áreas de la memoria (stack/heap) en las que se va a almacenar el dato dependiendo del tipo de variable y su tiempo de vida.

## Dos zonas de memoria, dos comportamientos opuestos

El runtime de Go gestiona la memoria siguiendo dos modelos completamente opuestos. Entendamos cuáles son las principales características de estas zonas.

**Stack**

- Aislamiento e inicio ligero: Cada goroutine maneja su propio stack privado de 2 KB que crece dinámicamente según la demanda.
- Sin overhead del runtime: No existe un algoritmo ni una función especial del runtime para reservar variables en el stack; el compilador genera directamente el código máquina.
- Costo casi cero:
  - Asignar: La CPU resta bytes al registro del Stack Pointer `(SP = SP - n)`.
  - Liberar: La CPU suma los bytes de vuelta `(SP = SP + n)`.
- Impacto: Al reducirse a simples sumas y restas de registros a nivel de hardware, el impacto en rendimiento es virtualmente inexistente.

**Heap**

- Compartido y centralizado: Es un estanque global de memoria compartido entre todas las goroutines del proceso.
- Overhead del runtime: Cada asignación requiere ejecutar la función `runtime.mallocgc` para buscar bloques libres del tamaño adecuado y actualizar los metadatos del sistema.
- Costo elevado:
  - Asignar: La CPU debe gestionar la contención de bloqueos y la fragmentación al buscar memoria disponible.
  - Liberar: La memoria no se limpia al instante; depende del Garbage Collector (GC), el cual debe pausar o interrumpir procesos para escanear, marcar y barrer los objetos fuera de uso.
- Impacto: Introduce latencia por ciclos de CPU consumidos, fragmentación de memoria y presión constante sobre el Garbage Collector.

Entonces la pregunta real es: ¿quién decide cuál de las dos estrategias usar?

## Escape analysis: preguntarle al compilador

Go no tiene `malloc` ni `free`. Tiene una función del compilador llamada **escape analysis**. Si el compilador puede probar que una variable nunca sobrevive a su función, la pone en el stack. Si no puede probarlo, la variable "escapa" al heap.

Esa decisión se lee directamente al ejecutar el siguiente comando:

```sh
go build -gcflags="-m"
```

Hagamos el ejemplo de stack.

```go
//go:noinline
func sum(a, b int) int {
	c := a + b
	return c
}

//go:noinline
func main() {
	sum(1, 2)
}
```

El compilador responde:

```sh
$ go build -gcflags="-m"
$
```

No tenemos ninguna salida, lo que nos indica que no hubo escape to heap.

Ahora un ejemplo de heap.

```go
//go:noinline
func sum(a, b int) *int {
	c := a + b
	return &c
}

//go:noinline
func main() {
	sum(1, 2)
}
```

```sh
$ go build -gcflags="-m"
./main.go:5:2: moved to heap: c
$
```

En este ejemplo, el compilador nos está indicando que la variable c se fue al heap.

## Las rutas de escape de las que nadie te avisa

Devolver un puntero es la manera obvia de mandar algo al heap. Vamos a agregar más formas de mandar algo al heap.

### Closures

```go
func counter() func() int {
	x := 0
	return func() int {
		x++
		return x
	}
}
```

```sh
./main.go:35:2: moved to heap: x
./main.go:36:9: func literal escapes to heap
```

La función devuelta sobrevive a `counter` y sigue necesitando `x`. Entonces `x` no puede vivir en `counter`, que está por desaparecer.

### Goroutines

```go
func run() {
	for i := 0; i < 3; i++ {
		go func() {
			fmt.Println(i)
		}()
	}
}
```

La goroutine se ejecuta después de que termina la iteración. La goroutine necesita acceder a la variable i. La concurrencia altera el ciclo de vida de los datos, un detalle que se pasa por alto.

### Interfaces

```go
func Log(v interface{}) {
	fmt.Println(v) // v escapa
}
```

Asignar un valor concreto a una interfaz normalmente obliga al runtime a llevar la información de tipo junto con el dato. Por eso abusar de `fmt.Println` o `json.Marshal` en un hot path genera asignaciones en el heap de forma silenciosa..

Veamos un ejemplo de `fmt.Println`.

```go
//go:noinline
func sum(a, b int) int {
	c := a + b
	return c
}

//go:noinline
func main() {
	res := sum(1, 2)
	fmt.Println(res)
}
```

```bash
./main.go:14:14: res escapes to heap
```

## **La importancia de estos conceptos**

Entender el escape analysis no es para una optimización prematura. Se trata de desarrollar el modelo mental que Go espera que tengas.

Una vez que comprendes los tiempos de vida (*lifetimes*):

- Eliges entre valor y puntero de forma deliberada.
- Diseñas APIs que minimizan las asignaciones ocultas.
- Tu código escala mejor bajo carga.
- Evitas problemas comunes de concurrencia.
- Tu comportamiento resulta predecible para el compilador.
