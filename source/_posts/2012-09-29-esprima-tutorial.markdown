---
layout: post
title: "Esprima tutorial"
date: 2012-09-29 12:27
comments: true
categories:
- node
- javascript
---

Recently I had a task that sounded something like this:

>Get a JS code and generate some other JS code from it.

The first part was most difficult. Info we need from code was too complex to get using regular expressions.
So the search for JS parser began. It ended pretty soon when I found [Esprima](http://esprima.org/) parser.
Now I want to share my experience with it in a form of tutorial. We won't be building code generator - I think 
simpler task will be enough to dive in.

<!-- more -->

## What we'll build

We will build a very simple static analyzer which will run from command line.
It will warn about:

1. Declared, but not called functions;
2. Calls to undeclared functions;
3. Functions declared multiple times.

Of course, it's not intended for compelling with excellent [JSHint](http://www.jshint.com/) or any other static 
analyzer. The only purpose it serves is to show you the basics of JS parsing using Esprima. The things that our 
analyzer will NOT do:

1. Recognize any form of function declaration except:
``` javascript
function name(...) {
    ...
}
```
2. Recognize any form of function call except:
``` javascript
name(...)
```
3. Know something about imports, exports or predefined globals;
4. Work with multiple files.

This is not the flaws of a parser. All this features can be easily implemented, they are just out of scope of
tutorial.

Example will be built using NodeJS, but Esprima works also in a browser.

## Preparations

I will be using node v0.8.10 and Esprima v0.9.9. 
Each of following commands can also be done with GUI or Windows Shell, but I'll give examples only for Unix-like 
OS terminal.
Create a directory for your project and install the library:

``` bash
mkdir esprima-tutorial
cd esprima-tutorial
npm install esprima@0.9.9
```

Create a script named analyze.js with the following content:

``` javascript
var fs = require('fs'),
    esprima = require('esprima');

function analyzeCode(code) {
    // 1
}

// 2
if (process.argv.length < 3) {
    console.log('Usage: analyze.js file.js');
    process.exit(1);
}

// 3
var filename = process.argv[2];
console.log('Reading ' + filename);
var code = fs.readFileSync(filename);

analyzeCode(code);
console.log('Done');
```

What happens here:

1. We declare a function where we will be doing all code parsing and analysis stuff. For now its empty;
2. We are checking that user passes command-line argument with a file name to analyze. Why we checking for the third argument? The answer is in [node documentation](http://nodejs.org/api/process.html#process_process_argv):
> The first element will be 'node', the second element will be the name of the JavaScript file.
> The next elements will be any additional command line arguments.
3. We are reading the file content and passing it to `analyzeCode`. For simplicity, I use sync version of 
`fs.readFile`.

## Parsing code and walking through AST

Parsing can be done with a single line of code:

``` javascript
function analyzeCode(code) {
    var ast = esprima.parse(code);
}
```

`esprima.parse` accepts string or node's `Buffer` object. You can also pass additional options as the second
parameter to include comments, line and column numbers, tokens, etc but this is out of the scope of this tutorial.

The result of the `esprima.parse` code will be an abstract syntax tree (AST) of your program in 
[Spider Monkey Parser API format](https://developer.mozilla.org/en-US/docs/SpiderMonkey/Parser_API).

AST is a representation of a program code in a tree structure. For example, if we have the expression:

``` javascript
2 * 3
```

AST can be graphically represented as:

![AST Example](/assets/images/ast_example.png)

Same expression in Parser API format will look like:

``` javascript
{
    "type": "Program",
    "body": [
        {
            "type": "ExpressionStatement",
            "expression": {
                "type": "BinaryExpression",
                "operator": "*",
                "left": {
                    "type": "Literal",
                    "value": 2
                },
                "right": {
                    "type": "Literal",
                    "value": 3
                }
            }
        }
    ]
}
```
The main entity of `esprima.parse` output is node. Each node has a type (root node is always has `Program` type),
0 or more properties and 0 or more subnodes. `type` is the only common property for nodes - names of the
other properties and subnodes depends on it.

In above example, Program has the only one subnode - `body` of `ExpressionStatement` type which too contains only 
`expression` subnode of type `BinaryExpression`. `BinaryExpression` has property `operator` with value of "*" and 
`left` and `right` `Literal` subnodes.

To be able to analyze the code we need a way to loop throught AST. Add following code before `analyzeCode` function:

``` javascript
function traverse(node, func) {
    func(node);//1
    for (var key in node) { //2
        if (node.hasOwnProperty(key)) { //3
            var child = node[key];
            if (typeof child === 'object' && child !== null) { //4

                if (Array.isArray(child)) {
                    child.forEach(function(node) { //5
                        traverse(node, func);
                    });
                } else {
                    traverse(child, func); //6
                }
            }
        }
    }
}
```

This function accepts root node and a function and calls it recursively for each node in a tree. 

1. Calling the function for a root node;
2. Loop through all properties of a root a node;
3. Ignore inherited properties of an object;
4. Ignore simple and null properties. All nodes are actually complex objects and all properties has simple type. 
Null check is necessary, because `typeof null === 'object'`;
5. Each child can be either single subnode or array of subnodes. If its an array, we call `traverse` recursively 
for each subnode in it.
6. If child is not an array, just call `traverse` recurively on it.

To test the function replace `analyzeCode` function with the following code:

``` javascript
function analyzeCode(code) {
    var ast = esprima.parse(code);
    traverse(ast, function(node) {
        console.log(node.type);
    });
}
```
When you execute you script on some JS code you should see types of all nodes in a tree.

## Getting data for analysis

To do the analysis we need to loop through an AST and count number of calls and declarations for each function.
So, we need to know format of two node types. The first one is function declaration and it looks like this:

```
{
    "type": "FunctionDeclaration",
    "id": {
        "type": "Identifier",
        "name": "myAwesomeFunction"
    },
    "params": [
        ...
    ],
    "body": {
        "type": "BlockStatement",
        "body": [
            ...
        ]
    }
}
```

Type of a node is `FunctionDeclaration`. Identifier is stored in `id` subnode and name of the function is in a
`name` property of this node. `params` and `body` contain parameters and body of the function. Rest of the node
properties is omitted.

Second node is CallExpression: 

``` javascript
"expression": {
    "type": "CallExpression",
    "callee": {
        "type": "Identifier",
        "name": "myAwesomeFunction"
    },
    "arguments": []
}
```

`callee` is an object being called. We are interested only in callee with `Identifier` type. Few other possible
types are `MemberExpression` (call of the object method) and `FunctionExpression` (self invoking function).

Now we have all information to perform the analysis. Replace `analyzeCode` function with:

``` javascript
function analyzeCode(code) {
    var ast = esprima.parse(code);
    var functionsStats = {}; //1
    var addStatsEntry = function(funcName) { //2
        if (!functionsStats[funcName]) {
            functionsStats[funcName] = {calls: 0, declarations:0};
        }
    };

    traverse(ast, function(node) { //3
        if (node.type === 'FunctionDeclaration') {
            addStatsEntry(node.id.name); //4
            functionsStats[node.id.name].declarations++;
        } else if (node.type === 'CallExpression' && node.callee.type === 'Identifier') {
            addStatsEntry(node.callee.name);
            functionsStats[node.callee.name].calls++; //5
        }
    });
}

```

1. Creating a new empty object that will store calls and declarations statistics for each function. Function name
will be the key of statistics entry;
2. Declaring a function that will add an empty entry for the function name to the `functionStats` object, 
if we haven't done this previously;
3. Begin loop through the AST;
4. If we found function declaration, increase declaration count;
5. If we found function call by name, increase call.

## Processing results

Now the final part, processing the results we gathered and reporting all found issues.

Add following function:

``` javascript
function processResults(results) {
    for (var name in results) {
        if (results.hasOwnProperty(name)) {
            var stats = results[name];
            if (stats.declarations === 0) {
                console.log('Function', name, 'undeclared');
            } else if (stats.declarations > 1) {
                console.log('Function', name, 'decalred multiple times');
            } else if (stats.calls === 0) {
                console.log('Function', name, 'declared but not called');
            }
        }
    }
}
```

I think the code is pretty self-explanatory and doesn't need my comments.
Finally, add call to `processResults` to the bottom of `analyzeCode` function:

``` javascript
    processResults(functionsStats);
```

## Testing

Run the script on this meaningless code:

``` javascript
function declaredTwice() {
}

function main() {
    undeclared();
}

function unused() {
}

function declaredTwice() {
}

main();

```

You should see something like this:

``` bash
Inf:esprima-tutorial$ node analyze.js test.js
Reading test.js
Function declaredTwice decalred multiple times
Function undeclared undeclared
Function unused declared but not called
Done
```

Of course, if you run it on some real code you will see all the flaws we discussed at the beginning of the 
article. Again, point of the article is to teach how to use Esprima, not how to write static analyzers.

## Conclusion

Its time to end the tutorial. We've learnt how to parse JS code using Esprima, the format of SpiderMonkey Parse API
syntax tree. Also, we've learn how to traverse through AST and how to search for syntax constructions we
interested in.

## Useful links

* [Source code for the tutorial]();
* [Esprima library](http://esprima.org/);
* [SpiderMonkey Parser API](https://developer.mozilla.org/en-US/docs/SpiderMonkey/Parser_API);
* [Online parser demo](http://esprima.org/demo/parse.html).
