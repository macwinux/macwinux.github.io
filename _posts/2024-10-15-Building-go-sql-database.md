---
title: "Desarrollar una BBDD SQL en Go - Lexer"
categories: 
  - Databases
  - Golang
toc: true
toc_label: "Indice"
---

**Atención:** Esta serie de posts esta basado en el codigo de Phil Eaton llamado [`gosql`](https://github.com/eatonphil/gosql/tree/master)
{: .notice--info}

### Motivación

LLevo un par de semanas muy enfocado en entender un poco mejor como funcionan las BBDD por dentro. Esto me ha hecho empezar a ver ciertos cursos como el de introducción a los sistemas de bases de datos de la CMU o el de sistemas distribuidos del MIT.
A su vez sigo intentando motivarme a aprender go y se me ocurrio como implementar una pequeña bbdd sql en go. Hay varios ejemplos por internet y el de Phil Eaton me ha llamado bastante la atención por su sencillez.

### Lexico

Como punto de partida dentro del código, vamos a desarrollar el lexer, el motor que traduce un comando de SQL en una lista de tokens (lexing).

Estos tokens son básicamente identifiers, numbers, strings y symbols.

Lo primero que debemos hacer es definir los tipos y constantes que vamos a usar en el archivo `lexer.go`.

<details>
  <summary>tipos y constantes</summary>

  ### lexer.go
  ```go

  import (
  	"fmt"
  	"log/slog"
  	"strings"
  )

  // localizacion del token en el codigo
  type location struct {
  	line uint
  	col  uint
  }

  // para guardar las palabras clave reservadas de SQL
  type keyword string

  const (
  	selectKeyword keyword = "select"
  	fromKeyword   keyword = "from"
  	asKeyword     keyword = "as"
  	tableKeyword  keyword = "table"
  	createKeyword keyword = "create"
  	insertKeyword keyword = "insert"
  	intoKeyword   keyword = "into"
  	valuesKeyword keyword = "values"
  	intKeyword    keyword = "int"
  	textKeyword   keyword = "text"
  	whereKeyword  keyword = "where"
  	trueKeyword   keyword = "true"
  	falseKeyword  keyword = "false"
  	nullKeyword   keyword = "null"
  )

  // para guardar la sintaxis SQL
  type symbol string

  const (
  	semicolonSymbol  symbol = ";"
  	asteriskSymbol   symbol = "*"
  	commaSymbol      symbol = ","
  	leftParenSymbol  symbol = "("
  	rightParenSymbol symbol = ")"
  )

  type tokenKind uint

  const (
  	keywordKind tokenKind = iota
  	symbolKind
  	identifierKind
  	stringKind
  	numericKind
  	boolKind
  	nullKind
  )

  type token struct {
  	value string
  	kind  tokenKind
  	loc   location
  }

  // indica la posicion actual del lexer
  type cursor struct {
  	pointer uint
  	loc     location
  }

  func (t *token) equals(other *token) bool {
  	return t.value == other.value && t.kind == other.kind
  }

  type lexer func(string, cursor) (*token, cursor, bool)
  ```

</details>

