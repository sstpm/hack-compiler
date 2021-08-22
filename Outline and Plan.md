### Purpose
Compile Jack files into an intermediary representation (IR), then translate the
IR into hack assembly (handled by the hack assembler separately.)

#### Program Design ####
The overall program is broken into a parser/tokenizer, and a translator.
The parser/tokenizer splits the input into tokens that can be understood by the
translator, which then outputs Hack VM Code (the IR) for the assembler.

A token is an enumeration over various kinds of information the tokenizer/parser
could output:
- KEYWORD: one of the reserved keywords followed by some identifer (class, var, etc)
- IDENTIFER: some arbitrary name
- SYMBOL: one of + - * & | ~ < > / = ; , { [ ] } ( )
- INT_CONST: An integer, eg 56
- STR_CONST: A string, eg "Example"
- KEYWORD_CONST: A reserved word not followed by an identifer, eg "true, false"

The tokenizer will go through the input file, character by character, grouping
each set of characters into one of the above token categories. The tokenizer
will ignore anything between /* and */, and will ignore anything after "//" is
found. These symbols denote a block comment and a line-comment, respectively.

The code translator then takes each token and generates the correct hack VM code
for that token. 

### Python Implimentation Details
The python implimentation of the tokenizer is as follows:
- Have a Token class, consisting of a Value and a TokenKind
    - Value is always a string, or nothing
    - Value is present if the token type is one that carries data
    - str_const("Hello"), etc
- Have a TokenKind class, which is an Enum of token types

#### Tokenizer
- Tokenizer keeps track of the Tokens via a list of Token Instances
- Tokenizer marches through a line of a file, character by character.
##### Tokenize()
- The lines of a file are santized of control characters before any parsing
- Tokenizer tracks consumed characters via another list
- Tokenizer ignores anything between /* and */, and anything after //
    - Achieved via a continue on a for-loop
- Nested if-statements within the for loop determine behaviour
- Special case for string_constants:
    - If " is encountered, add everything next seen to the list of consumed_characters
    - When another " appears: 
        - append " to consumed_characters
        - take all consumed characters and create a str_constant token
    - Empty consumed_chars and continue parsing
- Otherwise:
    - If the current character is a space: 
        - if consumed_chars is not empty:
            - Create the token from consumed_chars
        - empty consumed_chars in either case because consumed_chars contains either a token or a space
    - elif the current character is in the list of symbols:
        -  if consumed_chars is not empty, create a new token from it
        - always create a token from the encountered symbol
        - always empty consumed_chars
    - else append the current character to consumed_chars

##### create_token(char_list)
- Join char_list into a string called value
- If value is in a list of keyword, symbols, or keyword_constants, create a new instance of Token(kind, value)
- If value has " in it, create Token(TokenKind::StrConst, value.strip("))
- If the first element of value is a number, create Token(TokenKind::IntConst, int(value))
    - be sure to catch the error resulting from something like 0NotANumber95
- Otherwise the token is Token(TokenKind::Identifier, value)

#### Compiler
- Given the filename, so as to create the corrosponding output file
- Given the list of tokens from the tokenizer
- Creates a new SymbolTable
- Tracks the classname, instanced to an empty string
- Tracks the subroutineName, instanced to an empty string.
- Tracks the number of /while/ keywords encountered, for label generation. Starts at 0.
- Tracks the number of /if/ keywords encountered, for label generation. Starts at 0.

##### eat_token()
- Returns the first token from the list of tokens
- Modifies the list of tokens to omit the first token

##### compile_class()
- Starts the compilation, which in Jack is always a class
- Eats a token, which should always be the "class" keyword
- Eats a token, setting self.className to returned value
- Eats a token, corrosponding to the opening paren {
- Eats a token, setting current_token to the value returned
- while-loops against current_token matching either "static" or "field":
    - call compile_classVarDec(current_token)
    - eat a token, setting current_token to the value returned
- while-loops against current_token matching "function", "method", "constructor":
    - call compile_subroutineDec(current_token)
    - eat a token, setting current_token to the value returned
- End; returns nothing

##### compile_classVarDec(static_field)
- Always takes Jack form "static KEYWORD IDENTIFIER(S);" or "field KEYWORD IDENTIFIER(S);"
- Generates no VM code
- Eats a token, storing the result in "type" variable
- Eats a token, storing the result in "varName" variable
- Defines a new symbolTable entry of varName.value, type.value, static_field.value
- Eats a token, setting current_token to be the result
- while-loops against current_token != ';'
    - if current_token is ','then do nothing
    - otherwise, define a new symbolTable entry of current_token.value, type_token.value, static_field.value
    - always eat a token, setting current_token to the result
- End; returns nothing

##### compile_subroutineDec(function_method_constructor):
- Statements encountered here are of the form: 
    - (function | method | constructor) (void | type) subroutineName ( (parameterList) ) { body }
- Passed token is the first field
- Eat a token, storing result in void_or_type
- Eat a token, storing result in current_token
- Set self.subroutineName to {self.className}.{current_token.value}
- Eat a token
- Eat a token, setting current_token to the returned value
- call symbol_table.startSubroutine()
- if function_method_constructor.value is "method", define a new entry in the symbolTable of "this", self.className, "arg"
- call compile_parameterList(currnet_token)
- define n_args = symbolTable.var_count("arg")
- define n_fields = symbolTable.var_count("field")
- call compile_subroutineBody(function_method_constructor, n_fields)
- End; returns nothing.

##### compile_parameterList(param1)
- Compiles the list of parameters passed to a subroutine
- Stores the names in the symbolTable
- compares param1.value to ')':
    - if true, do nothing and end
    - else:
        - set current_token = eat_token()
        - define symbolTable entry with current_token.value, param1.value, "arg"
        - set current_token = eat_token()
        - while-loop current_token.value != ')':
            - set current_token = eat_token()
            - set arg_type = current_token.value
            - advance current_token
            - define new symbolTable entry with current_token.value, arg_type, "arg"
            - advance current_token
- End; return nothing

##### compile_subroutineBody(subroutineType, n_args)
- form of { varDec* statements }
- Advances current_token twice
- while current_token == 'var'
    - call compile_varDec(current_token)
    - advance current_token
- set nvars = symbolTable.var_count("var")
- call writer.write_function(subroutineName, nvars)
- match subroutineType.value
    - constructor: call writer.write_push("constant", n_args)
                   call writer.write_call("Memory.alloc", "1")
                   call writer.write_pop("pointer", 0)
    - method:   call writer.write_push("arg", 0)
                call writer.write_pop("pointer", 0)
- Functions aren't handled separately.
- While-loop against current_token.value != '}'
    - call compile_statements(current_token)
    - advance current_token
- End; returns nothing.

##### compile_varDec(keyword)
- form of var Type varName (, Type varName )* ;
- advance current token
- store var_type = current_token.value
- advance current_Token
- store var_name = current_token.value
- define a new symbolTable entry (var_name, var_type, "var")
- Advance current_token
- while over current_token.value == ','
    - advance current_token
    - set var_name to current_token.value
    - Define symbolTable entry (var_name, var_type, "var")
    - Advance current token
- End; return nothing