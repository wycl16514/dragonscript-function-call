For all programming languages, they will never miss a feature that is group a piece of code into a unit and make sure they can reuse again and again. By supporting resue of code a language can greatly enhance
the efficiency for production. The smallest reuse unit is called function which is a necessary capacity for nearly all programming language, from this section we will see how to add function support to our
script.

A function call just like following:

```js
average(1,2);
```
It begins with the name of function, follow by the left parent, then its argument list sperated by comma, we take the string prefix the left parent as "callee". Actually a function call can be complex such as:
```js
getCallback()();
```
Since we put all the string before the left paren as "callee", then we have two callees here, the first one is getCallback, the second one is getCallback(), One of the most significant feature of function call
is that the string will end up with a pair of parentheses. Let's check the grammar rule for function call first:

```js
unary -> unary_recursive | call
call -> primary  do_call
do_call -> LEFT_PARENT argument_list | EPSILON
argument_list ->  arguments RIGHT_PAREN do_call 
arguments -> expression arguments_recursive
arguments_recursive -> COMMA arguments | EPSILON
```

Let's try to do some test for the grammar above, if we have a statment like getCallback(1,2,3)(4,5,6); How the grammar aboved to parse it? The string "getCallback" will be parsed in primary and match by 
IDENTIFIER, and we go into the rule of argument_list. Since the current token is LEFT_PAREN and we match the first symbol at the right of argument_list and enter the rule of arguments. Now the input symbols
are 1,2,3 , these symbols can be matched by the arguments rule, when we first visit "1", then it will be matched by rule of expression in the right of arguments then we go into arguments_recursive, and we 
can match the COMMA at the first symbol of right of rule arguments, then repeat for symbols "2", "," , "3", when we visit RIGHT_PARENT the rule arguments_recursive -> EPSILON will apply and we goback to
RIGHT_PARENT of argument_list to do the match, and we go into argument_list_recursive to match the "(4,5,6)" again.

Let's see how to use code to implement the parsing of function, we add the test case first:
```js
describe("parsing and evaluating function calls", () => {
    it("should enable the parsing of function call", () => {
        let code = `
        getCallback(1,2,3);
        `
        let codeToExecute = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(codeToExecute).not.toThrow()

        code = `
        getCallback(1,2,3)(4,5,6);
        `
        codeToExecute = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(codeToExecute).not.toThrow()

        code = `
        getCallback(getFirstArg(), getSecondArg(), getThirdArg());
        `
        codeToExecute = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(codeToExecute).not.toThrow()
    })
})
```
We have three cases in one test, first we make sure the parser can understand the normal function call, second we wish parser can understand chaining function call, third we embedded function call in the 
arguments of a functio call. Run the test and make sure it fail. Then we can add code to parser and make sure it can pass as following:
```js
 unary = (parentNode) => {
        //unary ->   unary_recursive | call
        /*
        add function call rule, check it is beginning with "!" or "-",
        if not then go to call rule otherwise goto unary_recursive node,
        the call rule will do the primary rule for unary rule before
        */
        const unaryNode = this.createParseTreeNode(parentNode, "unary")
        if (!this.unaryRecursive(unaryNode)) {
            this.call(unaryNode)
        }
        parentNode.children.push(unaryNode)
    }

    unaryRecursive = (parentNode) => {
        //unary_recursive -> epsilon |("!"|"-") unary
        const opToken = this.matchTokens([Scanner.BANG, Scanner.MINUS])
        if (opToken === null) {
            return false
        }
        ...
    }
```
At aboved code, we change the implementation of unary and unaryRecursive, in unary we first do the rule for unary_recursive, if the current token is "!" or "-", then method unaryRecursive return true and we will not go to the rule of call. If the
current token is neither "!" nor "-" than we go to the rule of call. Remember the code before change, we do the rule of primary in unary, this time we do the primary rule in the rule of call which can make sure we don't break the parsing logic 
before when we change the rule of unary.

