+++
date = "2016-01-31T20:18:01-06:00"
title = "Writing a database in Go: Part 3"
description = "Parsing"
tags = [ "Go", "Databases", "Parsing" ]

+++

<br/>

This is a continuation of the [second part][1] of this series where we discussed lexing the query language for this new database. In this third part we will write our parser.

### Parsing

We start by defining the `Parser`. A parser takes the tokens produced by the lexer and creates AST nodes which can then be used by a query executor.

For PrefixDB, our parser is  a simple struct with a `TokenBuffer`. We can also define a few helper methods to start with as well. We can create a new `Parser` from an `io.Reader` or from a string. After the parser has been created we use the `ParseStatement` function to initiate the parsing. It also reads the first token and delegates which specific parsing function to use for this query.

The `parseString` and `parseIndent` functions validate the next token as either a string or an identifier. The `scan`, `unscan`, `peekRune` and `scanIgnoreWhitespace` methods help manage the token buffer such as advancing the parser to the next token, etc..


```
package parser

import (
    "io"
    "strings"

    "github.com/eliquious/lexer"
    tokens "github.com/eliquious/prefixdb/lexer"
)

// Parser represents an PrefixDB parser.
type Parser struct {
    s *lexer.TokenBuffer
}

// NewParser returns a new instance of Parser.
func NewParser(r io.Reader) *Parser {
    return &Parser{s: lexer.NewTokenBuffer(r)}
}

// ParseString parses a statement string and returns its AST representation.
func ParseString(s string) (Node, error) {
    return NewParser(strings.NewReader(s)).ParseStatement()
}

// ParseStatement parses a string and returns a Node AST object.
func (p *Parser) ParseStatement() (Node, error) {

    // Inspect the first token.
    tok, pos, lit := p.scanIgnoreWhitespace()
    switch tok {
    case tokens.CREATE:
        return p.parseCreateStatement()
    case tokens.DROP:
        return p.parseDropStatement()
    case tokens.SELECT:
        return p.parseSelectStatement()
    case tokens.DELETE:
        return p.parseDeleteStatement()
    case tokens.UPSERT:
        return p.parseUpsertStatement()
    default:
        return nil, NewParseError(tokstr(tok, lit), []string{"CREATE", "DROP", "SELECT", "DELETE", "UPSERT"}, pos)
    }
}

// parserString parses a string.
func (p *Parser) parseString() (string, error) {
    tok, pos, lit := p.scanIgnoreWhitespace()
    if tok != lexer.STRING {
        return "", NewParseError(tokstr(tok, lit), []string{"string"}, pos)
    }
    return lit, nil
}

// parseIdent parses an identifier.
func (p *Parser) parseIdent() (string, error) {
    tok, pos, lit := p.scanIgnoreWhitespace()
    if tok != lexer.IDENT {
        p.unscan()
        return "", NewParseError(tokstr(tok, lit), []string{"identifier"}, pos)
    }
    return lit, nil
}

// scan returns the next token from the underlying scanner.
func (p *Parser) scan() (tok lexer.Token, pos lexer.Pos, lit string) { return p.s.Scan() }

// unscan pushes the previously read token back onto the buffer.
func (p *Parser) unscan() { p.s.Unscan() }

// peekRune returns the next rune that would be read by the scanner.
func (p *Parser) peekRune() rune { return p.s.Peek() }

// scanIgnoreWhitespace scans the next non-whitespace token.
func (p *Parser) scanIgnoreWhitespace() (tok lexer.Token, pos lexer.Pos, lit string) {
    tok, pos, lit = p.scan()
    if tok == lexer.WS {
        tok, pos, lit = p.scan()
    }
    return
}

// tokstr returns a string based on the token provided
func tokstr(tok lexer.Token, lit string) string {
    if tok == lexer.IDENT {
        return "IDENTIFIER (" + lit + ")"
    }
    return tok.String()
}

```

That's all we need to start writing the specific query types. Now that we have a parser we can now define the AST node interface. The `Node` interface is common for all queries.

```
type Node interface {
    NodeType() NodeType
    String() string
}
```

The interface provides a way to get a node's type as well as a method for printing the query. The `NodeType` is a simply enum and defined like so:

```
type NodeType int

const (
    CreateKeyspaceType NodeType = iota
    DropKeyspaceType
    SelectType
    UpsertType
    DeleteType
    StringLiteralType
    StringLiteralGroupType
    ExpressionType
    KeyAttributeType
    BetweenType
)
```

The AST ndoes are simple structs which fulfill the `Node` interface. Let's start with the `CREATE` query.

#### CREATE KEYSPACE

If you remember from the first part of this series, a `CREATE KEYSPACE` query might look something like this: `CREATE KEYSPACE users WITH KEYS username`.

