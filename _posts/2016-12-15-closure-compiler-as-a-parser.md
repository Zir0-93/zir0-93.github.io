---
title:  "Analyzing JavaScript Programmatically In Java Using The Google-Closure Compiler"
date:   2016-12-15 15:04:23
categories: [oop, programming, javascript, closure, compiler, google]
tags: [oop, programming, javascript, closure, compiler, google]
excerpt_separator: <!--more-->
---
**From the Closure Compiler website:**

*"The Closure Compiler is a tool for making JavaScript download and run faster. Instead of compiling from a source language to 
machine code, it compiles from JavaScript to better JavaScript. It parses your JavaScript, analyzes it, removes dead 
code and rewrites and minimizes what's left. It also checks syntax, variable references, and types, and warns about 
common JavaScript pitfalls."*
 <!--more-->
 
### Closure Compiler As A JavaScript Parser

Wanting to support ES6 programs for [Clarpse](https://github.com/Zir0-93/clarpse/blob/master/README.md), I searched for a Java based ES6 compatibile JavaScript compiler.
At the time of writing, the Google Closure-Compiler was the only tool that fit this category. While the tool is not natively meant to be used
as parser, its Java API can be consumed to perform static analysis on JavaScript code with a few modifications.


We will create a simple JavaScript source code analyzer that will output the names of all the classes and methods in a given JavaScript ES6 program.
First, we extend the [AbstractShallowCallback](https://github.com/google/closure-compiler/blob/master/src/com/google/javascript/jscomp/NodeTraversal.java#L159) class which will visit all nodes in the [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree)
representing the given JavaScript source; however, it will not traverse into any function bodies.
There are other traversal algorithms that can be used as well, for a full list check out the [NodeTraversal Class](https://github.com/google/closure-compiler/blob/master/src/com/google/javascript/jscomp/NodeTraversal.java).
Next, we need to provide an implementation for the ```visit()``` method which is activated everytime a Node is encountered. It is therefore in this method where we will output the value of the current node if the node corresponds
to a class or method as demonstrated below.

```java
public class JavaScriptAnalyzer extends AbstractShallowCallback {

    @Override
    public void visit(NodeTraversal t, Node n, Node parent) {
        if (n.isClass()) {
            System.out.println(n.getFirstChild().getString());
        }
        if (n.isMemberFunctionDef() || n.isGetterDef() || n.isSetterDef()) {
            System.out.println(n.getString());
        }
        if (n.isFunction()) {
            System.out.println(n.getFirstChild().getString());
        }
        // there is more work required to detect all types of methods that
        // has been left out  to improve brevity...
    }
}
```

Now, we initialize a compiler and run our created JavaScript analyzer on a given JavaScript source file.


```java
public void parse(String jsFileContent, String jsName) throws Exception {
        Compiler compiler = new Compiler();
        CompilerOptions options = new CompilerOptions();
        options.setIdeMode(true);
        compiler.initOptions(options);
        Node root = new JsAst(SourceFile.fromCode(jsName, jsFileContent)).getAstRoot(compiler);
        JavaScriptAnalyzer jsListener = new JavaScriptAnalyzer();
        NodeTraversal.traverseEs6(compiler, root, jsListener);
    }
```


And thats it! Running the above code on the following source file:


```
class Polygon {
  constructor(height, width) {}
  logWidth() {}
  set width(value) {}
  get height(value) {}
}
```


Produces the following output as expected:


```
constructor
logWidth
width
height
Polygon
```
