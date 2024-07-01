In this section let's see how to interprete a function call. Now we have two kinds of nodes that are related to function, one is node func_decl which is the dclaration of a function, the other is node with type of call which is calling a function.
We need to combine this two together to complete the evaluation of a function call.

Actually before introducing function, our code is just like in an anonymous function without input parameters. When we interprete the code, we begin with the root of the parsing tree which is the node of type program, then we traverse down the tree
,visit each node and trigger the visitor method of each node. It is similar when evaluating function, when we visit the func_decl node, we save the node in a map as value and the function name as key, and when we visit a call node, we take the name
of function being called, and use it to get the func_decl node from the map, then fill in the value of arguments from the code node to func_decl node and using interpreter to traverse the tree begining with the func_decl node.

The whole process just like following, when we complete parsing the code, and there is a func_decl node and a call node in the parsing tree:

![evaluating function](https://github.com/wycl16514/dragonscript-function-call/assets/7506958/8b71c99d-31f4-4773-97be-562cc179eb74)

In the process of evaluating the code, when we visit the func_decl node, we get the name of the function and the node itself and save them in a call map:

