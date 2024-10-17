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

# Motivación

LLevo un par de semanas muy enfocado en entender un poco mejor como funcionan las BBDD por dentro. Esto me ha hecho empezar a ver ciertos cursos como el de introducción a los sistemas de bases de datos de la CMU o el de sistemas distribuidos del MIT.
A su vez sigo intentando motivarme a aprender go y se me ocurrio como implementar una pequeña bbdd sql en go. Hay varios ejemplos por internet y el de Phil Eaton me ha llamado bastante la atención por su sencillez.

# Lexico

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

## lexNumeric

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
		slog.Debug("Pointers ", slog.Int("cur pointer", int(cur.pointer)), slog.Int("ic pointer", int(ic.pointer)), slog.Int("cur loc", int(cur.loc.col)))

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

Vamos a ir paso por paso explicando el código. Para ir entendiendolo hemos puesto dos logs dentro de la función para ir viendo que valores coge durante la iteración.

Ahora vamos a hacer un test sobre esta función y ver los resultados pasandole distintos valores.

``` go
func TestToken_lexNumeric(t *testing.T) {

	//Setting log level
	l := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	}))
	slog.SetDefault(l)

	tests := []struct {
		number bool
		value  string
	}{
		{
			number: true,
			value:  "105 ",
		},
		{
			number: true,
			value:  "123.145",
		},
		{
			number: true,
			value:  "1.1e+2",
		},
	}
	for _, test := range tests {
		slog.Info("Value to test: " + test.value)
		tok, _, ok := lexNumeric(test.value, cursor{})
		slog.Info("Token ", slog.Group("token", "loc.col", tok.loc.col, "kind", tok.kind, "value", tok.value, "loc.line", tok.loc.line))
		assert.Equal(t, test.number, ok, test.value)
		if ok {
			assert.Equal(t, strings.TrimSpace(test.value), tok.value, test.value)
		}
	}
}

```

Vale, dentro del test he elegido varios ejemplos que hay para ilustrar como funciona el proceso.

### Valor '105 '

Veamos el resultado de los logs para ese valor (hay que tener en cuenta el espacio):

```terminal
{"time":"2024-10-17T06:31:17.205229281Z","level":"INFO","msg":"Value to test: 105 "}
{"time":"2024-10-17T06:31:17.205277834Z","level":"DEBUG","msg":"Character","char":"1"}
{"time":"2024-10-17T06:31:17.205291079Z","level":"DEBUG","msg":"Pointers","cur pointer":0,"ic pointer":0,"cur loc":1}
{"time":"2024-10-17T06:31:17.205303989Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:31:17.205307907Z","level":"DEBUG","msg":"Character","char":"0"}
{"time":"2024-10-17T06:31:17.20531037Z","level":"DEBUG","msg":"Pointers","cur pointer":1,"ic pointer":0,"cur loc":2}
{"time":"2024-10-17T06:31:17.205312768Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:31:17.20532359Z","level":"DEBUG","msg":"Character","char":"5"}
{"time":"2024-10-17T06:31:17.205325847Z","level":"DEBUG","msg":"Pointers","cur pointer":2,"ic pointer":0,"cur loc":3}
{"time":"2024-10-17T06:31:17.205328787Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:31:17.205331273Z","level":"DEBUG","msg":"Character","char":" "}
{"time":"2024-10-17T06:31:17.20534267Z","level":"DEBUG","msg":"Pointers","cur pointer":3,"ic pointer":0,"cur loc":4}
{"time":"2024-10-17T06:31:17.205345188Z","level":"DEBUG","msg":"Type","isDigit":false,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:31:17.205359511Z","level":"INFO","msg":"Token ","token":{"loc.col":0,"kind":4,"value":"105","loc.line":0}}
```

* El primer valor es el 1:
    * Se incrementa el valor de cur.loc.col 
	* Se marca el valor como digito, dejando los otros marcadores a falso
	* Como cur.pointer es igual a ic.pointer entra en el primer if pero no hace nada
* El segundo valor y el tercero siguen la misma pauta:
	* Se incrementa el valor de cur.loc.col
	* Se mara como digito
	* No se hace nada más dentro del bucle
* El último valor es " ":
	* Se incrementa el valor de cur.loc.col
	* No se marca ni como digito, ni como periodo ni como expresión
	* Entra en el if !isDigit y rompe el bucle, por lo tanto no suma a cur.pointer 1 en el bucle for.

Como todos los valores han sido correctos se le asigna el kind 4 que es el de un valor numérico y se devuelve el valor "105" sin el espacio ya que se rompio el bucle antes de añadir el último valor de la cadena string, por lo tanto se omite el espacio.

### Valor '123.145'

