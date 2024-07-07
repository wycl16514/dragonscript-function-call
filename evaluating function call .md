In this section let's see how to interprete a function call. Now we have two kinds of nodes that are related to function, one is node func_decl which is the dclaration of a function, the other is node with type of call which is calling a function.
We need to combine this two together to complete the evaluation of a function call.

Actually before introducing function, our code is just like in an anonymous function without input parameters. When we interprete the code, we begin with the root of the parsing tree which is the node of type program, then we traverse down the tree
,visit each node and trigger the visitor method of each node. It is similar when evaluating function, when we visit the func_decl node, we save the node in a map as value and the function name as key, and when we visit a call node, we take the name
of function being called, and use it to get the func_decl node from the map, then fill in the value of arguments from the code node to func_decl node and using interpreter to traverse the tree begining with the func_decl node.

The whole process just like following, when we complete parsing the code, and there is a func_decl node and a call node in the parsing tree:

![evaluating function](https://github.com/wycl16514/dragonscript-function-call/assets/7506958/cb1b798c-29d8-47d3-899b-b0648637e25c)


In the process of evaluating the code, when we visit the func_decl node, we get the name of the function and the node itself and save them in a call map:

![evaluating function1](https://github.com/wycl16514/dragonscript-function-call/assets/7506958/bc02eb5e-63b2-43fe-a8a3-cb8940cdc773)

When we visit the call node, we get the function name out, and using it to get the conrrespoinding func_decl node, then get the arguments from the code node, fill in the arguemnts to the parameters of the func_decl node, and using an interpreter
object to visit the tree with root of func_decl:

![evaluating function (1)](https://github.com/wycl16514/dragonscript-function-call/assets/7506958/3de4b9c8-0f02-4a91-8f63-287a64c49026)

Let's add the test case first as following:
```js
it("should execute given function correctly", () => {
        let code = `
        var a = 1;
        func addToa(b, c) {
            a = a + b + c;
        }
        addToa(2,3);
        print(a);
        `

        root = createParsingTree(code)
        let intepreter = new Intepreter()
        root.accept(intepreter)
        console = intepreter.runTime.console
        expect(console.length).toEqual(1)
        expect(console[0]).toEqual(6)
    })
```
Run the test and make sure it fails. Then we add code to handle it.

To be notice that, each function is just like a block, it has its own scope, codes that outside of the function body can't visit its local variables but codes in the function body can visit global varaibles that are defined outside of the 
function. That is to say the enviroment of the function is just like the local enviroment of block, it is better to go through the code then we can have a better understand for the logic here, add the following code to runtime.js:
```js
export default class RunTime {
    constructor() {
     ....
      /*
        call map just like env, since we can declare function inside a block or 
        inside the body of a function
        */
        this.callMap = [{}]
    }

 addCallMap = () => {
        this.callMap.push({})
    }

    removeCallMap = () => {
        if (this.callMap.length > 1) {
            this.callMap.pop()
        }
    }

   getFunction = (funcName) => {
        for (let i = this.callMap.length - 1; i >= 0; i--) {
            const callMap = this.callMap[i]
            if (callMap[funcName]) {
                return callMap[funcName]
            }
        }
    }

    addFunction = (funcName, funcNode) => {
        const callMap = this.callMap[this.callMap.length - 1]
        callMap[funcName] = funcNode
    }
...
}
```
We design call map as a chain just like enviroment that's because function can be declared in a local block or in function body, and when there is a function call,we need to look for the root node of 
the function from the inner most call map first just like we access a variable inside a block, a function can be called if the function is declared outside of the current block or scope. 

Now turn into intepreter.js and do the following code:
```js
fillCallParams = (funcRoot, callNode) => {
        /*
        check the function being call need parameters or not, if needed,
        evaluate the arguments and fill those arguments to the local enviroment
        of the function being call
        */
        if (funcRoot.children[0].attributes.value &&
            funcRoot.children[0].attributes.value.length) {
            //evaluate arguments
            const args = callNode.children[0]
            this.visitChildren(args)
            //get name of each parameter
            let paraNames = funcRoot.children[0].attributes.value
            for (let i = 0; i < paraNames.length; i++) {
                this.runTime.bindLocalVariable(paraNames[i], args.children[i].evalRes)
            }
        }
    }

    visitCallNode = (parent, node) => {
        //get the function name
        const callName = node.attributes.values
        if (callName !== "anonymous_call") {
            const funcRoot = this.runTime.getFunction(callName)
            if (funcRoot) {
                this.runTime.addCallMap()
                this.runTime.addLocalEnv()
                this.fillCallParams(funcRoot, node)
                //evaluate the body of the function
                funcRoot.children[1].accept(this)

                let evalRes = funcRoot.evalRes
                if (!evalRes) {
                    evalRes = {
                        type: "NIL",
                        value: "null"
                    }
                }
                node.evalRes = evalRes

                this.runTime.removeLocalEnv()
                this.runTime.removeCallMap()
            }
        } else {
            // TODO
        }

        this.attachEvalResult(parent, node)
    }

    visitArgumentsNode = (parent, node) => {
        node.parent = parent
        this.visitChildren(node)
    }

    visitFuncDeclNode = (parent, node) => {
        //add the function name and node to call map
        this.runTime.addFunction(node["func_name"], node)
    }

    visitParametersNode = (parent, node) => {
        //we dont't do anything for it
    }
```
method fillCallParams used for setting the value for arugments of the function being called, it gets the name of parameters from the node of func_decl, and evaluate the children of node call which is
the expressions for arguments, then set the evaluated results with the name of parameters in the local enviroment we set up for the function, just notice the function body is just like a local scope.

When the visitCallNode is runned, which means we begin to executed the function call. We handle named function call first , we use the function name to query the func_decl node from the call map,
and we set up the running enviroment for the function, that is setting up a local call map and local enviroment for it. We set up the local call map because there may have new function declared inside
the function body. Then we call fillCallParams to set up the arguments in the local enviroment for the function body. Function arguments are just like variables that are defined and assigned at the 
local scope.

Then we get the block node from the func_decl node which is the second child of func_decl node, and we evaluate the block by using the same intepreter. After evaluating the function body, we check
there is any returned value or not(we havn't design the function return yet), if function returns none, we set the return value to be NIL. After completing the evaluation of the function body, we
removed the local enviroment and local call map, this is just like we go out of a scope.

The method of visitArgumentsNode is used to evaluate the value that will pass to the function of being called, and when visitFuncDeclNode is called, it means we are having a functin declaration, then
we add the func_decl node to the call map as value with the function name as key.

After completing aboved code, run the test again and make sure it can be passed. Since we have function object, we need to consider 
error relates to function, let's check some test cases:
```js
it("should not allow to assign value to function name", () => {
        let code = `
        func doSomething() {print("something");}
        doSomething = 1;
        `
        const executeCode = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(executeCode).toThrow()
    })
```
We need to check the identifier being assigned is variabe not name of function, therefore we add code like following:
```js
 visitAssignmentNode = (parent, node) => {
        this.visitChildren(node)
        if (node.attributes) {
            const name = node.attributes.value
            const val = node.evalRes
            //check the name is function name or not
            if (this.runTime.getFunction(name)) {
                throw new Error("can't assign value to function name")
            }
            this.runTime.bindGlobalVariable(name, val)
        }
        this.attachEvalResult(parent, node)
    }
```
The second case we need to consider is as name collision of variable and function:
```js
 it("should not allow name collision of variable and function", () => {
        let code = `
        func doSomething() {print("something");}
        var doSomething;
        `
        let executeCode = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(executeCode).toThrow()

        code = `
        var doSomething;
        func doSomething() {print("something");}
        `
        executeCode = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(executeCode).toThrow()
    })
```
In the aboved case, we consider two cases, the first is function declaration before variable definition, the second is function declaration after variable definition, the we add the following code to
make it passes: 
```js
visitVarDeclarationNode = (parent, node) => {
    ....
      /*
        avoid name collision with function
        */
        if (this.runTime.getFunction(variableName)) {
            throw new Error("variable name collection with defined function")
        }
    ....
}

visitFuncDeclNode = (parent, node) => {
        let collisionWithVar = false
        //prevent collision with variable name
        try {
            this.runTime.getVariable(node["func_name"])
            collisionWithVar = true
        } catch (err) {
            //if there is not name collection with variable, 
            //getVariable will cause execpetion 
        }

        if (!collisionWithVar) {
            //add the function name and node to call map
            this.runTime.addFunction(node["func_name"], node)
        } else {
            throw new Error("function name collision with defined variable")
        }

    }
```
Let's see another case:
```js
 it("should not allow more than one function with the same name", () => {
        let code = `
        func doSomething() {print("something");}
        func doSomething(a,b) {print(a); print(b);}
        `
        let executeCode = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(executeCode).toThrow()
    })
```
We need to check whether the function name is already exist when we add the given function to call map as following:
```js
 visitFuncDeclNode = (parent, node) => {
    ....
 if (!collisionWithVar) {
            //check name already exist 
            if (this.runTime.getFunction(node["func_name"])) {
                throw new Error("Fuction already declared")
            }
            //add the function name and node to call map
            this.runTime.addFunction(node["func_name"], node)
        } else {
            throw new Error("function name collision with defined variable")
        }
}
```
Then we think about the mismatch between parameters in function declaration and function calling:
```js
it("should make sure arguments in calling match parameters in function declaration", () => {
        let code = `
        var a = 1;
        func doSomething() {print("something");}
        doSomething(a);
        `
        let executeCode = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(executeCode).toThrow()

        code = `
        func doSomething(a, b) {print("something");}
        var a = 1;
        doSomething(a);
        `
        executeCode = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(executeCode).toThrow()
    })
```
In the aboved case, we consider two cases, one is arguments in the calling are more than parameters in declaration, the second is arguments in calling are less than parameters in declaration. And we
need to check this when we fill in parameters for function calling:
```js
 fillCallParams = (funcRoot, callNode) => {
        //make sure arguments in calling match up to parameters in declaration
        const args = callNode.children[0]
        if (args.children.length !==
            funcRoot.children[0].attributes.value.length) {
            throw new Error("arguments in calling mismatch parameters in declaration")
        }
    ....
}
```

That's are some cases we think about now, we may add more in the futhure.Now we can add return statement for function, we can pass result
from the function body to ourside. The grammar rule for return statement is simple as following:

statement -> returnStmt

returnStmt -> RETURN return_expr 

return_expr -> SEMICOLON | expression SEMICOLON

Let's add a test case for it as following:
```js
it("should enable to parse return statement", () => {
        let code = `
        func getSomething(a, b) {return a+b;}
        `

        let parseCode = () => {
            createParsingTree(code)
        }

        expect(parseCode).not.toThrow()
    })
```
Then we add code to implement the grammar rule aboved:
```js
statement = (parent) => {
    ....
     //get return keyword then parse return statement
        token = this.matchTokens([Scanner.RETURN])
        if (token) {
            this.advance()
            this.returnStmt(stmtNode)
            parent.children.push(stmtNode)
            return
        }
    ....
}

 matchSemicolon = () => {
        let token = this.matchTokens([Scanner.SEMICOLON])
        if (token) {
            //return_expr -> SEMICOLON
            this.advance()
            return true
        }

        return false
    }

    returnStmt = (parent) => {
        const returnNode = this.createParseTreeNode(parent, "return")
        parent.children.push(returnNode)
        if (this.matchSemicolon()) {
            return
        }

        //return_expr -> expression SEMICOLON
        this.expression(returnNode)
        if (!this.matchSemicolon()) {
            throw new Error("return statement missing semicolon")
        }
    }

addAcceptForNode = (parent, node) => {
        switch (node.name) {
        ....
         case "return":
                node.accept = (visitor)=> {
                    visitor.visitReturnNode(parent, node)
                }
        }
    }
```
Since we add a new node, then we should add the visitor method to tree adjustor:
```js
 visitReturnNode = (parent, node) => {
        this.visitChildren(node)
    }
```
After completing aboved code, make sure the newly test case can be passed. The evaluation of return statement just like the continue or break statement, after executing it, all statements that 
following the return statement will be ignored by the intepreter. Let's add the test case first:
```js
 it("should evaluate return statement correctly", () => {
        let code = `
        var a = 1;
        func getSomething(b,c) {
            return b+c;
            a = 2;
        }
        var d = getSomething(1,2);
        print(d);
        print(a);
        `

        let root = createParsingTree(code)
        let intepreter = new Intepreter()
        root.accept(intepreter)
        console = intepreter.runTime.console
        expect(console.length).toEqual(2)
        expect(console[0]).toEqual(3)
        expect(console[1]).toEqual(1)

        code = `
        var a = 1;
        func doSomething() {
           a = 2;
        }
        var d = doSomething();
        print(d);
        `

        root = createParsingTree(code)
        intepreter = new Intepreter()
        root.accept(intepreter)
        console = intepreter.runTime.console
        expect(console.length).toEqual(1)
        expect(console[0]).toEqual("null")
    })
```
Run the test make sure it fails and we add code to make it pass, we will handle return just like handling break or continue, therefore in intepreter.js:
```js
export default class Intepreter {
    constructor() {
        this.runTime = new RunTime();
        //flag for continue and break
        this.isContinue = false
        this.isBreak = false

        this.inLoop = false

        this.isReturn = false
    }

    ....
     visitDeclarationRecursiveNode = (parent, node) => {
        for (const child of node.children) {
            child.accept(this)
            ....
            
            if (this.isReturn) {
                this.isReturn = false
                //stop executing after return statement
                break
            }
        }
        this.attachEvalResult(parent, node)
    }

     visitCallNode = (parent, node) => {
        //get the function name
        const callName = node.attributes.values
        if (callName !== "anonymous_call") {
            const funcRoot = this.runTime.getFunction(callName)
            if (funcRoot) {
                ....
                let evalRes = funcRoot.evalRes
                /*
                if the evalRes dosen't have is_return field, then we set the return
                value of function to nil
                */
                if (!evalRes["is_return"]) {
                    evalRes = {
                        type: "NIL",
                        value: "null"
                    }
                }
                node.evalRes = evalRes

                ....
            }
        } else {
            // TODO
        }

        this.attachEvalResult(parent, node)
    }

    visitReturnNode = (parent, node) => {
        this.isReturn = true
        if (node.children.length > 0) {
            this.visitChildren(node)
        } else {
            node.evalRes = {
                type: "NIL",
                value: "null"
            }
        }
        //this field indicated the value is returned by return
        node.evalRes["is_return"] = true
        this.attachEvalResult(parent, node)
    }
}
```
In aboved code, we set up a flag "isReturn" to indicate if the return statement is encountered or not, if it is, then the flag will set to true. When the body of function is running, all statements
inside the function body will be executed one by one, this is done in visitDeclarationRecursiveNode, which child node in the loop is one statement, after finish the execution of one statement, it 
checks whether a return statement is executed, if it is, the flag isReturn is set to true, then the loop will stop immediately.

When the function is complete of execution, we need to check the return value of the function, at before, the evaluation result of a block is the same result of the evaluation result of the last line.
This will bring confusion to us that whether the result is returned by a return statement or caused by the last line in the function body. In order to differentiate the two, when we execute the return
statement, we add a field name "is_return", this is what we have done in visitReturnNode method. In this methond it first check whehter the return node has any children, if it has which means the 
return keyword is followed by an expression, then we do evaluate the expression, get its result and use the result as the result returned by the return statement, if there is not expression following
the return keyword, we construct a nil result.

After completing the code aboved, run the test again and make sure it can be passed. The same as continue or break, we need to make sure return statement can only appear inside the body of function,
let's add a test case for it:
```js
it("should only allow return statement inside function body", () => {
        let code = `
        var a = 1;
        return a;
        var b = 2;
        `
        let codeToExecute = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }

        expect(codeToExecute).toThrow()
    })
```
Run the case and make sure it fails. Now we add code to make it passed. Just like how we do for continue and break, we use an indicator to check whether the return statement is inside a function body 
or not:
```js
export default class Intepreter {
    constructor() {
    ...
    this.inFunc = false
    }

     visitCallNode = (parent, node) => {
     ...
     if (funcRoot) {
     ...
     this.inFunc = true
     ...
      this.runTime.removeLocalEnv()
      this.runTime.removeCallMap()
      this.inFunc = false
      } else {
       //TODO
      }
     this.attachEvalResult(parent, node)
    }

visitReturnNode = (parent, node) => {
        //check is in function body
        if (!this.inFunc) {
            throw new Error("return only allowed in function body")
        }
    ...
}
```
After having the aboved code, make sure the test case can be passed. Let's think twice, the way we check return inside function body is correct enough, check about the following test case:
```js
    it("should evaluate function delcared inside function", () => {
        let code = `
        func makeCounter() {
            var i = 0;
            func count() {
                i = i+1;
            }
            count();
            return i;
        }
        var a = makeCounter();
        print(a);
        `

        let root = createParsingTree(code)
        let intepreter = new Intepreter()
        root.accept(intepreter)
        console = intepreter.runTime.console
        expect(console.length).toEqual(1)
        expect(console[0]).toEqual(1)
    })
```
Running the test will get an exception for return not inside function body, that's because when we finish the execution of the function inside another function, we set the flag isReturn to false,
but at that time, we still inside the body of the wrapping function, the way we solve it is change isReturn from bool to counter, each time we inside a function we increase the counter, and decrease
the counter when we complete the execution of function. And when we handle the return statement, we check the counter more than 0 or not, if it is more than 0 than we make sure we sitll inside the
body of function, therefore we have the following changes:
```js
constructor() {
  ....
   this.inFunc = 0
}

visitCallNode = (parent, node) => {
    ....
    if (funcRoot) {
        this.runTime.addCallMap()
        this.runTime.addLocalEnv()
        this.inFunc += 1
        ....
         this.runTime.removeLocalEnv()
         this.runTime.removeCallMap()
         this.inFunc -= 1
    }
   ...
}

 visitReturnNode = (parent, node) => {
        //check is in function body
        if (this.inFunc <= 0) {
            throw new Error("return only allowed in function body")
        }
    ....
 }
```
Run the test again and make sure it is ok. Finally let's do the evaluation for annonymous function. Annoymous function is function without
a name, it is muck like lambda or clousre, one speciality of it is it can be assigned to varaible, let's add the case first:
```js
it("should enable annonymous function to be assignable", () => {
        let code = `
            var a = func (b,c) {
                return b+c;
            }
            var b = a(1,2);
            print(b);
        `
        let root = createParsingTree(code)
        let intepreter = new Intepreter()
        root.accept(intepreter)
        console = intepreter.runTime.console
        expect(console.length).toEqual(1)
        expect(console[0]).toEqual(3)
    })
```
Run the case and make sure it fails. In order to support annonymous, we need to enable function declaration withou a name, therefore we
make the following changes in parser:
```js
 funcDecl = (parent) => {
        const funcDeclNode = this.createParseTreeNode(parent, "func_decl")
        parent.children.push(funcDeclNode)
        //we do function rule in here
        let token = this.matchTokens([Scanner.IDENTIFIER])
        //enable function declaration without a name
        if (token && token.lexeme) {
            //over function name
            this.advance()
            funcDeclNode["func_name"] = token.lexeme
        }
       ...
}
```

In aboved code, we allow the name of function to be empty.In order to make function to be assignable, 
we need change in the rule for primary, and we make the changes as following:
```js
primary = (parentNode) => {
        //add an identifier
        //primary -> NUMBER | STRING | "true" | "false" | "nil" | "(" expression ")" |IDENTIFIER| epsilon
        //enable function to be assignable
        const token = this.matchTokens([Scanner.NUMBER, Scanner.STRING,
        Scanner.TRUE, Scanner.FALSE, Scanner.NIL,
        Scanner.LEFT_PAREN, Scanner.IDENTIFIER,
        Scanner.FUNC
        ])
        if (token === null) {
            //primary -> epsilon
            return false
        }

        const primary = this.createParseTreeNode(parentNode, "primary")
        /*
        if it is left parenthese, then we set the node value to grouping
        and append the expression as the child node of primary
        */
        //primary -> "(" expression ")"
        if (token.token === Scanner.LEFT_PAREN) {
            //over left paren
            this.advance()

            this.expression(primary)
            if (!this.matchTokens([Scanner.RIGHT_PAREN])) {
                throw new Error("Miss matching ) in expression")
            }

            primary.attributes = {
                value: "grouping",
            }
            //scann over )
            this.advance()
        }
        //check it is assignning function or not
        else if (token.token === Scanner.FUNC) {
            //over keyword func
            this.advance()
            this.funcDecl(primary)
        }
        else {
            primary.attributes = {
                value: token.lexeme,
            }
            primary.token = token
            this.advance()
        }

        parentNode.children.push(primary)
        return true
    }
```
In the aboved code, we do the parsing of function declaration as a child node of primary node, then it can be assigned just like number
or string can be assign to variable. Then when the intepreter visit the node of func_decl, we will return an evaluation result with type
func and value as the node itself. Since primary node can have child which is func_decl node this time, we need to readjust its child in tree adjustor in tree_adjustor_visitor.js:
```js
   visitPrimaryNode = (parent, node) => {
        /*
        it is possilbe it will contain func_decl node, and need to readjust its 
        parsing tree
        */
        this.visitChildren(node)
    }
```

now its time to change the code in intepreter.js:

```js
visitFuncDeclNode = (parent, node) => {
    ...
     //construct evaluation result of this node
        node.evalRes = {
            type: "func",
            value: node,
        }
        this.attachEvalResult(parent, node)
}
```
Now when we have a function code, the name of the calling function it is likely to be an identifier instead of function name, therefore
when handling function calling, we need to check the name in call map and in local enviroment:
```js
 visitCallNode = (parent, node) => {
        //get the function name
        const callName = node.attributes.values
        if (callName !== "anonymous_call") {
            let funcRoot = this.runTime.getFunction(callName)
            if (!funcRoot) {
                //the name of calling may be an identifier
                //since annonymous function can be assigned to identifier
                const val = this.runTime.getVariable(callName)
                funcRoot = val.value
            }
         ...
      }
    ...
}
```

And because function can be assignable, then in primary we need to construct the assign value:
```js
visitPrimaryNode = (parent, node) => {
    ....
    switch (token.token) {
            case Scanner.FUNC:
                type = "func"
                value = node.children[0]
                break
     ...
    }
     ....
}
```
After completing aboved code, try to run the test and make sure it passed. We bring in new thing which is enable function call base on variable, but if we are calling a variable which
is not assigned with function, then it should be error, as the following case:
```js
it("should not allow function call on normal variable", () => {
        let code = `
            var a = 1;
            a(1,2);
        `
        let codeToExecute = () => {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }
        expect(codeToExecute).toThrow()
    })
```
Run the aboved case you may find it passes. But the code causes the intepreter to break down which is not our intention, there we need to add code to prevent it as following:
```js
 visitCallNode = (parent, node) => {
        //get the function name
        const callName = node.attributes.values
        if (callName !== "anonymous_call") {
            let funcRoot = this.runTime.getFunction(callName)
            if (!funcRoot) {
                //the name of calling may be an identifier
                //since annonymous function can be assigned to identifier
                const val = this.runTime.getVariable(callName)
                //need to make sure the variable assigned with function
                if (val.type !== "func") {
                    throw new Error("can not call on variable never assigend with function")
                }
                funcRoot = val.value
            }
        ...
        }
    ...
}
```
By changing aboved code, we can make sure our intepreter can throw out exception instead of breaking down. Finally let's see how to handle annoymous function chain call as following:
```js
it("should evaluate annonymous function chain call", () => {
        let code = `
        var a = 1;
        var b = func (num) {
            a = a + num;
            return func (num1, num2) {
                a = a + num1 + num2;
                return func(num3) {
                    return a + num3;
                };
            };
        };

        b(1)(2,3)(4);
        print(a);
        `

        let root = createParsingTree(code)
        let intepreter = new Intepreter()
        root.accept(intepreter)
        console = intepreter.runTime.console
        expect(console.length).toEqual(1)
        expect(console[0]).toEqual(11)
    })

```

No wander the code like "b(1)(2,3)(4);" it is very very urgly, but it is allow, no matter how good you design your language, there are always someone to write urgly shit code.Let's see
how we can evaluate the code. It is better to check its parsing tree first, try running the following command at the web console:
```js
recursiveparsetree var a=1; var b=func(num) {a = a+num; return func (num1, num2){a = a+num1+num2; return func(num3){ return a+ num3;}; };};
```
The structure of the tree is crazy complex, I don't know why should we come out this idear, it is just like shooting on my head instead of my feet, following is the key part of the tree:


![截屏2024-07-08 00 03 09](https://github.com/wycl16514/dragonscript-function-call/assets/7506958/4cea7f15-03c7-4249-a341-9ad6735efecc)

The key here is the first call has a name b, then it has two descendant call with name anonymous_call, this will help us to do the evaluation, in runtime.js, we do the following:
```js
export default class RunTime {
    constructor() {
    ....
    this.returned_call = undefined
    }
getAnonymousCall = () => {
        const funcRoot = this.returned_call
        this.returned_call = undefined
        return funcRoot
    }

    addAnonymousCall = (funcRoot) => {
        this.returned_call = funcRoot
    }
```
In aboved code, we use the field returned_call to save the root node of function that is being returned, and the getAnonymousCall will return this object and set it back to undefined, and 
addAnonymousCall will used to set this field.

in intepreter.js we change the code as following:
```js
visitCallNode = (parent, node) => {
        //get the function name
        const callName = node.attributes.values
        let funcRoot = undefined
        if (callName !== "anonymous_call") {
            funcRoot = this.runTime.getFunction(callName)
        } else {
            funcRoot = this.runTime.getAnonymousCall()
        }

        if (!funcRoot) {
            //the name of calling may be an identifier
            //since annonymous function can be assigned to identifier
            const val = this.runTime.getVariable(callName)
            //need to make sure the variable assigned with function
            if (val.type !== "func") {
                throw new Error("can not call on variable never assigend with function")
            }
            funcRoot = val.value
        }

        if (funcRoot) {
            this.runTime.addCallMap()
            this.runTime.addLocalEnv()
            this.inFunc += 1

            this.fillCallParams(funcRoot, node)
            //evaluate the body of the function
            funcRoot.children[1].accept(this)

            let evalRes = funcRoot.evalRes
            /*
            if the evalRes dosen't have is_return field, then we set the return
            value of function to nil
            */
            if (!evalRes["is_return"]) {
                evalRes = {
                    type: "NIL",
                    value: "null"
                }
            }
            node.evalRes = evalRes
            //if the returned is function add it to run time
            if (evalRes.type === "func") {
                this.runTime.addAnonymousCall(evalRes.value)
            }

            this.runTime.removeLocalEnv()
            this.runTime.removeCallMap()
            this.inFunc -= 1
        }


        this.attachEvalResult(parent, node)
    }

```
In the aboved code, when we get returned value from a function call, we need to check whether it is returning a function, if it is, we add the root node of the given function to run time, and
when we visit a call node and the name of the node is anoymous_call, then we get the root node from run time. After having the code aboved, run the test case and make sure it passes.

It is time to look at some error case for function chain call:
```js
 it("should not allow chain call if return is not function", ()=> {
        let code = `
        var a = func (b) {return b+1;};
        a(1)();
        `

        let codeToExecute = ()=> {
            let root = createParsingTree(code)
            let intepreter = new Intepreter()
            root.accept(intepreter)
        }

        expect(codeToExecute).toThrow()
    })
```
Run the case aboved and make sure it can be passed, and let's try the last case:
```js
 it("should allow returned function to be assigned", () => {
        let code = `
        var a = func (b) {return func (c) {return c+b;};};
        var b = a(1);
        var c = b(2);
        print(c);
        `

        let root = createParsingTree(code)
        let intepreter = new Intepreter()
        root.accept(intepreter)
        console = intepreter.runTime.console
        expect(console.length).toEqual(1)
        expect(console[0]).toEqual(3)
    })
```
I randomly come out with this case and I though it should be easy to make it passes, but when I dive deep into it, I found it is totally a new difficult topic, let's leaving it for next charpter!
