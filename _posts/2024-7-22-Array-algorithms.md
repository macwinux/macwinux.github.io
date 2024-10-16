---
title: "Algoritmos sobre Listas en Golang"
categories: 
  - Golang
toc: true
toc_label: "Indice"
---

## Algoritmos basicos sobre listas

Normalmente, cuando intentas acceder a un puesto de trabajo como desarrollador en una empresa, se ha puesto de moda el tener varias entrevistas técnicas. Entre ellas, una de las más de moda y que pusieron de moda las grandes tecnológicas es la de desarrollar algoritmos sobre estructuras de datos.
He decidido estudiar estos algoritmos poco a poco. Conozco los básicos y creo que se implementarlos en otros lenguajes como puede ser Scala o Java pero me falta mucha base y precisamente es para lo que sirve desarrollar estos algoritmos. Te dan una base necesaria para que sea más fácil implementar luego otros algoritmos más complejos.

### Binary Search

Este algoritmo se basa en la presunción de que la lista con la que trabajas ya está ordenada. Lo que hace es ir diviendo la lista en dos partes y buscando en que sublista se encuentra el valor deseado, en caso de encontrarlo devuelve true

``` go

func BinarySearch(data []int, value int) bool {
  size := len(data)
  var mid int
  l := 0
  h := size - 1 //La última posición de la lista empezando por 0
  for l <= h {
    mid = l + (h-l)
  } 

}

```
**WIP** - Iré añadiendo algoritmos en go sobre la marcha para ir completando el post
{: .notice--warning}