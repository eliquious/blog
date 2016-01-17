+++
date = "2016-01-16T20:02:44-06:00"
title = "Writing a database in Go: Part 2"
description = "Lexing"
tags = [ "Go", "Databases", "Lexical analysis" ]
+++

</br>

This is a continuation of the [first part][1] of this series where we discussed the data model and query language for this new database. In this second part we will setup our lexer.

### Lexing

[Lexing][2] is the process for transforming a textual input into tokens representing each keyword, character or puncuation. Fortunately, there's little we have to do for lexing the input due to a [library][3] I have written previous.

The library already handles a large majority of the syntax of the query language we designed previously. However, we will need to create the keyword tokens. We revist the first post and simply record each keyword as a constant.

```
package lexer

import (
    "github.com/eliquious/lexer"
)

const (
    // Starts the keywords with an offset from the built in tokens so our new tokens don't overlap.
    startKeywords lexer.Token = iota + 1000

    // CREATE starts a CREATE KEYSPACE query.
    CREATE

    // SELECT starts a SELECT FROM query.
    SELECT

    // DELETE deletes keys from a keyspace.
    DELETE

    // DROP deletes an entire keyspace.
    DROP

    // FROM specifies which keyspace to select and delete from.
    FROM

    // WHERE allows for key filtering when selecting or deleting.
    WHERE

    // KEYSPACE signifies what is being created.
    KEYSPACE

    // WITH is used to prefix query options.
    WITH

    // KEY signifies a key attribute follows.
    KEY

    // KEYS signifies several key attributes follow.
    KEYS
    endKeywords

    // Separates the keywords from the conditionals
    startConditionals

    // BETWEEN filters an attribute by two values.
    BETWEEN
    endConditionals
)

```

Now that the tokens are created we need to load them into the lexer. This is super easy to do. We just create a map of the tokens we want to add and call `lexer.LoadTokenMap()`.

```
func init() {
    // Loads keyword tokens into lexer
    lexer.LoadTokenMap(tokenMap)
}

var tokenMap = map[lexer.Token]string{
    CREATE:   "CREATE",
    SELECT:   "SELECT",
    DELETE:   "DELETE",
    DROP:     "DROP",
    FROM:     "FROM",
    WHERE:    "WHERE",
    KEYSPACE: "KEYSPACE",
    WITH:     "WITH",
    KEY:      "KEY",
    BETWEEN:  "BETWEEN",
}

```


#### Testing the lexer

Now we need to test to see if the keywords have been loaded and the lexer can lex our input. To test the lexer we can write a short program to read from `os.Stdin` and print the tokens as well as their locations.

```
package main

import (
        "flag"
        "fmt"
        "os"
        "strconv"

        "github.com/eliquious/lexer"
        _ "github.com/eliquious/prefixdb/lexer"
)

var skipWhitespace = flag.Bool("w", false, "skip whitespace tokens")

func main() {
        flag.Parse()
        l := lexer.NewScanner(os.Stdin)
        for {
                tok, pos, lit := l.Scan()

                // exit if EOF
                if tok == lexer.EOF {
                        break
                }

                // skip whitespace tokens
                if tok == lexer.WS && *skipWhitespace {
                        continue
                }

                // Print token
                if len(lit) > 0 {
                        fmt.Printf("[%4d:%-3d] %10s - %s\n", pos.Line, pos.Char, tok, strconv.QuoteToASCII(lit))
                } else {
                        fmt.Printf("[%4d:%-3d] %10s\n", pos.Line, pos.Char, tok)
                }
        }
}
```

Now that we have a simple tool, let's test some queries.

#### Lexer Output

`CREATE KEYSPACE`

```
echo "CREATE KEYSPACE users WITH KEY username" | ./lexer -w=true
```

```
[   0:0  ]     CREATE
[   0:7  ]   KEYSPACE
[   0:16 ]      IDENT - "users"
[   0:22 ]       WITH
[   0:27 ]        KEY
[   0:31 ]      IDENT - "username"
```

`SELECT`

```
echo "SELECT FROM users WHERE username = 'eliquious'" | ./lexer -w=true
```

```
[   0:0  ]     SELECT
[   0:7  ]       FROM
[   0:12 ]      IDENT - "users"
[   0:18 ]      WHERE
[   0:24 ]      IDENT - "username"
[   0:33 ]          =
[   0:34 ]    TEXTUAL - "eliquious"
```

I could show all the queries, but I'll let you run the rest of them. Setting up the lexer didn't take much time at all, but the parser and AST nodes will take a while longer. We'll tackle those next time.


[1]: https://eliquious.github.io/post/writing-a-database-part-1/ "Part 1: Data model and query language"
[2]: https://en.wikipedia.org/wiki/Lexical_analysis/ "Wikipedia: Lexical Analysis"
[3]: https://github.com/eliquious/lexer
