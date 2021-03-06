Overview
========

The fslex.exe tool is a lexer generator for byte and Unicode character input. It follows a similar specification to the 'ocamllex' tool, though it is a reimplementation and supports some different features. See http://caml.inria.fr/pub/docs/manual-ocaml/manual026.html

Getting Started
---------------

Install the FsLexYacc nuget package.

MSBuild support
---------------

The nuget package includes MSBuild support for FsLex and FsYacc. You must add a FsLexYacc.targets reference 
to your project file manually like this (adjust the nuget package number if needed):

```
 <Import Project="..\packages\FsLexYacc.6.0.0\bin\FsLexYacc.targets" />
 ```

You must also add [FsLex andd FsYacc entries] like this:

```
    <FsYacc Include="..\LexAndYaccMiniProject\Parser.fsy">
      <OtherFlags>--module Parser</OtherFlags>
    </FsYacc>
    <FsLex Include="..\LexAndYaccMiniProject\Lexer.fsl">
      <OtherFlags>--unicode</OtherFlags>
    </FsLex>
```

Sample input
------------

This is taken from the 'Parsing' sample previously in the F# distribution. See below for information on 'newline' and line counting.
```
 let digit = ['0'-'9']
 let whitespace = [' ' '\t' ]
 let newline = ('\n' | '\r' '\n')


 rule token = parse
 | whitespace        { token lexbuf }
 | newline        { newline lexbuf; token lexbuf }
 | "while"        { WHILE }
 | "begin"        { BEGIN }
 | "end"                { END }
 | "do"                { DO }
 | "if"                { IF }
 | "then"                { THEN }
 | "else"                { ELSE }
 | "print"        { PRINT }
 | "decr"         { DECR }
 | "("            { LPAREN }
 | ")"            { RPAREN }
 | ";"                { SEMI }
 | ":="                { ASSIGN }
 | ['a'-'z']+     { ID(lexeme lexbuf) }
 | ['-']?digit+       { INT (Int32.Parse(lexeme lexbuf)) }
 | ['-']?digit+('.'digit+)?(['e''E']digit+)?   { FLOAT (Double.Parse(lexeme lexbuf)) }
 | eof   { EOF }
```


More than one lexer state is permitted - use
```
  rule state1 = 
   | "this"    { state2 lexbuf }
   | ...
  and state2 = 
   | "that"    { state1 lexbuf }
   | ...

```

States can be passed arguments:
```
  rule state1 arg1 arg2 = ...
   | "this"    { state2 (arg1+1) (arg2+2) lexbuf }
   | ...
  and state2 arg1 arg2 = ...
   | ...
```


Using a lexer

If in the first example above the constructors 'INT' etc generate values of type 'tok' then the above generates a lexer with a function
```
  val token : LexBuffer<byte> -> tok
```


Once you have a lexbuffer you can call the above to generate new tokens. Typically you use some methods from Microsoft.FSharp.Text.Lexing to create lex buffers, either a LexBuffer<byte> for ASCII lexing, or LexBuffer<char> for Unicode lexing.

Some ways of creating lex buffers are by using:
```
  LexBuffer<_>.FromChars  
  LexBuffer<_>.FromFunction
  LexBuffer<_>.FromStream
  LexBuffer<_>.FromTextReader
  LexBuffer<_>.FromArray
```


Within lexing actions the variable "lexbuf" is in scope and you may use properties ono the LexBuffer type such as:
```
 lexbuf.Lexeme  // get the lexeme as an array of characters or bytes
 LexBuffer.LexemeString lexbuf // get the lexeme as a string, for Unicode lexing
```

Lexing positions give locations in source files (the relevant type is Microsoft.FSharp.Text.Lexing.Position).

Generated lexers are nearly always used in conjunction with parsers generarted by fsyacc (also documented on this site). See the Parsed Language starter template.

 Command line options
```
fslex <filename>
        -o <string>: Name the output file.

        --codepage <int>: Assume input lexer specification file is encoded with the given codepage.

        --light: (ignored)

        --light-off: Add #light "off" to the top of the generated file

        --lexlib <string>: Specify the namespace for the implementation of the lexer table interperter (default Microsoft.FSharp.Text.Lexing)

        --unicode: Produce a lexer for use with 16-bit unicode characters.

        --help: display this list of options

        -help: display this list of options
```

Positions and line counting in lexers

Within a lexer lines can in theory be counted simply by incrementing a global variable or a passed line number count:
```
  rule token line = ...
   | "\n" | '\r' '\n'    { token (line+1) }
   | ...
```


However for character positions this is tedious, as it means every action becomes polluted with character counting, as you have to manually attach line numbers to tokens. Also, for error reporting writing service it is useful to have to have position information associated held as part of the state in the lexbuffer itself.

Thus F# follows the ocamllex model where the lexer and parser state carry 'position' values that record information for the current match (lex) and the l.h.s/r.h.s of the grammar productions (yacc).

The information carried for each position is:

 * a filename 
 * a current 'absolute' character number 
 * a placeholder for a user-tracked beginning-of-line marker 
 * a placeholder for a user-tracked line number count. 

Passing state through lexers
---------------------------

It is sometimes under-appreciated that you can pass arguments around between lexer states. For example, in one example we used imperative state to track a line number.
```
 let current_line = ref 0
 let current_char = ref 0
 let set_next_line lexbuf = ..

 ...
 rule main = parse
   | ...
   | "//" [^ '\n']* '\n' {
        set_next_line lexbuf; main lexbuf
     }
```

This sort of imperative code is better replaced by passing arguments:
```
 rule main line char = parse
   | ...
   | "//" [^ '\n']* '\n' {
        main (line+1) 0 lexbuf
     }
```


A good example is that when lexing a comment you want to pass through the start-of-comment position so that you can give a good error message if no end-of-comment is found. Or likewise you may want to pass through the number of nested of comments.