In order to represent that query in a struct we need to store the keyspace name as well as the keys. It might look something like this:

```
type CreateStatement struct {
    Keyspace string
    Keys     []string
}

func (CreateStatement) NodeType() NodeType {
    return CreateKeyspaceType
}
```

The struct also needs the `String()` method to fully implement the Node interface, but it simply recreates the query string from the struct.

Now that we have an AST node we can parse the query. In order to fully parse the `CREATE` statement we need to define four new methods. First we need to define the top level method for the query. It inspecs the first token after the `CREATE` keyword, which should be `KEYSPACE`.

If `KEYSPACE` is found it will continue parsing the query.

```
// parseCreateStatement parses a string and returns an AST object.
// This function assumes the "CREATE" token has already been consumed.
func (p *Parser) parseCreateStatement() (Node, error) {

    // Inspect the first token.
    tok, pos, lit := p.scanIgnoreWhitespace()
    switch tok {
    case tokens.KEYSPACE:
        return p.parseCreateKeyspaceStatement()
    default:
        return nil, NewParseError(tokstr(tok, lit), []string{"KEYSPACE"}, pos)
    }
}
```

After the keywords have been consumed, we need the read the name of the keyspace and keys. Once the keyspace name is determined, we verify the `WITH KEY(S)` keywords, then the name of the keys.

```
// parseCreateKeyspaceStatement parses a string and returns a CreateKeyspaceStatement.
// This function assumes the "CREATE" token has already been consumed.
func (p *Parser) parseCreateKeyspaceStatement() (*CreateStatement, error) {
    stmt := &CreateStatement{}

    // Parse the name of the keyspace to be used
    lit, err := p.parseKeyspace()
    if err != nil {
        return nil, err
    }
    stmt.Keyspace = lit

    // Inspect the WITH token.
    tok, pos, lit := p.scanIgnoreWhitespace()
    switch tok {
    case tokens.WITH:

        // Inspect the KEY or KEYS token.
        tok, pos, lit := p.scanIgnoreWhitespace()
        switch tok {
        case tokens.KEY:
            k, err := p.parseIdent()
            if err != nil {
                return nil, err
            } else {
                stmt.Keys = append(stmt.Keys, k)
            }
        case tokens.KEYS:
            k, err := p.parseIdentList()
            if err != nil {
                return nil, err
            } else {
                stmt.Keys = k
            }
        default:
            return nil, NewParseError(tokstr(tok, lit), []string{"KEY", "KEYS"}, pos)
        }
    default:
        return nil, NewParseError(tokstr(tok, lit), []string{"WITH"}, pos)
    }

    // Verify end of query
    tok, pos, lit = p.scanIgnoreWhitespace()
    switch tok {
    case lexer.EOF:
    case lexer.SEMICOLON:
    default:
        return nil, NewParseError(tokstr(tok, lit), []string{"EOF", "SEMICOLON"}, pos)
    }

    return stmt, nil
}
```

To parse the name of the keyspace we need a new method:

```
// parseKeyspace returns a keyspace title or an error
func (p *Parser) parseKeyspace() (string, error) {
    var keyspace string
    tok, pos, lit := p.scanIgnoreWhitespace()
    if tok != lexer.IDENT {
        return "", NewParseError(tokstr(tok, lit), []string{"keyspace"}, pos)
    }
    keyspace = lit

    // Scan entire keyspace
    // Keyspaces are a period delimited list of identifiers
    var endPeriod bool
    for {
        tok, pos, lit = p.scan()
        if tok == lexer.DOT {
            keyspace += "."
            endPeriod = true
        } else if tok == lexer.IDENT {
            keyspace += lit
            endPeriod = false
        } else {
            break
        }
    }

    // remove last token
    p.unscan()

    // Keyspaces can't end on a period
    if endPeriod {
        return "", NewParseError(tokstr(tok, lit), []string{"identifier"}, pos)
    }
    return keyspace, nil
}
```

Keyspace names can be a single identifier, or several identifiers divided by a period. Identifiers can also have underscores but not spaces.

To wrap up the `CREATE` statement we need a method to parse a list of key names for the keyspace. Keys are identifiers that are optionally separated by a comma.

```
// parseIdentList returns a list of attributes or an error
func (p *Parser) parseIdentList() ([]string, error) {
    var keys []string
    tok, pos, lit := p.scanIgnoreWhitespace()
    if tok != lexer.IDENT {
        return keys, NewParseError(tokstr(tok, lit), []string{"identifier"}, pos)
    }
    keys = append(keys, lit)

    // Scan entire list
    // Key lists are comma delimited
    for {
        tok, pos, lit = p.scanIgnoreWhitespace()
        if tok == lexer.EOF {
            break
        } else if tok != lexer.COMMA {
            return keys, NewParseError(tokstr(tok, lit), []string{"COMMA", "EOF"}, pos)
        }

        k, err := p.parseIdent()
        if err != nil {t
            return keys, err
        }
        keys = append(keys, k)
    }

    // remove last token
    p.unscan()

    return keys, nil
}
```

