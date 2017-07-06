Parser combinator library in Golang
===================================

[![Join the chat at https://gitter.im/prataprc/goparsec](https://badges.gitter.im/prataprc/goparsec.svg)](https://gitter.im/prataprc/goparsec?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Build Status](https://travis-ci.org/prataprc/goparsec.svg?branch=master)](https://travis-ci.org/prataprc/goparsec)
[![Coverage Status](https://coveralls.io/repos/github/prataprc/goparsec/badge.svg?branch=master)](https://coveralls.io/github/prataprc/goparsec?branch=master)
[![GoDoc](https://godoc.org/github.com/prataprc/goparsec?status.png)](https://godoc.org/github.com/prataprc/goparsec)

A library to construct top-down recursive backtracking parsers using
parser-combinators.  To know more about theory of parser
combinators, [refer here](http://en.wikipedia.org/wiki/Parser_combinator).

This package contains following components,

* Standard set of combinators, parsing algorithms can create input
  specific combinator functions on top of the standard set.
* [Regular expression](https://golang.org/pkg/regexp/) based simple-scanner.
* Standard set of tokenizers.
* To construct syntax-trees based on detailed grammar try with
  [AST struct](http://godoc.org/github.com/prataprc/goparsec#AST).
  * Standard set of combinators are exported as methods to AST.
  * Generate dot-graph EG: [dotfile](testdata/simple.dot)
  for [html](testdata/simple.html).
  * Pretty print on the console.
  * Make debugging easier.

**NOTE that AST object is a recent development and expect user to adapt to
newer versions**

Quick links
-----------

* [List of combinators](#list-of-combinators).
* [Using the builtin scanner](#using-the-builtin-scanner).
* [Projects using goparsec](#projects-using-goparsec).
* [Simple html parser](#simple-html-parser).
* [Articles](#articles).
* [How to contribute](#how-to-contribute).

Combinators
-----------

Every combinator should confirm to the following signature,

```go
    // ParsecNode type defines a node in the AST
    type ParsecNode interface{}

    // Parser function parses input text, higher order parsers are
    // constructed using combinators.
    type Parser func(Scanner) (ParsecNode, Scanner)

    // Nodify callback function to construct custom ParsecNode.
    type Nodify func([]ParsecNode) ParsecNode
```

Combinators take a variable number of parser functions and
return a new parser function.

``Nodify`` is user supplied callback function, called by combinator when the
rule matches current input text. The callback function receives a
collection of ParsecNode interface generated by ``parsers``.

A ``Parser`` function must take a new copy of ``Scanner`` instance, with current
cursor, as argument, and consume the scanner as long as there is a match. If
the parser could not match with input text it returns the scanner without
consuming any input. For example, the following snippet is from ``And``
combinator,

```go
    var ns = make([]ParsecNode, 0, len(parsers))
    var n ParsecNode
    news := s.Clone()
    for _, parser := range parsers {
        n, news = doParse(parser, news)
        if n == nil {
            return nil, s
        }
        ns = append(ns, n)
    }
    return docallback(callb, ns), news
```

We can see that a new instance of ``s`` is passed to each ``parser`` and when one
of the parser returns failure (where n==nil), it simply returns the scanner
without consuming any tokens. Otherwise, it returns the new-scanner ``news``
returned by the last parser.

If the intermediate nodes are created by the Combinators then it will be of
the following types:

* **And** combinator, by default, return []ParsecNode, with arity of array
  equal to number parsers supplied to the combinator.
* **OrdChoice** combinator, by default, return []ParsecNode with arith of
  array as one.
* **Kleene** combinator, by default, return []ParsecNode with arity >= 0. Note
  that even if arity is ZERO Kleene combinator succeeds, this can be a common
  pitfall.
* **Many** combinator, by default, return []ParsecNode with arity > 0.
* **ManyUntil** combinator, by default, return []ParsecNode with arity > 0.
* **Maybe**, combinators, by default, return ParsecNode.

List of combinators
===================

And
---

```go
    func And(callb Nodify, parsers ...interface{}) Parser {
```

Accepts a collection of parsers that must sequentially match current
input text, combinator fails when any of the parser fails to match.

Nodify callback is called with a slice of ParsecNodes obtained from each
parser, otherwise callback is ignored.

OrdChoice
---------

```go
    func OrdChoice(callb Nodify, parsers ...interface{}) Parser {
```

Accepts a collection of parsers, where at least one of the parser should
match current input text, combinator fails when all the parser fail to
match.

Nodify callback is called with a slice of single ParsecNode element when
one of the input parser matches with input text, otherwise callback is
ignored.

When a parser matches the input text remaining parsers are not tried.

Kleene
------

```go
    func Kleene(callb Nodify, parsers ...interface{}) Parser {
```

Accepts a pair of parser, where the first element must match _zero or more
times_ with current input text and the second optional element acts as token
separator, kleene combinator will exit when the first parser or the
second parser, if specified, fails.

Nodify callback is called with a slice of ParsecNodes obtained from every
match of the first parser, otherwise called with empty-slice of ParsecNodes.

Many
----

```go
    func Many(callb Nodify, parsers ...interface{}) Parser {
```

Accepts a pair of parser, where the first element must match _one or more
times_ with current input text and the second optional element acts as token
separator. Note that the Many repetition will exit when first parser or
second parser, if specified, fails.

Nodify callback is called with a slice of ParsecNodes obtained from every
match of the first parser, otherwise callback is ignored.

ManyUntil
---------

```go
    func ManyUntil(callb Nodify, parsers ...interface{}) Parser {
```

Accepts a two or three parsers, where the first element must match _one or more
times_ with current input text and the second optional element acts as token
separator. The last parser specifies a final token to stop matching. Note that
the ManyUntil repetition will exit when first parser or second parser, if
specified, fails or the last parser succeeds.

Nodify callback is called with a slice of ParsecNodes obtained from every
match of the first parser, otherwise callback is ignored.

**Note:** The third parser, aka Until parser, consumes the text upon
successful match.

Maybe
-----

```go
    func Maybe(callb Nodify, parser interface{}) Parser {
```

Accepts a parser that can either match or does-not-match with current
input text.

Nodify callback is called with a slice of single ParsecNode element of type
``MaybeNone`` in case Maybe fails to match, otherwise ParsecNode will
correspond to the matching node.

Using the builtin scanner
-------------------------

The builtin scanner library manages the input buffer and implements a cursor
into the buffer. Create a new scanner instance,

```go
    s := parsec.NewScanner(text)
```

The scanner library supplies method receivers like ``Match(pattern)``,
``SkipAny(pattern)`` and ``Endof()``, refer to scanner.go for more information
on each of these methods.

Examples
--------

* expr/expr.go, implements a parsec grammar to parse arithmetic expressions.
* json/json.go, implements a parsec grammar to parse JSON document.

Clone the repository run the benchmark suite

```bash
    $ cd expr/
    $ go test -test.bench=. -test.benchmem=true
    $ cd json/
    $ go test -test.bench=. -test.benchmem=true
```

To run the example program,

```bash
    # to parse expression
    $ go run tools/parsec/parsec.go -expr "10 + 29"

    # to parse JSON string
    $ go run tools/parsec/parsec.go -json '{ "key1" : [10, "hello", true, null, false] }'
```

Simple html parser
------------------

```go
func makehtmly(ast *AST) Parser {
	var tag Parser

	opentag := AtomExact("<", "OT")
	closetag := AtomExact(">", "CT")
	equal := AtomExact("=", "EQUAL")
	slash := TokenExact("/[ \t]*", "SLASH")
	tagname := TokenExact("[a-z][a-zA-Z0-9]*", "TAG")
	attrkey := TokenExact("[a-z][a-zA-Z0-9]*", "ATTRK")
	text := TokenExact("[^<>]+", "TEXT")
	ws := TokenExact("[ \t]+", "WS")

	element := ast.OrdChoice("element", nil, text, &tag)
	elements := ast.Kleene("elements", nil, element)
	attr := ast.And("attribute", nil, attrkey, equal, String())
	attrws := ast.And("attrws", nil, attr, ast.Maybe("ws", nil, ws))
	attrs := ast.Kleene("attributes", nil, attrws)
	tstart := ast.And("tagstart", nil, opentag, tagname, attrs, closetag)
	tend := ast.And("tagend", nil, opentag, slash, tagname, closetag)
	tag = ast.And("tag", nil, tstart, elements, tend)
	return tag
}
```

Projects using goparsec
-----------------------

* [Monster](https://github.com/prataprc/monster), production system in golang.
* [GoLedger](https://github.com/tn47/goledger), ledger re-write in golang.

If your project is using goparsec you can raise an issue to list them under
this section.

Articles
--------

* [Parsing by composing functions](http://prataprc.github.io/parser-combinator-composition.html)
* [Parser composition for recursive grammar](http://prataprc.github.io/parser-combinator-recursive.html)
* [How to use the ``Maybe`` combinator](http://prataprc.github.io/parser-combinator-maybe.html)

How to contribute
-----------------

* Pick an issue, or create an new issue. Provide adequate documentation for
the issue.
* Assign the issue or get it assigned.
* Work on the code, once finished, raise a pull request.
* Goparsec is written in [golang](https://golang.org/), hence expected to follow the
global guidelines for writing go programs.
* If the changeset is more than few lines, please generate a
[report card](https://goreportcard.com/report/github.com/prataprc/goparsec).
* As of now, branch ``master`` is the development branch.
