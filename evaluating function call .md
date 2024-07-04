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

That's are some cases we think about now, we may add more in the futhure.