That's all we need to parse the `CREATE` statement. The `DROP` statement is very similar to `CREATE` except for the list of keys so we're going to skip it. You can always find it in the repo if you want to see it.

#### SELECT

The `SELECT` statement is perhaps the most difficult to parse so we'll do it next. The select query looks fairly simple but parsing the `WHERE` clause might cause you greif. A sample query might look like this:

```SELECT FROM users.settings WHERE user = "eliquious" AND setting = "email";```

There are 2 main parts to the select query: the keyspace and the where clause. So our AST node looks like this.

```
type SelectStatement struct {
    Keyspace string
    Where    []Expression
}
```

We can start parsing the query by defining the top level function.

```
// parseSelectStatement parses a string and returns an AST object.
// This function assumes the "SELECT" token has already been consumed.
func (p *Parser) parseSelectStatement() (Node, error) {

    ks, where, err := p.parseFromWhere()
    if err != nil {
        return nil, err
    }

    return &SelectStatement{
        Keyspace: ks,
        Where:    where,
    }, nil
}
```

After the `SELECT` keyword we have `FROM` and then the keyspace name. After the keyspace name, we can parse the `WHERE` clause.

```
func (p *Parser) parseFromWhere() (string, []Expression, error) {
    var keyspace string
    var exprs []Expression

    // Inspect the FROM token.
    tok, pos, lit := p.scanIgnoreWhitespace()
    switch tok {
    case tokens.FROM:

        // Parse the name of the keyspace to be used
        lit, err := p.parseKeyspace()
        if err != nil {
            return "", nil, err
        }
        keyspace = lit

        // Inspect the WHERE token.
        tok, pos, lit := p.scanIgnoreWhitespace()
        switch tok {
        case tokens.WHERE:

            // Parse the WHERE clause
            exp, err := p.parseWhereClause(true, true)
            if err != nil {
                return "", nil, err
            }
            exprs = exp

        default:
            return "", nil, NewParseError(tokstr(tok, lit), []string{"WHERE"}, pos)
        }

    default:
        return "", nil, NewParseError(tokstr(tok, lit), []string{"FROM"}, pos)
    }
    return keyspace, exprs, nil
}
```

Parsing the `WHERE` is interesting and because the `UPSERT`, `DELETE` and `SELECT` queries all have similar clauses, we can reuse this method. The `UPSERT` query does not make use of `BETWEEN` or a logical `OR` so we can disable them by arguments.

```
// parseWhereClause parses a string and returns an AST object.
// This function assumes the "WHERE" token has already been consumed.
func (p *Parser) parseWhereClause(allowBetween bool, allowLogicalOR bool) ([]Expression, error) {
    var expr []Expression

    // Read expression
    exp, err := p.parseExpression(allowBetween, allowLogicalOR)
    if err != nil {
        return expr, err
    }
    expr = append(expr, exp)

OUTER:
    for {

        // Test if there is another expression
        tok, pos, lit := p.scanIgnoreWhitespace()
        switch tok {
        case lexer.EOF, lexer.SEMICOLON:
            break OUTER
        case lexer.AND:

            exp, err := p.parseExpression(allowBetween, allowLogicalOR)
            if err != nil {
                return expr, err
            }
            expr = append(expr, exp)

        default:
            return nil, NewParseError(tokstr(tok, lit), []string{"EOF", "SEMICOLON", "AND"}, pos)
        }
    }
    return expr, nil
}
```

The allowed conditions are not as full featured as a traditional relational database due to the decisions made when creating the data model. Each expression is `AND`d with the next. The only place an `OR` is allowed is inside the equality operator (`attr = "name2" OR "name2"`).

There are only 2 real types of expression operators: the equality and between operators. We can decide which expression is next via the following method. First we parse the key name, then the operator and values.