Then we will add code to do the rule for call and those rules following the call rule:
```js
 call = (parent) => {
        this.primary(parent)
        if (parent.children.length > 0 && parent.children[0].attributes) {
            
            parent["call_name"] = parent.children[0].attributes.value
        }

        this.do_call(parent)
    }

 do_call = (parent) => {
        if (this.matchTokens([Scanner.LEFT_PAREN])) {
            //over the beginning (
            this.advance()
            const callNode = this.createParseTreeNode(parent, "call");
            parent.children.push(callNode)
            let callName = "anonymous_call"
            /*
            The anonymous_call is for function call lie getCallBack()(), the second parenthese
            trigger another function call but this time we don't have the function name,
            therefore we use anonymous_call as its name
            */
            if (parent.call_name) {
                callName = parent.call_name
            }

            callNode.attributes = {
                values: callName,
            }
            this.argument_list(callNode)
        }
    }

 argument_list = (parent) => {
        this.arguments(parent)
        if (!this.matchTokens([Scanner.RIGHT_PAREN])) {
            throw new Error("function call missing ending right paren")
        }
        //over the ending )
        this.advance()

        this.do_call(parent)

    }

    arguments = (parent) => {
        const argumentsNode = this.createParseTreeNode(parent, "arguments")
        parent.children.push(argumentsNode)
        //check empty arguments
        if (this.matchTokens([Scanner.RIGHT_PAREN])) {
            return
        }
        while (true) {
            this.expression(argumentsNode)
            if (!this.matchTokens([Scanner.COMMA])) {
                return
            }
            //over the comma
            this.advance()
        }
    }

```
In the call method, it will do the rule of primary first, this will get the name of the function being called, for example the line "getCallback()", the primary rule will take the string "getCallback" as identifier, then this info will save
in the children of the parent node passed in. We will save this string as the name of the function being called, and save as field with name "function_name", then we go into the rule of do_call, in there we check whether the current token is 
LEFT_PAREN, if it is, then we are having an indetifier string combine with a left paren, this is signature of a function call, then we begin parsing the argument list.

Each argument can as simple as a number or string, or complicate as an expression or a function call. In each case we can reply on the rule of expression for parsing. Since arguments in function call are expressions sperated with comma, 
therefore in rule of argument_list, we goto the rule of argument to do the parsing for argument, and check whether there is a comma following, if it is, then there is another argument need to do the parsing again. If we complete the parsing
of all arguments, we need to match the right paren, then we go back to the rule of do_call again, pay attention to here, if there is another function call following such as "getCallback()()", then the second left parent after the first right
paren will be match in do call again, then we repeat the process for argument parsing.

Since we have add new nodes, then we need to add their visitor method:
```js

    addAcceptForNode = (parent, node) => {
        switch (node.name) {
        ...
          case "call":
                node.accept = (visitor) => {
                    visitor.visitCallNode(parent, node)
                }
                break
            case "arguments":
                node.accept = (visitor) => {
                    visitor.visitArgumentsNode(parent, node)
                }
                break
       ...
    }
}
```
And add visitor methods to tree adjustor:
```js
visitCallNode = (parent, node) => {
        node.parent = parent
        this.visitChildren(node)
    }

    visitArgumentsNode = (parent, node) => {
        node.parent = parent
        this.visitChildren(node)
    }
```
After completing the aboved code, let's run the command "recursiveparsetree gtbackcall(1,2,3)(4,5,6);" and check its parsing tree as following:

