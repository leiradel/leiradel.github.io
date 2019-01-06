---
layout: post
title: Auto-vectorization for the Masses (part 3 of n) - Semantic validation of SimpleC Programs
---

In [my last post on the Auto-vectorization for the Masses series](http://altdevblogaday.org/2011/05/20/auto-vectorization-for-the-masses-part-2-of-n-construction-of-the-ast/) we saw how to build an abstract syntax tree of a program written in SimpleC. But until now our SimpleC parser doesn't complain about semantic problems in the source code, only syntactic ones. In this post we'll add semantic validation to our SimpleC grammar so we can be sure our syntactic tree represents a correct program we can then analyze, optimize and transform into SIMD code.

## Semantic Validation

A programming language is not only its syntax. Without semantic validation, the following source code is correctly parsed despite of being incorrect:

```c
static float fat( float x )
{
  if ( y <= 1.0 ) // error, y wasn't declared
  {
    return 1.0;
  }
  else
  {
    return x * fat( x - 1.0, x ); // error, wrong number of parameters to fat
  }
}
```

To semantically validate a SimpleC program we have to keep track of things the parser alone can't validate:

* Undeclared variables and functions
* Functions called with a wrong number of arguments
* Incorrect types used with operators

We'll also take the chance to warn the user when implicit casts can lead to overflow or loss of precision. We only support 32-bit integers and floats, so any implicit conversion from one to the other should generate an warning: integers can't represent large float values nor fractions, and floats don't have as many significant digits as integers.

To keep track of declared variables and functions, we must save them somewhere so when an identifier is found we can check if it has been declared. Since we can declare variables inside blocks nested within other blocks, we also need to scope these declarations. The class `Scope` below will hold all identifiers declared within a determined scope:

```java
package com.leiradella.sv.parser;

import java.util.HashMap;
import java.util.Map;

import com.leiradella.sv.ast.ArgDecl;
import com.leiradella.sv.ast.Function;
import com.leiradella.sv.ast.Node;
import com.leiradella.sv.ast.VarDecl;

public class Scope
{
  private Map< String, Node > vars;
  
  public Scope()
  {
    vars = new HashMap< String, Node >();
  }
  
  public void addVar( String name, Node node )
  {
    vars.put( name, node );
  }
  
  public Node getVar( String name )
  {
    return vars.get( name );
  }
  
  public boolean isDeclared( String name )
  {
    return vars.get( name ) != null;
  }
  
  public String[] getVars()
  {
    return vars.keySet().toArray( new String[ vars.size() ] );
  }
}
```

But that gives us only one scope. What we need is a stack of scopes. Every time we enter a new block, a new scope is pushed onto the stack and all variables declared within that block are added to this new scope. When we leave a block, the scope is popped from the stack and discarded so the variables declared in it will no longer be accessible. The scope stack is implemented in the SimpleC grammar.

It's interesting to note that we're creating a new scope inside the `unit` rule. This is scope is the global scope and all functions are declared there since SimpleC, like C, doesn't support functions being defined within functions. If we decide to support global variables, they'll be added in that scope too.

With the scope code in place all we have to do is to keep track of the scopes and its identifiers during the parsing, checking for duplicate identifiers, undeclared identifiers and functions being incorrectly called as we go. The SimpleC grammar with the semantic actions to check for those things is below:

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

import java.util.Stack;

import com.leiradella.sv.ast.*;
import com.leiradella.sv.parser.Scope;

COMPILER SimpleC

private Stack< Scope > stack = new Stack< Scope >();
private Function       currentFunction = null;

private Scope pushScope()
{
  return stack.push( new Scope() );
}

private void popScope()
{
  stack.pop();
}

private Scope topScope()
{
  return stack.peek();
}

private void addVar( String name, Node node )
{
  if ( topScope().isDeclared( name ) )
  {
    Error( "Duplicate identifier: " + name );
  }
  else
  {
    topScope().addVar( name, node );
  }
}

private Node getVar( String name )
{
  int numScopes = stack.size();
  
  for ( int i = numScopes - 1; i >= 0; i-- )
  {
    Node node = stack.get( i ).getVar( name );
    
    if ( node != null )
    {
      return node;
    }
  }
  
  return null;
}

private void checkVar( String name )
{
  if ( getVar( name ) == null )
  {
    Error( "Unknown identifier: " + name );
  }
}

private void checkFunctionCall( String funcName, Node call )
{
  Node func = getVar( funcName );
  
  if ( !( func instanceof Function ) )
  {
    Error( "Not a function: " + funcName );
  }
  else
  {
    Function f = (Function)func;
    CallExpr c = (CallExpr)call;
    
    if ( f.getNumArgs() != c.getNumArgs() )
    {
      Error( "Wrong number of arguments: " + funcName );
    }
    
    for ( int i = 0; i < f.getNumArgs(); i++ )
    {
      Node farg = f.getArg( i );
      Node carg = c.getArg( i );
      
      checkImplicitCast( farg, carg );
    }
  }
}

private void checkImplicitCast( Node left, Node right )
{
  if ( left.getType() != right.getType() )
  {
    if ( left.getType() == Type.INT32 )
    {
      Warn( "Implicit conversion from float to int, possible loss of precision" );
    }
    else
    {
      Warn( "Implicit conversion from int to float, possible loss of precision" );
    }
  }
}

private void checkType( Node n, Type type )
{
  if ( n.getType() != type )
  {
    Error( "Wrong type argument" );
  }
}

private void Warn( String msg )
{
  errors.Warning( t.line, t.col, msg );
}

private void Error( String msg )
{
  errors.Warning( t.line, t.col, msg );
  System.exit( 1 );
}

private boolean IsCastExpr()
{
  Token x = scanner.Peek();
  return la.val.equals( "(" ) && ( x.val.equals( "int" ) || x.val.equals( "float" ) );
}

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

SimpleC< out Node result > = unit< out result > .

unit< out Node result > (.
                          Unit unit = new Unit();
                          result = unit;
                          Node function;
                          pushScope();
                        .) =
  function< out function >   (. unit.addFunction( function ); .)
  {
    function< out function > (. unit.addFunction( function ); .)
  }
  .

function< out Node result > (.
                              Function function = new Function();
                              result = function;
                              Type type;
                              String name;
                              Node stmt;
                            .) =
  [
    "static"                (. function.setStatic( true ); .)
  ]
  [
    "inline"                (. function.setInline( true ); .)
  ]
  type< out type > id< out name > (.
                                    function.setName( name );
                                    function.setType( type );
                                    addVar( name, function );
                                    pushScope();
                                  .)
  '('
  [
    argDecl< out stmt >     (.
                              function.addArg( stmt );
                              addVar( ((ArgDecl)stmt).getName(), stmt );
                            .)
    {
      ','
      argDecl< out stmt >   (.
                              function.addArg( stmt );
                              addVar( ((ArgDecl)stmt).getName(), stmt );
                            .)
    }
  ]
  ')'                       (. currentFunction = function; .)
  '{'
  {
    statement< out stmt >   (. function.addStatement( stmt ); .)
  }
  '}'                       (.
                              currentFunction = null;
                              popScope();
                            .)
  .

argDecl< out Node result > (.
                             ArgDecl arg = new ArgDecl();
                             result = arg;
                             Type type;
                             String name;
                           .) =
  type< out type >
  id< out name > (.
                   arg.setName( name );
                   arg.setType( type );
                 .)
  .

id< out String result > = (. result = la.val; .) ident .

type< out Type result > (. result = null; .) =
    "int"   (. result = Type.INT32;   .)
  | "float" (. result = Type.FLOAT32; .)
  .

statement< out Node result > (. result = null; .) =
    compoundStmt< out result >
  | varDecls< out result >
  | ifStmt< out result >
  | forStmt< out result >
  | doStmt< out result >
  | whileStmt< out result >
  | returnStmt< out result >
  | expression< out result > ';'
  .

compoundStmt< out Node result > (.
                                  CompoundStmt compound = new CompoundStmt();
                                  result = compound;
                                  Node stmt;
                                .) =
  '{'                     (. pushScope(); .)
  {
    statement< out stmt > (. compound.addStatement( stmt ); .)
  }
  '}'                     (. popScope(); .)
  .

varDecls< out Node result > (.
                              VarDecls vars = new VarDecls();
                              result = vars;
                              VarDecl var;
                              Type type;
                              String name;
                              Node expr;
                            .) =
  type< out type >
  id< out name >             (.
                               var = new VarDecl();
                               var.setName( name );
                               var.setType( type );
                               addVar( name, var );
                             .)
  [
    '='
    assignExpr< out expr >   (.
                               var.setValue( expr );
                               checkImplicitCast( var, expr );
                             .)
  ]                          (. vars.addVarDecl( var ); .)
  {
    ','
    id< out name >           (.
                               var = new VarDecl();
                               var.setName( name );
                               var.setType( type );
                               addVar( name, var );
                             .)
    [
      '='
      assignExpr< out expr > (.
                               var.setValue( expr );
                               checkImplicitCast( var, expr );
                             .)
    ]                        (. vars.addVarDecl( var ); .)
  }
  ';'
  .

ifStmt< out Node result > (.
                            IfStmt ifstmt = new IfStmt();
                            result = ifstmt;
                            Node stmt;
                          .) =
  "if"
  '('
  expression< out stmt >  (. ifstmt.setCondition( stmt ); .)
  ')'
  statement< out stmt >   (. ifstmt.setWhenTrue( stmt );  .)
  [
    "else"
    statement< out stmt > (. ifstmt.setWhenFalse( stmt ); .)
  ]
  .

forStmt< out Node result > (.
                             ForStmt forstmt = new ForStmt();
                             result = forstmt;
                             Node stmt;
                           .) =
  "for"
  '('
  [
    expression< out stmt > (. forstmt.setInit( stmt );      .)
  ]
  ';'
  [
    expression< out stmt > (. forstmt.setCondition( stmt ); .)
  ]
  ';'
  [
    expression< out stmt > (. forstmt.setUpdate( stmt );    .)
  ]
  ')'
  statement< out stmt >    (. forstmt.setBody( stmt );      .)
  .

doStmt< out Node result > (.
                            DoStmt dostmt = new DoStmt();
                            result = dostmt;
                            Node stmt;
                          .) =
  "do"
  statement< out stmt >  (. dostmt.setBody( stmt );      .)
  "while"
  '('
  expression< out stmt > (. dostmt.setCondition( stmt ); .)
  ')'
  ';'
  .

whileStmt< out Node result > (.
                               WhileStmt whilestmt = new WhileStmt();
                               result = whilestmt;
                               Node stmt;
                             .) =
  "while"
  '('
  expression< out stmt > (. whilestmt.setCondition( stmt ); .)
  ')'
  statement< out stmt >  (. whilestmt.setBody( stmt );      .)
  .

returnStmt< out Node result > (.
                                ReturnStmt retstmt = new ReturnStmt();
                                result = retstmt;
                                Node expr;
                              .) =
  "return"
  expression< out expr > (.
                           retstmt.setValue( expr );
                           checkImplicitCast( currentFunction, expr );
                         .)
  ';'
  .

expression< out Node result > (.
                                Expression expr = new Expression();
                                result = expr;
                                Node assign;
                              .) =
  assignExpr< out assign >   (. expr.addExpression( assign ); .)
  {
    ','
    assignExpr< out assign > (. expr.addExpression( assign ); .)
  }
  .

assignExpr< out Node result > (.
                                AssignExpr assign = new AssignExpr();
                                Node right;
                                BinaryOp binop = null;
                              .) =
  ternaryExpr< out result >
  [
    (
      '='   (.
              assign.setLeft( result );
              binop = assign;
            .)
    | "+="  (.
              assign.setLeft( result );
              binop = new AddExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    | "-="  (.
              assign.setLeft( result );
              binop = new SubExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    | "*="  (.
              assign.setLeft( result );
              binop = new MulExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    | "/="  (.
              assign.setLeft( result );
              binop = new DivExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    | "%="  (.
              assign.setLeft( result );
              binop = new RemExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    | "&="  (.
              assign.setLeft( result );
              binop = new BitwiseAndExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    | "^="  (.
              assign.setLeft( result );
              binop = new BitwiseXorExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    | "|="  (.
              assign.setLeft( result );
              binop = new BitwiseOrExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    | "<<=" (.
              assign.setLeft( result );
              binop = new ShiftLeftExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    | ">>=" (.
              assign.setLeft( result );
              binop = new ShiftRightExpr();
              binop.setLeft( result.dup() );
              assign.setRight( binop );
            .)
    )
                            (.
                              if ( !( result instanceof IdExpr ) )
                                Error( "Invalid lvalue" );
                            .)
    assignExpr< out right > (.
                              checkImplicitCast( result, right );
                              binop.setRight( right );
                              result = assign;
                            .)
  ]
  .

ternaryExpr< out Node result > (. Node whentrue, whenfalse; .) =
  orExpr< out result >
  [
    '?'
    expression< out whentrue >
    ':'
    ternaryExpr< out whenfalse > (.
                                   if ( whentrue.getType() != whenfalse.getType() )
                                     Warn( "Different types in ternary operator" );
                                   TernaryExpr ternary = new TernaryExpr();
                                   ternary.setCondition( result );
                                   ternary.setWhenTrue( whentrue );
                                   ternary.setWhenFalse( whenfalse );
                                   result = ternary;
                                 .)
  ]
  .

orExpr< out Node result > (. Node right; .) =
  andExpr< out result >
  {
    "||"
    andExpr< out right > (.
                           OrExpr or = new OrExpr();
                           or.setLeft( result );
                           or.setRight( right );
                           result = or;
                         .)
  }
  .

andExpr< out Node result > (. Node right; .) =
  bitwiseOrExpr< out result >
  {
    "&&"
    bitwiseOrExpr< out right > (.
                                 AndExpr and = new AndExpr();
                                 and.setLeft( result );
                                 and.setRight( right );
                                 result = and;
                               .)
  }
  .

bitwiseOrExpr< out Node result > (. Node right; .) =
  bitwiseXorExpr< out result >
  {
    '|'
    bitwiseXorExpr< out right > (.
                                  checkType( result, Type.INT32 );
                                  checkType( right, Type.INT32 );
                                  BitwiseOrExpr bor = new BitwiseOrExpr();
                                  bor.setLeft( result );
                                  bor.setRight( right );
                                  result = bor;
                                .)
  }
  .

bitwiseXorExpr< out Node result > (. Node right; .) =
  bitwiseAndExpr< out result >
  {
    '^'
    bitwiseAndExpr< out right > (.
                                  checkType( result, Type.INT32 );
                                  checkType( right, Type.INT32 );
                                  BitwiseXorExpr bxor = new BitwiseXorExpr();
                                  bxor.setLeft( result );
                                  bxor.setRight( right );
                                  result = bxor;
                                .)
  }
  .

bitwiseAndExpr< out Node result > (. Node right; .) =
  equalityExpr< out result >
  {
    '&'
    equalityExpr< out right > (.
                                checkType( result, Type.INT32 );
                                checkType( right, Type.INT32 );
                                BitwiseAndExpr band = new BitwiseAndExpr();
                                band.setLeft( result );
                                band.setRight( right );
                                result = band;
                              .)
  }
  .

equalityExpr< out Node result > (.
                                  BinaryOp binop;
                                  Node right;
                                .) =
  relationalExpr< out result >
  {
    (
      "==" (. binop = new EqualityExpr();   .)
    | "!=" (. binop = new InequalityExpr(); .)
    )
    relationalExpr< out right > (.
                                  binop.setLeft( result );
                                  binop.setRight( right );
                                  result = binop;
                                .)
  }
  .

relationalExpr< out Node result > (.
                                    BinaryOp binop;
                                    Node right;
                                  .) =
  shiftExpr< out result >
  {
    (
      '<'  (. binop = new LessThanExpr();     .)
    | "<=" (. binop = new LessEqualExpr();    .)
    | '>'  (. binop = new GreaterThanExpr();  .)
    | ">=" (. binop = new GreaterEqualExpr(); .)
    )
    shiftExpr< out right > (.
                             binop.setLeft( result );
                             binop.setRight( right );
                             result = binop;
                           .)
  }
  .

shiftExpr< out Node result > (.
                               BinaryOp binop;
                               Node right;
                             .) =
  addExpr< out result >
  {
    (
      "<<" (. binop = new ShiftLeftExpr();  .)
    | ">>" (. binop = new ShiftRightExpr(); .)
    )
    addExpr< out right > (.
                           checkType( result, Type.INT32 );
                           checkType( right, Type.INT32 );
                           binop.setLeft( result );
                           binop.setRight( right );
                           result = binop;
                         .)
  }
  .

addExpr< out Node result > (.
                             BinaryOp binop;
                             Node right;
                           .) =
  mulExpr< out result >
  {
    (
      '+' (. binop = new AddExpr(); .)
    | '-' (. binop = new SubExpr(); .)
    )
    mulExpr< out right > (.
                           binop.setLeft( result );
                           binop.setRight( right );
                           result = binop;
                         .)
  }
  .

mulExpr< out Node result > (.
                             BinaryOp binop;
                             Node right;
                           .) =
  castExpr< out result >
  {
    (
      '*' (. binop = new MulExpr(); .)
    | '/' (. binop = new DivExpr(); .)
    | '%' (. binop = new RemExpr(); .)
    )
    castExpr< out right > (.
                            binop.setLeft( result );
                            binop.setRight( right );
                            result = binop;
                          .)
  }
  .

castExpr< out Node result > (.
                              Type type;
                              result = null;
                            .) =
    IF ( IsCastExpr() ) '('
    type< out type >
    ')'
    castExpr< out result >  (.
                              CastExpr cast = new CastExpr();
                              cast.setType( type );
                              cast.setValue( result );
                              result = cast;
                            .)
  | unaryExpr< out result >
  .

unaryExpr< out Node result > (.
                               UnaryOp unop;
                               result = null;
                             .) =
    (
      '+' (. unop = null;             .)
    | '-' (. unop = new NegateExpr(); .)
    | '~' (. unop = new InvertExpr(); .)
    | '!' (. unop = new NotExpr();    .)
    )
    castExpr< out result > (.
                             if ( unop != null ) {
                               if ( unop instanceof InvertExpr )
                                 checkType( result, Type.INT32 );
                               unop.setValue( result );
                               result = unop;
                             }
                           .)
  | primaryExpr< out result >
  .

primaryExpr< out Node result > (.
                                 String name;
                                 Node arg;
                                 result = null;
                                 boolean isFunction = false;
                               .) =
    id< out name > (.
                     checkVar( name );
                     IdExpr id = new IdExpr();
                     result = id;
                     Node var = getVar( name );
                     if ( var instanceof ArgDecl ) id.setVar( (ArgDecl)var );
                   .)
    [
      '('          (.
                     if ( !( var instanceof Function ) )
                       Error( "Not a function: " + name );
                     isFunction = true;
                     CallExpr call = new CallExpr();
                     call.setFunction( (Function)var );
                     result = call;
                   .)
      [
        assignExpr< out arg >   (. call.addArg( arg ); .)
        {
          ','
          assignExpr< out arg > (. call.addArg( arg ); .)
        }
      ]
      ')' (. checkFunctionCall( name, call ); .)
    ]     (.
            if ( !isFunction && var instanceof Function )
              Error( "Missing arguments to " + name );
          .)
  | octalConst< out result >
  | decimalConst< out result >
  | hexadecimalConst< out result >
  | floatConst< out result >
  | '(' expression< out result > ')'
  .

octalConst< out Node result > =
  (.
    ConstantExpr constant = new ConstantExpr();
    constant.setValue( Long.parseLong( la.val, 8 ) );
    constant.setType( Type.INT32 );
    result = constant;
  .)
  oct_const .

decimalConst< out Node result > =
  (.
    ConstantExpr constant = new ConstantExpr();
    constant.setValue( Long.parseLong( la.val, 10 ) );
    constant.setType( Type.INT32 );
    result = constant;
  .)
  dec_const .

hexadecimalConst< out Node result > =
  (.
    ConstantExpr constant = new ConstantExpr();
    constant.setValue( Long.parseLong( la.val.substring( 2 ), 16 ) );
    constant.setType( Type.INT32 );
    result = constant;
  .)
  hex_const .

floatConst< out Node result > =
  (.
    ConstantExpr constant = new ConstantExpr();
    constant.setValue( la.val );
    constant.setType( Type.FLOAT32 );
    result = constant;
  .)
  float_const .

END SimpleC .
```

Note the methods we've added at the top of the grammar to deal with scoping and checking for semantic issues:

* `pushScope`: pushes a new scope onto the scope stack
* `popScope`: discards the top scope of the scope stack
* `topScope`: returns the scope on the top of the scope stack
* `addVar`: adds a new identifier to the top scope and errors out if its a duplicate. Note that not only variables but functions and their arguments are also added to scopes
* `getVar`: searches for an identifier through all scopes on the stack starting at the top scope. Returns `null` if the identifier wasn't declared
* `checkVar`: errors out if the identifier wasn't declared
* `checkFunctionCall`: checks if an identifier is a functions and if the arguments of a `CallExpr` match the function arguments
* `checkImplicitCast`: checks for a cast between different types and issues an warning if it can lead to a loss of precision
* `checkType`: checks if a node has the required type
* `Warn`: Issues an warning and continues with the parsing
* `Error`: Issues an error and terminates the parsing

The `Error` method exits the parser to avoid exceptions being thrown because of classes being wrongly casted, null pointers etc. that can occur if a semantic problem has been found but we let the parsing continue. We also have a new member variable `currentFunction` that allows us to check implicit casts of return values.

I also took the chance to reformat the grammar a little and give the variables better names. But it does not hide the fact that the grammar is getting more difficult to read, that's why I generally prefer to write parsers by hand and have a grammar only for documentation purposes.

The updated source code can be downloaded [here]({{ site.url }}/assets/2011-7-4-Auto-Vectorization-3/sv.zip).

## About the GPL

I've recently learned that companies tend to stay away from GPL tools/applications/whatever, despite the fact that the **_result_** of an user operating these software is **_not_** covered by the GPL. In example, `cc1` is a GPL'ed compiler (`gcc` is in fact a driver that calls the compiler, the assembler and the linker depending on command-line arguments), but by using it you're **_not_** obligated to release your source code nor the resulting object files under that license.

Unfortunately I'm not in a position to educate people about the GPL so I'm willing to change the license for the entire source code and the SimpleC grammar to the MIT license. The problem is, we'll be using a `BigRational` class later and the one I've found is GPL'ed, meaning that if I use it all the code must be GPL'ed. If you know of another similar class with a more liberal license please let me know.

Maybe it's a bit too much for this experiment to worry about license and corporate usage, but the more people that can play with it the better.

## Want to Help?

I'd love to have some help to test the parser. With the semantic tests added I could really use some feedback on their correctness. Please try running some SimpleC programs through the parser and let me know if you find any bugs. Thanks!

## 'Till Next Post!

Well, this is it. Sorry for a short post but it was necessary to prepare terrain for the optimizations we'll be doing on part 4. With this post the series now is:

* [Auto-vectorization for the Masses (part 1 of *n*): SimpleC and Abstract Syntax Trees](http://altdevblogaday.com/2011/05/05/auto-vectorization-for-the-masses-part-1-of-n-simplec-and-abstract-syntax-trees/)
* [Auto-vectorization for the Masses (part 2 of *n*): Construction of the AST](http://altdevblogaday.com/2011/05/20/auto-vectorization-for-the-masses-part-2-of-n-construction-of-the-ast/)
* Auto-vectorization for the Masses (part 3 of *n*): Semantic validation of SimpleC Programs (this post)
* Auto-vectorization for the Masses (part 4 of *n*): Constant folding optimization
* Auto-vectorization for the Masses (part 5 of *n*): Code generation

While you wait for part 4, take a look at the recently launched [Intel SPMD Program Compiler](http://ispc.github.com/). It's a much more robust tool that does what we're trying to accomplish here and much more, but it targets Intel processors only.

See ya!
