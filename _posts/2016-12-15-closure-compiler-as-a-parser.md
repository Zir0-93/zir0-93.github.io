---
title:  "Analyzing JavaScript Programmatically In Java Using The Google-Closure Compiler"
date:   2016-12-15 15:04:23
icon: /images/closurecompiler.png
tags: [javascript, java, parsing, static analysis, compiler]
description: "The Google Closure Compiler is designed to minify and optimize JavaScript, but its internals include a full AST parser that can be repurposed for programmatic code analysis. This post demonstrates how to use the Closure Compiler as a JavaScript parsing backend from Java, giving you access to a structured representation of any JavaScript source file without writing your own parser. The approach enables use cases like dependency analysis, code quality checks, and automated refactoring tooling. It is particularly useful when you need to reason about JavaScript code programmatically within a JVM-based toolchain."
excerpt_separator: <!--more-->
---
The Closure Compiler is a tool for making JavaScript download and run faster. Instead of compiling from a source language to
machine code, it compiles from JavaScript to better JavaScript. It parses your JavaScript, analyzes it, removes dead
code and rewrites and minimizes what's left. It also checks syntax, variable references, and types, and warns about
common JavaScript pitfalls.

<div style="border:1px solid rgba(15,23,42,0.08);border-radius:12px;padding:14px 18px;margin:16px 0;background:rgba(255,255,255,0.6);">
<p style="margin:0 0 8px 0;font-size:0.75rem;font-weight:700;letter-spacing:0.08em;text-transform:uppercase;color:#6b7280;">Relevant Repos</p>
<div style="display:flex;flex-wrap:wrap;gap:6px;align-items:center;">
<a href="https://github.com/hadi-technology/clarpse"><img src="https://img.shields.io/badge/Static%20Analysis-Clarpse-6a0dad?logo=github" alt="clarpse"></a>
</div>
</div>
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


```javascript
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
