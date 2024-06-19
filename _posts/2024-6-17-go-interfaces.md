---
title: Curiosidades de las interfaces en go
categories: 
  - Golang
toc: true
toc_label: "Índice"
---

Varias curiosidades sobre las interfaces en go.

### Poliformismo
Normalmente en go se usan las interfaces como identificadores, al igual que en otros lenguajes. Es decir, si tienes una interfaz llamada Animal que contiene un método llamado por ejemplo Aullar() y luego creas un `struct` llamado Perro por ejemplo que implementa dicho método. De esta manera, puedes implementar polimorfismo en go:

``` go

package main

import "fmt"

type Animal interface {
	Aullar()
}

type Perro struct{}

func (Perro) Aullar() {
	fmt.Println("Guau guau")
}

func main() {

	perros := []Animal{
		Perro{},
		Perro{},
	}
	for _, perro := range perros {
		perro.Aullar()
	}
}

``` 

### Type assertion y type-switch

Pero el tipo interfaz se puede usar como un saco para cualquier tipo de valor y luego usar un `type assertion` para definir que tipo de valor es. Y para el ejemplo vamos a usar el bucle switch que nos permite definir el tipo que contiene dicha interfaz.

```go

package main

import "fmt"

func main() {
	values := []interface{}{12334, "abc", nil}
	for _, x := range values {
		switch v := x.(type) {
		case int:
			fmt.Println("el valor es un int: ", v)
		case string:
			fmt.Println("el valor es una cadena de texto: ", v)
		case nil:
			fmt.Println(v)
		default:
			fmt.Println("others: ", v)
		}
	}
}

```

Este uso de las interfaces es curioso. No se si útil realmente pero es bastante curioso.