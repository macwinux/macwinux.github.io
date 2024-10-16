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

## Motivación

LLevo un par de semanas muy enfocado en entender un poco mejor como funcionan las BBDD por dentro. Esto me ha hecho empezar a ver ciertos cursos como el de introducción a los sistemas de bases de datos de la CMU o el de sistemas distribuidos del MIT.
A su vez sigo intentando motivarme a aprender go y se me ocurrio como implementar una pequeña bbdd sql en go. Hay varios ejemplos por internet y el de Phil Eaton me ha llamado bastante la atención por su sencillez.

## Lexico

Como punto de partida dentro del código, vamos a desarrollar el lexer, el motor que traduce un comando de SQL en una lista de tokens (lexing).

Estos tokens son básicamente identifiers, numbers, strings y symbols.

Lo primero que debemos hacer es definir los tipos y constantes que vamos a usar en el archivo `lexer.go`.

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

### lexNumeric

Ahora vamos a implementar el primer lexer para números:

``` go

func lexNumeric(source string, ic cursor) (*token, cursor, bool) {
	cur := ic

	periodFound := false
	expMarkerFound := false

	for ; cur.pointer < uint(len(source)); cur.pointer++ {
		c := source[cur.pointer]
		cur.loc.col++

		isDigit := c >= '0' && c <= '9'
		isPeriod := c == '.'
		isExpMarker := c == 'e'

		slog.Debug("Character ", slog.String("char", string(c)))
		slog.Debug("Pointers ", slog.Int("cur pointer", int(cur.pointer)), slog.Int("ic pointer", int(ic.pointer)))
		// Must start with a digit or period
		if cur.pointer == ic.pointer {
			if !isDigit && !isPeriod {
				return nil, ic, false
			}
			periodFound = isPeriod
			continue
		}

		if isPeriod {
			if periodFound {
				return nil, ic, false
			}

			periodFound = true
			continue
		}

		if isExpMarker {
			if expMarkerFound {
				return nil, ic, false
			}

			// No periods allowed after expMarker
			periodFound = true
			expMarkerFound = true

			// expMarker must be followeb by digits
			if cur.pointer == uint(len(source)-1) {
				return nil, ic, false
			}

			cNext := source[cur.pointer+1]
			if cNext == '-' || cNext == '+' {
				cur.pointer++
				cur.loc.col++
			}
			continue
		}
		if !isDigit {
			break
		}
	}

	// No characters accumulated
	if cur.pointer == ic.pointer {
		return nil, ic, false
	}

	return &token{
		value: source[ic.pointer:cur.pointer],
		loc:   ic.loc,
		kind:  numericKind,
	}, cur, true
}

```

Vamos a ir paso por paso explicando el código.
**WIP** - Iré añadiendo algoritmos en go sobre la marcha para ir completando el post
{: .notice--warning}