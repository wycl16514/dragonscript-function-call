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
call -> primary  argument_list 
argument_list -> LEFT_PAREN arguments RIGHT_PAREN argument_list_recursive
argument_list_recursive -> argument_list | EPSILON
arguments -> expression arguments_recursive
arguments_recursive -> COMMA arguments | EPSILON
```

Let's try to do some test for the grammar above, if we have a statment like getCallback(1,2,3)(4,5,6); How the grammar aboved to parse it? The string "getCallback" will be parsed in primary and match by 
IDENTIFIER, and we go into the rule of argument_list. Since the current token is LEFT_PAREN and we match the first symbol at the right of argument_list and enter the rule of arguments. Now the input symbols
are 1,2,3 , these symbols can be matched by the arguments rule, when we first visit "1", then it will be matched by rule of expression in the right of arguments then we go into arguments_recursive, and we 
can match the COMMA at the first symbol of right of rule arguments, then repeat for symbols "2", "," , "3", when we visit RIGHT_PARENT the rule arguments_recursive -> EPSILON will apply and we goback to
RIGHT_PARENT of argument_list to do the match, and we go into argument_list_recursive to match the "(4,5,6)" again.