```
// parseExpression parses a string and returns an AST object.
func (p *Parser) parseExpression(allowBetween, allowLogicalOR bool) (Expression, error) {

    // Inspect the FROM token.
    tok, pos, lit := p.scanIgnoreWhitespace()
    if tok != lexer.IDENT {
        return nil, NewParseError(tokstr(tok, lit), []string{"identifier"}, pos)
    }
    ident := lit

    // Inspect the operator token.
    tok, pos, lit = p.scanIgnoreWhitespace()
    switch tok {
    case lexer.EQ:
        expr, err := p.parseEqualityExpression(ident, allowLogicalOR)
        if err != nil {
            return nil, err
        }
        return expr, nil
    case tokens.BETWEEN:
        if !allowBetween {
            return nil, &ParseError{Message: "BETWEEN not allowed", Pos: pos}
        }

        expr, err := p.parseBetweenExpression(ident)
        if err != nil {
            return nil, err
        }
        return expr, nil
    default:
        if allowBetween {
            return nil, NewParseError(tokstr(tok, lit), []string{"EQ", "BETWEEN"}, pos)
        }
        return nil, NewParseError(tokstr(tok, lit), []string{"EQ"}, pos)
    }
}
```

To parse the equality expression we define the method below. As mentioned above the equality operator can have 2 values. If after the first value, there can be an `AND` or an `OR` keyword.

```
// EqualityExpression represents a filter condition which filters based on an exact match.
type EqualityExpression struct {
    KeyAttribute string
    Value        Node
}

...

// parseEqualityExpression parses a string and returns an AST object.
func (p *Parser) parseEqualityExpression(ident string, allowLogicalOR bool) (Expression, error) {
    // expr := &EqualityExpression{KeyAttribute: ident}
    // var expr Expression

    // Parse the string value
    value, err := p.parseString()
    if err != nil {
        return nil, err
    }

    // Inspect the AND / OR token.
    tok, pos, lit := p.scanIgnoreWhitespace()
    switch tok {
    case lexer.AND, lexer.EOF:
        p.unscan()
        return EqualityExpression{KeyAttribute: ident, Value: StringLiteral{value}}, nil
    case lexer.OR:
        // Upserts cannot contain OR equality clauses.
        if !allowLogicalOR {
            return nil, &ParseError{Message: "OR not allowed", Pos: pos}
        }

        values := []string{value}

        // Parse the string value
        value, err := p.parseString()
        if err != nil {
            return nil, err
        }
        values = append(values, value)
        return EqualityExpression{KeyAttribute: ident, Value: StringLiteralGroup{Operator: OrOperator, Values: values}}, nil
    default:
        return nil, NewParseError(tokstr(tok, lit), []string{"AND", "OR"}, pos)
    }
}

// BetweenExpression represents a filter condition which filters keys between 2 given strings.
type BetweenExpression struct {
    KeyAttribute string
    Values       StringLiteralGroup
}

...

// parseBetweenExpression parses a string and returns an AST object.
func (p *Parser) parseBetweenExpression(ident string) (Expression, error) {
    expr := BetweenExpression{KeyAttribute: ident}

    // Parse the string value
    value, err := p.parseString()
    if err != nil {
        return nil, err
    }

    // Inspect the AND / OR token.
    tok, pos, lit := p.scanIgnoreWhitespace()
    switch tok {
    case lexer.AND:
        values := []string{value}

        // Parse the string value
        value, err := p.parseString()
        if err != nil {
            return nil, err
        }
        values = append(values, value)
        expr.Values = StringLiteralGroup{Operator: AndOperator, Values: values}
    default:
        return nil, NewParseError(tokstr(tok, lit), []string{"AND"}, pos)
    }
    return expr, nil
}
```

#### Errors

One thing I have left out, besides the other queries, is how errors are handled. I've defined a single error type for all parse errors which is called ironically a `ParseError`. It is fairly straight forward as it will return a string whieh shows the current token vs. what was expected.

```
// ParseError represents an error that occurred during parsing.
type ParseError struct {
    Message  string
    Found    string
    Expected []string
    Pos      lexer.Pos
}

// newParseError returns a new instance of ParseError.
func NewParseError(found string, expected []string, pos lexer.Pos) *ParseError {
    return &ParseError{Found: found, Expected: expected, Pos: pos}
}

// Error returns the string representation of the error.
func (e *ParseError) Error() string {
    if e.Message != "" {
        return fmt.Sprintf("%s at line %d, char %d", e.Message, e.Pos.Line+1, e.Pos.Char+1)
    }
    return fmt.Sprintf("found %s, expected %s at line %d, char %d", e.Found, strings.Join(e.Expected, ", "), e.Pos.Line+1, e.Pos.Char+1)
}
```

### Conclusion

This turned out the be a very long post despite not including all of the query types. However, the additional queries use the same techniques and helper methods as the `CREATE` and `SELECT` statements. If you want to checkout the rest of the parser as well as how to test one, you can find the code in the [repo][2].


[1]: https://eliquious.github.io/post/writing-a-database-part-2/ "Part 2: Lexing a query language"
[2]: http://github.com/eliquious/prefixdb/  "PrefixDB on GitHub"

