---
layout: post
title: Auto-vectorization for the Masses (part 1 of n) - SimpleC and Abstract Syntax Trees
---

In [my first post](http://altdevblogaday.org/2011/04/06/if-simd-o_o-or-how-to-vectorize-code-with-different-execution-paths-2/) at [#AltDevBlogADay](http://altdevblogaday.org/) I talked about some simple techniques to help programmers vectorize their code and benefit from the speedup provided by SIMD instructions available on most modern processors. A comment made in that post brought my attention to gcc's auto-vectorization feature and I decided to take a look at how it could remove the burden of code vectorization from programmers.

After checking the GNU documentation about gcc's auto-vectorization feature and studying its assembly output, it was clear that the feature is severely limited on what it can vectorize. It's not gcc's fault, it's just that the compiler can't do much based on the information available to it in the source code. There are alignment and size constraints that hinders gcc from vectorizing the code, and my conclusion is that it doesn't really help programmers beyond the most simple code which they could vectorize themselves anyway.

So I decided that I could do better, "better" being my own vision of what gcc can do and what I think I can do. This is the first post of a series where we'll build a auto-vectorization translator which parses C-like code and outputs C code for SSE, AltiVec and SPU. Parsing freestyle C code is a hard task so I'll concentrate on number-crunching code first which helps to reduce parser complexity and is a (the?) major application of SIMD anyway.

One benefit of doing this as a series of posts is that I can get feedback on each post to help me move along. At the end of each post I'll add some questions with the hope that people participate and help to shape this tool.

In this post I'll talk a little about what an [Abstract Syntax Tree](http://en.wikipedia.org/wiki/Abstract_syntax_tree) (AST) is and will show how we'll parse a simple C language which I'll call, well, SimpleC. My plan for the next posts is:

* Part 2 of *n*: Construction of the AST.
* Part 3 of *n*: [Constant folding](http://en.wikipedia.org/wiki/Constant_folding) optimization.
* Part 4 of *n*: Code generation.

where *n* can grow freely to accommodate things I'm not anticipating right now.

I'm considering this an experiment where everyone will be able to both teach and learn something, including me.

## Goals

The main goal is to write a source code translator that will parse input written in our SimpleC language and output the vectorized C code corresponding to that input. Subgoals include:

* SimpleC must be as identical as possible to C in what it implements so programmers don't have to learn a new language. Ideally, programs written in SimpleC could be compiled using a C compiler without changes.
* Generated code must be ready for compilation by a suitable C compiler without further editing.
* Generated code should be readable enough to allow for editing if necessary.
* Properly separation of code parsing, code optimization and code generation to make it easy to add new parsers, optimizations and target processors.

The code will be released under the GPL. I think it's a good option for this kind of tool because it doesn't cover the output of the tool (the generated code is yours) and forces people to contribute back if they change the tool.

## Abstract Syntax Tree

The Wikipedias's definition for Abstract Syntax Tree, or AST for short, is:

> In computer science, an abstract syntax tree (AST), or just syntax tree, is a tree representation of the abstract syntactic structure of source code written in a programming language. Each node of the tree denotes a construct occurring in the source code. The syntax is 'abstract' in the sense that it does not represent every detail that appears in the real syntax. For instance, grouping parentheses are implicit in the tree structure, and a syntactic construct such as an if-condition-then expression may be denoted by a single node with two branches.

While one can compile or transform source code without constructing an AST, the kind of code transformation we'll need to do in order to turn scalar code into a vectorized one will be much easier to implement if we do. Besides, it makes other types of code analysis and optimization easier, and we'll take the opportunity to do some. So in this first post we'll deal with source code parsing and AST construction.

## SimpleC

SimpleC's main object is a **unit** which is just a collection of declared functions in a source code file, much like a C compilation unit. Functions are declared exactly as in C, and can contain local variable declarations and statements just like C (or just like C++ as we'll allow variable declarations anywhere instead of only at the beginning of functions.)

The main characteristics of the first version of the language are:

* No enums, unions and structures.
* No switches.
* Only `int` and `float` data types are available, both having 32 bits.
* All variables are vectors having four values of the declared type.
* Pre and post increment/decrement will be left out.

Things that I think will be necessary or nice to have but didn't put much thought into yet are:

* Pre-processing so we can include headers with function declarations, have macros etc.
* Other data types such as 16-bit integers, booleans and unsigned integers. 16-bit integers would be great to take advantage of the 16-bit multipliers of some ISAs, or in other words, to better deal with the absence of 32-bit multipliers on these ISAs.
* Reinterpret a float as an int or vice-versa without actually converting the value. This is necessary i.e. if we need to access the bit pattern of a float to extract its exponent. In C we could use `int i = *(int*)&f;` or use an union but SimpleC won't implement pointers and structured types right now.

## Parsing Source Code

I'm not very good at compiler theory; I just know how to put up simple parsers that happen to work. To make things easier we'll use a compiler-compiler tool. These tools take a [EBNF](http://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_Form)-like grammar and output a lexical analyzer which transforms character streams into token streams, and a parser which, well, parses the token stream and produces whatever we want it to produce -- code, ASTs -- or just outputs errors and warnings resulting from the parsing. I wanted to use [ANTLR](http://www.antlr.org) but it was acting up on me so I switched to [Coco/R](http://ssw.jku.at/coco/).

The grammar for the simple C language we'll be using here is:

```
/*
Coco/R grammar for the SimpleC language.
Copyright (C) 2011 Andre Leiradella

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
*/

// This grammar is based on http://ssw.jku.at/coco/C/C.atg. Its license is
// not specified neither in the grammar itself nor in the website where the
// download link is provided. If you know the original grammar's license
// please let me know: andre AT leiradella DOT com.

COMPILER SimpleC

CHARACTERS

nz_dec_digit = "123456789" .
oct_digit    = "01234567" .
dec_digit    = "0123456789" .
hex_digit    = "0123456789abcdefABCDEF" .
alpha        = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ" .
alnum        = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789" .

TOKENS

ident       = alpha { alnum } .
oct_const   = '0' { oct_digit } .
dec_const   = nz_dec_digit { dec_digit } .
hex_const   = ( "0x" | "0X" ) hex_digit { hex_digit } .

float_const = '.' dec_digit { dec_digit } [ ( 'e' | 'E' ) [ '+' | '-' ] dec_digit { dec_digit } ] [ 'f' ]
            | dec_digit { dec_digit } '.' { dec_digit } [ ( 'e' | 'E' ) [ '+' | '-' ] dec_digit { dec_digit } ] [ 'f' ]
            | dec_digit { dec_digit } ( 'e' | 'E' ) [ '+' | '-' ] dec_digit { dec_digit } [ 'f' ]
            .

COMMENTS FROM "/*" TO "*/"
COMMENTS FROM "//" TO '\n'

IGNORE '\t' + '\r' + '\n'

PRODUCTIONS

SimpleC = unit .

unit = function { function } .

function = [ "static" ] [ "inline" ]
           type id '(' [ argDecl { ',' argDecl } ] ')'
           ( compoundStmt | ';' )
           .

argDecl = type id .

id = ident .

type = "int"
     | "float"
     .

statement = compoundStmt
          | varDecl
          | ifStmt
          | forStmt
          | doStmt
          | whileStmt
          | returnStmt
          | expression ';'
          .

compoundStmt = '{' { statement } '}' .

varDecl = type id [ '=' assignExpr ] { ',' id [ '=' assignExpr ] } ';' .

ifStmt = "if" '(' expression ')' statement [ "else" statement ] .

forStmt = "for" '(' [ expression ] ';' [ expression ] ';' [ expression ] ')' statement .

doStmt = "do" statement "while" '(' expression ')' ';' .

whileStmt = "while" '(' expression ')' statement .

returnStmt = "return" expression ';' .

expression = assignExpr { ',' assignExpr } .

assignExpr = ternaryExpr
             [
               ( '='  | "+=" | "-=" | "*="  | "/="  | "%="
               | "&=" | "^=" | "|=" | "<<=" | ">>="
               )
               assignExpr
             ]
             .

ternaryExpr = orExpr [ '?' expression ':' ternaryExpr ] .

orExpr = andExpr { "||" andExpr } .

andExpr = bitwiseOrExpr { "&&" bitwiseOrExpr } .

bitwiseOrExpr = bitwiseXorExpr { '|' bitwiseXorExpr } .

bitwiseXorExpr = bitwiseAndExpr { '^' bitwiseAndExpr } .

bitwiseAndExpr = equalityExpr { '&' equalityExpr } .

equalityExpr = relationalExpr { ( "==" | "!=" ) relationalExpr } .

relationalExpr = shiftExpr { ( '<' | "<=" | '>' | ">=" ) shiftExpr } .

shiftExpr = addExpr { ( "<<" | ">>" ) addExpr } .

addExpr = mulExpr { ( '+' | '-' ) mulExpr } .

mulExpr = castExpr { ( '*' | '/' | '%' ) castExpr } .

castExpr = '(' type ')' castExpr
           | unaryExpr
           .

unaryExpr = ( '+' | '-' | '~' | '!' ) castExpr | primaryExpr .

primaryExpr = id [ '(' assignExpr { ',' assignExpr } ')' ]
            | octalConst
            | decimalConst
            | hexadecimalConst
            | floatConst
            | '(' expression ')'
            .

octalConst = oct_const .

decimalConst = dec_const .

hexadecimalConst = hex_const .

floatConst = float_const .

END SimpleC .
```

Some notes about the grammar:

* `[` and `]` surround expressions that can occur 0 or 1 time.
* `{` and `}` surround expressions that can occur 0 or more times.
* `(` and `)` group expressions.
* `|` is the *or* operator, in the sense that only one of the operands is followed by the parser.
* The `id` and `(octal | decimal | hexadecimal | float)Const` productions aren't strictly needed now but we'll use them later.
* Operator precedence and associativity of SimpleC is implicit in the grammar.
* The grammar only has sufficient information to check the syntax of an input, we'll deal with the semantics of the language later (i.e. undeclared variables.)

To compile the grammar, download your preferred Coco/R flavor along with the Scanner.frame and Parser.frame files. I'll use the Java version in the articles and so should you if you want to try the tool. Using the Java version, save all three files into a folder along with the grammar in a file named SimpleC.atg, then type

```
$ java -jar Coco.jar SimpleC.atg
Coco/R (Nov 16, 2010)
checking
  LL1 warning in parseIf: "else" is start & successor of deletable structure
  LL1 warning in parseCastExpr: "(" is start of several alternatives
parser + scanner generated
0 errors detected
```

There are two warnings in this grammar. The first is the well-known "dangling else" problem. The parser doesn't which `if` statement the `else` keyword belongs to because `else` is preceded by a `statement` production which can be an `if`. The good thing is that the kind of parser generated by Coco/R just makes the `else` stick to the innermost `if` which is exactly what we want.

The second problem comes from the fact that the `castExpr` production starts with a `'('` or can be a `unaryExpr` production. `unaryExpr` can be a `primaryExpr`, which in its turn can also start with a `'('`. So Coco/R doesn't know which alternative is the right one and the generated parser will use the first one it finds, in this case the `'(' type ')' castExpr` alternative of the `castExpr` production which is clearly not what we want.

To solve this problem we use what Coco/R calls a Conflict Resolver. It's really just an `if` inside the grammar that helps the parser choose the right path, and it must be inserted in the place the warning complains about. This resolver will call a function that checks if the token after the left parenthesis is a type, and our `castExpr` production becomes:

```
castExpr = IF ( IsCastExpr() ) '(' type ')' castExpr
         | unaryExpr .
```

We have to add the `IsCastExpr` method to the grammar just below the `COMPILER SimpleC` statement. This method will be added to the generated parser so we must write it in Java.

```java
private boolean IsCastExpr()
{
  Token x = scanner.Peek();
  return la.val.equals( "(" ) && ( x.val.equals( "int" ) || x.val.equals( "float" ) );
}
```

To test the grammar we create a `SimpleC.java` file in the same folder as the grammar with the following contents:

```java
public class SimpleC
{
  public static void main( String[] args )
  {
    // Must have one argument with the file name we want to parse.
    if ( args.length != 1 )
    {
      System.err.println( "USAGE: java -cp . SimpleC <inputfile>" );
      return;
    }
    
    // Create a new scanner (lexical analyzer) that reads from the file.
    Scanner scanner = new Scanner( args[ 0 ] );
    // Create a new parser that pulls tokens from the scanner.
    Parser parser = new Parser( scanner );
    // Parse the input file.
    parser.Parse();
  }
}
```

Compile everything together and you'll have a program that takes an input file in SimpleC and outputs any syntax errors in it.

## 'Till Next Post!

In the next post we'll construct ASTs for inputs written in the SimpleC language. Until then I'd love to get feedback about what has been presented here, especially about the SimpleC language and grammar:

* What features are missing from SimpleC that you think are too important to be left out?
* How would you add a preprocessor to SimpleC? I'm thinking of using an external tool to do the preprocessing and feed the result to SimpleC's parser. Ideally there would exist a C-like preprocessor written in Java that I could integrate. Do you know of any?
* How important are 16-bit and unsigned integers for number-crunching? How can 16-bit integers interact with other data types with different sizes? Should we consider only 4 16-bit integers in a vector instead of 8, by manipulating the upper 16 bits of each slot, just like scalar code on 32-bit architectures do?
* How could we reinterpret a float as an integer and vice-versa without adding too much syntax to the grammar? I thought about parsing `reinterpret_cast<type>(expression)`, not as a template being instantiated but as a language construct.
* Do you have any considerations about SimpleC's grammar? Are there better ways to describe it?
* What other compiler-compiler tools do you know? Would you recommend a different one instead of Coco/R?
* Do you have strong feelings against the GPL? The license doesn't cover the generated source code so I think it's a good license that will make sure all additions to the tool are also open-source while allowing private use of the vectorized code.

I already have an idea of how to setup the AST nodes but if you have your own ideas of how the AST nodes should be written I'd also like to hear about it.

See ya!