``` terminal
{"time":"2024-10-17T06:46:15.439077625Z","level":"INFO","msg":"Value to test: 123.145"}
{"time":"2024-10-17T06:46:15.439079529Z","level":"DEBUG","msg":"Character","char":"1"}
{"time":"2024-10-17T06:46:15.439091275Z","level":"DEBUG","msg":"Pointers","cur pointer":0,"ic pointer":0,"cur loc":1}
{"time":"2024-10-17T06:46:15.439093698Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439095738Z","level":"DEBUG","msg":"Character","char":"2"}
{"time":"2024-10-17T06:46:15.43910722Z","level":"DEBUG","msg":"Pointers","cur pointer":1,"ic pointer":0,"cur loc":2}
{"time":"2024-10-17T06:46:15.439119545Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439131075Z","level":"DEBUG","msg":"Character","char":"3"}
{"time":"2024-10-17T06:46:15.439142481Z","level":"DEBUG","msg":"Pointers","cur pointer":2,"ic pointer":0,"cur loc":3}
{"time":"2024-10-17T06:46:15.439153893Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439165565Z","level":"DEBUG","msg":"Character","char":"."}
{"time":"2024-10-17T06:46:15.439177017Z","level":"DEBUG","msg":"Pointers","cur pointer":3,"ic pointer":0,"cur loc":4}
{"time":"2024-10-17T06:46:15.439188961Z","level":"DEBUG","msg":"Type","isDigit":false,"isPeriod":true,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439200398Z","level":"DEBUG","msg":"Character","char":"1"}
{"time":"2024-10-17T06:46:15.439212124Z","level":"DEBUG","msg":"Pointers","cur pointer":4,"ic pointer":0,"cur loc":5}
{"time":"2024-10-17T06:46:15.439224027Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439226764Z","level":"DEBUG","msg":"Character","char":"4"}
{"time":"2024-10-17T06:46:15.439238608Z","level":"DEBUG","msg":"Pointers","cur pointer":5,"ic pointer":0,"cur loc":6}
{"time":"2024-10-17T06:46:15.439240866Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439252236Z","level":"DEBUG","msg":"Character","char":"5"}
{"time":"2024-10-17T06:46:15.439264019Z","level":"DEBUG","msg":"Pointers","cur pointer":6,"ic pointer":0,"cur loc":7}
{"time":"2024-10-17T06:46:15.439275794Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439301609Z","level":"INFO","msg":"Token ","token":{"loc.col":0,"kind":4,"value":"123.145","loc.line":0}}
```

* El primer valor es el 1:
    * Se incrementa el valor de cur.loc.col 
	* Se marca el valor como digito, dejando los otros marcadores a falso
	* Como cur.pointer es igual a ic.pointer entra en el primer if pero no hace nada porque es un digito
* El segundo valor y el tercero siguen la misma pauta:
	* Se incrementa el valor de cur.loc.col
	* Se mara como digito
	* No se hace nada más dentro del bucle
* El cuarto valor es ".":
	* Se incrementa el valor de cur.loc.col
	* Se marca como periódico
	* Entra en el if isPeriod y marca periodFound a true, por si luego encuentras otro . o un exponencial.

El resto de valores siguen igual pero hay que prestar atención a que al ser el último valor un valor válido el for incrementará cur.pointer hasta 7 y romperá el bucle porque 7 ya es igual a la longitud de la cadena.


### Valor '1.1e2'

```terminal
{"time":"2024-10-17T06:46:15.439327171Z","level":"INFO","msg":"Value to test: 1.1e2"}
{"time":"2024-10-17T06:46:15.439329285Z","level":"DEBUG","msg":"Character","char":"1"}
{"time":"2024-10-17T06:46:15.43933289Z","level":"DEBUG","msg":"Pointers","cur pointer":0,"ic pointer":0,"cur loc":1}
{"time":"2024-10-17T06:46:15.439345466Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439347825Z","level":"DEBUG","msg":"Character","char":"."}
{"time":"2024-10-17T06:46:15.439359637Z","level":"DEBUG","msg":"Pointers","cur pointer":1,"ic pointer":0,"cur loc":2}
{"time":"2024-10-17T06:46:15.439362148Z","level":"DEBUG","msg":"Type","isDigit":false,"isPeriod":true,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439373572Z","level":"DEBUG","msg":"Character","char":"1"}
{"time":"2024-10-17T06:46:15.4393851Z","level":"DEBUG","msg":"Pointers","cur pointer":2,"ic pointer":0,"cur loc":3}
{"time":"2024-10-17T06:46:15.439397172Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439410641Z","level":"DEBUG","msg":"Character","char":"e"}
{"time":"2024-10-17T06:46:15.439424536Z","level":"DEBUG","msg":"Pointers","cur pointer":3,"ic pointer":0,"cur loc":4}
{"time":"2024-10-17T06:46:15.439429837Z","level":"DEBUG","msg":"Type","isDigit":false,"isPeriod":false,"isExpMarker":true}
{"time":"2024-10-17T06:46:15.439441935Z","level":"DEBUG","msg":"Character","char":"2"}
{"time":"2024-10-17T06:46:15.439443916Z","level":"DEBUG","msg":"Pointers","cur pointer":4,"ic pointer":0,"cur loc":5}
{"time":"2024-10-17T06:46:15.439446325Z","level":"DEBUG","msg":"Type","isDigit":true,"isPeriod":false,"isExpMarker":false}
{"time":"2024-10-17T06:46:15.439471361Z","level":"INFO","msg":"Token ","token":{"loc.col":0,"kind":4,"value":"1.1e2","loc.line":0}}
```

Este caso es similar al anterior hasta que encuentras el exponencial:
* El valor "e":
    * Se incrementa el valor de cur.loc.col 
	* Se marca el valor como isExpMarker, dejando los otros marcadores a falso
	* Entras en el if isExpMarker
		* Se marca periodFound a true (aunque ya lo estaba) y expMarkerFound a true.
		* El resto de ifs no nos aplican en este caso asi que continuamos el bucle.


## Conclusión 

Cuando ves el código de primeras puede parecer confuso y exotérico pero la verdad es que es bastante sencillo de seguir una vez debugueas un poco las variables y vas comprobando como funciona.
Hay un libro que explica esta parte muy bien que te explica paso por paso como escribir un compilador en go `Writing An Interpreter In Go` de Throsten Ball, por si quieres profundizar más en esta parte.

En el siguiente post veremos el resto de funciones para los otros tipos y luego ya pasaremos a la chicha propiamente dicha.
