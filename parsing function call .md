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
unary_recursive-> ("!"|"-") unary | call
call -> primary LEFT_PAREN arguments RIGHT_PAREN
arguemnts -> expression argument_recursive
argumen_recursive -> COMMA arguments | EPSILON
```

Let's try to do some test for the grammar above, if we have a statment like getCallback()(); How the grammar aboved to parse it?  First we will go from program down the load to unary_recursive then to call,
when applying the rule of call, we first using primary rule to handle the string "getCallback", which it will match up with IDENTIFIER for the rule of primary -> IDENTIFIER, then we go back to the symbol next
to primary which is matching the left paren, then we goes into the rule of arguments and through arguments we go into express, and from expression we go into assignment. Since the current token is RIGHT_PAREN,
we won't do anything in assigment and simply return the symbol next 