![截屏2024-06-28 18 24 19](https://github.com/wycl16514/dragonscript-function-call/assets/7506958/9fdaea9e-b4a0-403b-b06b-831800bc84b7)

You can see there is a node named "call" and it has value "getbackcall" as attribute, this is the name of the function being call and it has a child named "arguments", then the "arguments" node has three children which are the arguments we 
passed to the function. Then it has second child with name "call" again, this time the call node has attribute with value "anoymous_call", this indicates that the call is derived from the first call, and it has a child node "arguments", and
this child node has three children which are the arguemnts we passed to the second call.

Then run the test again and make sure the test can be passed this time. As we mentioned before, function name can only be identifier, not other objects that can be matched up by rule of primary such as
string, number, true, false, nil, then we need to make sure the name of function is correct, add the following test case first:
```js
it("should only allow indentifier as function name", () => {
        let code = `
        123(1,2,3);
        `
        let codeToParse = () => {
            createParsingTree(code)
        }
        expect(codeToParse).toThrow()


        code = `
        "hello,world!"(1,2,3);
        `
        codeToParse = () => {
            createParsingTree(code)
        }
        expect(codeToParse).toThrow()

        code = `
        true(1,2,3);
        `
        codeToParse = () => {
            createParsingTree(code)
        }
        expect(codeToParse).toThrow()

        code = `
        false(1,2,3);
        `
        codeToParse = () => {
            createParsingTree(code)
        }
        expect(codeToParse).toThrow()

        code = `
        nil(1,2,3);
        `
        codeToParse = () => {
            createParsingTree(code)
        }
        expect(codeToParse).toThrow()
    })
```
Run the test and make sure it fails. Since in the method of parsing, we attach the token object as a field for the node, then we can check the type of the token to decide currently matched token in 
primary is identifier or not, then in method do_call we do following:
```js
 do_call = (parent) => {
        if (this.matchTokens([Scanner.LEFT_PAREN])) {
            //only identifier is allowed to be name of function
            if (parent.children.length > 0 &&
                parent.children[0].token &&
                parent.children[0].token.token !== Scanner.IDENTIFIER) {
                throw new Error("function name illegal")
            }
      ....
}
```
Then run the test and make sure it can be passed. Now let's see how to declare and define function, we make test case first:
```js
it("should enable parsing function declaration", () => {
        let code = `
        func doSomeThing(a, b, c) {
            var d = a+b+c;
            print(d);
        }
        `
        let codeToParse = () => {
            createParsingTree(code)
        }
        expect(codeToParse).not.toThrow()
    })
```
Run the test and make sure it fail. The declaration of function begins with keyword func then following is the name of function which is
identifier, and left paren following the function name, and it is parameter list which is identifiers separted with comma and right paren
is the ending. After the function name and parameter list, following is a block.

Therefore we have following grammar for function declaration:

declaration_recursive -> func_decl | var_decl | statement

func_decl -> FUNC function

function -> IDENTIFIER LEFT_PAREN parameters RIGHT_PAREN block

paremeters -> identifier_list | EPSILON

identifier_list -> IDENTIFIER identifier_list_recursive

identifier_list_recursive -> COMMA identifier_list | EPSILON

Now we can add code to implement the parsing rules aboved:

```js
declarationRecursive = (parent) => {
    ....
     //if current token is FUNC, goto func_decl
        token = this.matchTokens([Scanner.FUNC])
        if (token) {
            //over keyword func
            this.advance()
            this.funcDecl(declNode)
            return
        }
    ....
}

funcDecl = (parent) => {
        const funcDeclNode = this.createParseTreeNode(parent, "func_decl")
        parent.children.push(funcDeclNode)
        //we do function rule in here
        let token = this.matchTokens([Scanner.IDENTIFIER])
        if (!token) {
            throw new Error("function declaration missing function name")
        }
        //over function name
        this.advance()
        funcDeclNode["func_name"] = token.lexeme

        token = this.matchTokens([Scanner.LEFT_PAREN])
        if (!token) {
            throw new Error("function declaration missing left paren")
        }
        //over left paren
        this.advance()
        this.parameters(funcDeclNode)

        token = this.matchTokens([Scanner.RIGHT_PAREN])
        if (!token) {
            throw new Error("function declaration missing right paren")
        }
        //over right paren
        this.advance()
        this.block(funcDeclNode)
    }

    parameters = (parent) => {
        const parametersNode = this.createParseTreeNode(parent, "parameters")
        parent.children.push(parametersNode)

        //parameter -> identifier_list | EPSILON
        const parametersList = []
        while (true) {
            let token = this.matchTokens([Scanner.IDENTIFIER])
            if (!token) {
                break
            }
            this.advance()
            //record the name of each argument
            parametersList.push(token.lexeme)
            token = this.matchTokens([Scanner.COMMA])
            if (!token) {
                break
            }
            this.advance()
        }

        parametersNode.attributes = {
            value: parametersList,
        }
    }

 addAcceptForNode = (parent, node) => {
        switch (node.name) {
        ....
         case "func_decl":
                node.accept = (visitor) => {
                    visitor.visitFuncDeclNode(parent, node)
                }
                break
            case "parameters":
                node.accept = (visitor) => {
                    visitor.visitParametersNode(parent, node)
                }
                break
    。。。。
   }
}
```
Since we add new nodes to parsing tree, we need to add visitor methods to tree adjustor:
```js
 visitFuncDeclNode = (parent, node) => {
        node.parent = parent
        this.visitChildren(node)
    }

    visitParametersNode = (parent, node) => {
        node.parent = parent
        this.visitChildren(node)
    }
```
After completing the aboved code, let's using following command to check its parsing tree first:

```js
recursiveparsetree func doSomething(a,b,c) {print(a);}
```
Then we get the following parsing tree:

![截屏2024-06-30 14 07 57](https://github.com/wycl16514/dragonscript-function-call/assets/7506958/1449f947-ad83-49f3-a7fe-98349eee79f0)

Now run the test again and make sure it passed.
