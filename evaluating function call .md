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
            if (!this.callMap[funcName]) {
                return this.callMap[funcName]
            }
        }
    }
...
}
```
We design call map as a chain just like enviroment that's because function can be declared in a local block or in function body, and when there is a function call, we need to look for the root node of the function from the inner most call map 
first just like we access a variable inside a block, a function can be called if the function is declared outside of the current block or scope. Now turn into intepreter.js and do the following code:
```js

```
