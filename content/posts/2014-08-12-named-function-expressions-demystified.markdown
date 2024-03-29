---
showDate: true
date: "2014-08-12"
layout: post
tags: ['Javascript', 'tech', 'note', 'collection']
title: "\u547D\u540D\u51FD\u6570\u8868\u8FBE\u5F0F\u63A2\u79D8"
---

>作者：Juriy "kangax" Zaytsev  译者：为之漫笔 译文原文：<a target="_blank" href="//www.cn-cuckoo.com/main/wp-content/uploads/2009/12/named-function-expressions-demystified.html">命名函数表达式探秘</a> 。  
关于Javascript的经典译文，由李松峰老师翻译。曾帮助我更好地理解Javascript基础。值得收藏。    
 
### 目录：

<!--more-->

<ol>
  <li><a href="#introduction">前言</a></li>
  <li><a href="#expr-vs-decl">函数表达式与函数声明</a></li>
  <li><a href="#function-statements">函数语句</a></li>
  <li><a href="#named-expr">命名函数表达式</a></li>
  <li><a href="#names-in-debuggers">调试器中的函数名</a></li>
  <li><a href="#jscript-bugs">JScript的bug</a></li>
  <li><a href="#jscript-memory-management">JScript的内存管理</a></li>
  <li><a href="#tests">测试</a></li>
  <li><a href="#safari-bug">Safari中存在的bug</a></li>
  <li><a href="#spidermonkey-peculiarity">SpiderMonkey的怪癖</a></li>
  <li><a href="#solution">解决方案</a></li>
  <li><a href="#alt-solution">替代方案</a></li>
  <li><a href="#webkit-displayName">WebKit的displayName</a></li>
  <li><a href="#future-considerations">对未来的思考</a></li>
  <li><a href="#credits">致谢</a></li>
</ol>

<h2 id="introduction">前言</h2>

我觉得很奇怪，网上好像一直没有人认真地讨论过命名函数表达式（Named Function Expression，即“有名字函数表达式”，与“匿名函数”相对。——译者注）。而这也许正是各种各样的误解随处可见的一个原因。在这篇文章里，我打算从理论和实践两个方面出发，对这些令人惊叹的JavaScript结构的优缺点给出一个结论。

简单来讲，命名函数表达式只有一个用处——在调试器或性能分析程序中描述函数的名称。没错，也可以使用函数名实现递归，但你很快就会知道，目前来看这通常是不切实际的。当然，如果你不关注调试，那就没什么可担心的。否则，就应该往下看一看，看看在跨浏览器开发中都会出现哪些小毛病（glitch），也看看应该怎样解决它们。

一开始呢，我会先介绍一下什么是函数表达式，以及现代调试器如何处理它们之类的内容。要是你比较心急，请直接跳到<a href="#solution">“最终方案”</a>部分，该部分详细说明了怎样才能安全地使用这些结构。

<h2 id="expr-vs-decl">函数表达式与函数声明</h2>

在ECMAScript中，有两个最常用的创建函数对象的方法，即使用函数表达式或者使用函数声明。这两种方法之间的区别可谓 相当地令人困惑；至少我是相当地困惑。对此，ECMA规范只明确了一点，即函数声明 必须始终带有一个标识符（Identifier）——也就是函数名呗，而函数表达式 则可省略这个标识符：

>函数声明：  
function Identifier ( FormalParameterList opt ){ FunctionBody }

>函数表达式：  
function Identifier opt ( FormalParameterList opt ){ FunctionBody }


显然，在省略标识符的情况下， “表达式” 也就只能是表达式了。可要是不省略标识符呢？谁知道它是一个函数声明，还是一个函数表达式——毕竟，这种情况下二者是完全一样的啊？实践表明，ECMAScript是通过上下文来区分这两者的：假如 function foo(){} 是一个赋值表达式的一部分，则认为它是一个函数表达式。而如果 function foo(){} 被包含在一个函数体内，或者位于程序（的最上层）中，则将它作为一个函数声明来解析。

```javascript
function foo(){}; // 声明，因为它是程序的一部分
var bar = function foo(){}; // 表达式，因为它是赋值表达式（AssignmentExpression）的一部分

new function bar(){}; // 表达式，因为它是New表达式（NewExpression）的一部分

(function(){
    function bar(){}; // 声明，因为它是函数体（FunctionBody）的一部分
})();
```

还有一种不那么显而易见的函数表达式，就是被包含在一对圆括号中的函数—— (function foo(){})。将这种形式看成表达式同样是因为上下文的关系：(和)构成一个分组操作符，而分组操作符只能包含表达式：

下面再多看几个例子吧：

```javascript
function foo(){}; // 函数声明
(function foo(){}); // 函数表达式：注意它被包含在分组操作符中
  
try {
    (var x = 5); // 分组操作符只能包含表达式，不能包含语句（这里的var就是语句）
} catch(err) {
    // SyntaxError（因为“var x = 5”是一个语句，而不是表达式——对表达式求值必须返回值，但对语句求值则未必返回值。——译者注）
}
```

不知道大家有没有印象，在使用 eval 对JSON求值的时候，JSON字符串通常是被包含在一对圆括号中的—— eval('(' + json + ')')。这样做的原因当然也不例外——分组操作符，也就是那对圆括号，会导致解析器强制将JSON的花括号当成表达式而不代码块来解析：

```javascript
try {
  { "x": 5 }; // {和}会被作为块来解析
} catch(err) {
  // SyntaxError（“'x':5”只是构建对象字面量的语法，但该语法不能出现在外部的语句块中。——译者注）
}

({ "x": 5 }); // 分组操作符会导致解析器强制将{和}作为对象字面量来解析
```

声明和表达式的行为存在着十分微妙而又十分重要的差别。

首先，函数声明会在任何表达式被解析和求值之前先行被解析和求值。即使声明位于源代码中的最后一行，它也会先于同一作用域中位于最前面的表达式被求值。还是看个例子更容易理解。在下面这个例子中，函数 fn 是在 alert 后面声明的。但是，在 alert 执行的时候，fn已经有定义了：

```javascript
alert(fn());

function fn() {
  return 'Hello world!';
}
```

函数声明还有另外一个重要的特点，即通过条件语句控制函数声明的行为并未标准化，因此不同环境下可能会得到不同的结果。有鉴于此，奉劝大家千万不要在条件语句中使用函数声明，而要使用函数表达式。

```javascript
// 千万不要这样做！
// 有的浏览器会把foo声明为返回first的那个函数
// 而有的浏览器则会让foo返回second

if (true) {
  function foo() {
    return 'first';
  }
}
else {
  function foo() {
    return 'second';
  }
}
foo();

// 记住，这种情况下要使用函数表达式：
var foo;
if (true) {
  foo = function() {
    return 'first';
  };
}
else {
  foo = function() {
    return 'second';
  };
}
foo();
```

想知道使用函数声明的实际规则到底是什么？继续往下看吧。嗯，有人不想知道？那请跳过下面这段摘录的文字。

`FunctionDeclaration`（函数声明）只能出现在`Program`（程序）或`FunctionBody`（函数体）内。从句法上讲，它们 不能出现在`Block`（块）（{ ... }）中，例如不能出现在 if、while 或 for 语句中。因为 `Block`（块） 中只能包含`Statement`（语句）， 而不能包含`FunctionDeclaration`（函数声明）这样的`SourceElement`（源元素）。另一方面，仔细看一看产生规则也会发现，唯一可能让`Expression`（表达式）出现在`Block`（块）中情形，就是让它作为`ExpressionStatement`（表达式语句）的一部分。但是，规范明确规定了`ExpressionStatement`（表达式语句）不能以关键字`function`开头。而这实际上就是说，`FunctionExpression`（函数表达式）同样也不能出现在`Statement`（语句）或`Block`（块）中（别忘了`Block`（块）就是由`Statement`（语句）构成的）。

由于存在上述限制，只要函数出现在块中（像上面例子中那样），实际上就应该将其看作一个语法错误，而不是什么函数声明或表达式。但问题是，我还没见过哪个实现是按照上述规则来解析这些函数的；好像每个实现都有自己的一套。

有必要提醒大家一点，根据规范的描述，实现可以引入语法扩展（见第16部分），只不过任何情况下都不能违反规定。而目前的诸多客户端也正是照此办理的。其中有一些会把块中的函数声明当作一般的函数声明来解析——把它们提升到封闭作用域的顶部；另一些则引入了不同的语义并采用了稍复杂一些的规则。

<h2 id="function-statements">函数语句</h2>

在诸如此类的对ECMAScript的语法扩展中，有一项就是函数语句，基于Gecko的浏览器（在Mac OS X平台的Firefox 1-3.7a1pre中测试过）目前都实现了该项扩展。可是不知道为什么，很多人好像都不知道这项扩展，也就更谈不上对其优劣的评价了（<a target="_blank" href="https://developer.mozilla.org/En/Core_JavaScript_1.5_Reference:Functions#Conditionally_defining_a_function">MDC（Mozilla Developer Center，Mozilla开发者中心）提到过这个问题</a>，但是只有那么三言两语）。请大家记住，我们是抱着学习和满足自己好奇心的态度来讨论函数语句的。因此，除非你只针对基于Gecko的环境编写脚本，否则我不建议你使用这个扩展。

闲话少说，下面我们就来看看这些非标准的结构有哪些特点：

1.一般语句可以出现的地方，函数语句也可以出现。当然包括块中：

```javascript
if (true) {
  function f(){ }
}
else {
  function f(){ }
}
```

2.函数语句可以像其他语句一样被解析，包含基于条件执行的情形：

```javascript
if (true) {
  function foo(){ return 1; }
}
else {
  function foo(){ return 2; }
}
foo(); // 1
// 注意其他类型的客户端会把这里的foo解析为函数声明
// 因此，第二个foo会覆盖第一个，结果返回2而不返回1
```

3.函数语句不是在变量初始化期间声明的，而是在运行时声明的——与函数表达式一样。不过，一旦声明，函数语句的标识符就在函数的整个作用域有效了。标识符有效性正是导致函数语句与函数表达式不同的关键所在（下一节将会展示命名函数表达式的具体行为）。

```javascript
// 此时，foo还没有声明
typeof foo; // "undefined"
if (true) {
  // 一进入这个块，foo就被声明并在整个作用域中有效了
  function foo(){ return 1; }
}
else {
  // 永远不会进入这个块，因此这里的foo永远不会被声明
  function foo(){ return 2; }
}
typeof foo; // "function"

```

通常，可以通过下面这样符合标准（但更繁琐一点）的代码来模拟前例中函数语句的行为:

```javascript
var foo;
if (true) {
  foo = function foo(){ return 1; };
}
else {
  foo = function foo() { return 2; };
}
```

4.函数语句与函数声明或命名函数表达式的字符串表示类似（而且包含标识符——即此例中的foo）：

```javascript
if (true) {
  function foo(){ return 1; }
}
String(foo); // function foo() { return 1; }
```

5.最后，早期基于Gecko的实现（Firefox 3及以前版本）中存在一个bug，即函数语句覆盖函数声明的方式不正确。在这些早期的实现中，函数语句不知何故不能覆盖函数声明：

```javascript
// 函数声明
function foo(){ return 1; }
if (true) {
  // 使用函数语句来重写
  function foo(){ return 2; }
}
foo(); // FF及以前版本返回1，FF3.5及以后版本返回2

// 但是，如果前面是函数表达式，则没有这个问题
var foo = function(){ return 1; };
if (true) {
  function foo(){ return 2; }
}
foo(); // 在所有版本中都返回2
```

大家请注意，Safari的某些早期版本（至少包括1.2.3、2.0 - 2.0.4以及3.0.4，可能也包括更早的版本）实现了与**SpiderMonkey**完全一样的函数语句。本节所有的例子（不包括最后一个bug示例），在Safari的那些版本中都会得到与Firefox完全相同的结果。此外，Blackberry（至少包括8230、9000和9530）浏览器好像也具有类似的行为。上述这种行为的差异化再次说明——千万不能盲目地依赖这些扩展啊（如下所述，可以根据特性测试来使用函数表达式。——译者注）！

<h2 id="named-expr">命名函数表达式</h2>

函数表达式实际上还是很常见的。Web开发中有一个常用的模式，即基于对某种特性的测试来“伪装”函数定义，从而实现性能最优化。由于这种伪装通常都出现在相同的作用域中，因此基本上一定要使用函数表达式。毕竟，如前所述，不应该根据条件来执行函数声明：

```javascript
// 这里的contains取自APE Javascript库的源代码，网址为//dhtmlkitchen.com/ape/，作者盖瑞特·斯密特（Garrett Smit）
var contains = (function() {
  var docEl = document.documentElement;

  if (typeof docEl.compareDocumentPosition != 'undefined') {
    return function(el, b) {
      return (el.compareDocumentPosition(b) & 16) !== 0;
    }
  }
  else if (typeof docEl.contains != 'undefined') {
    return function(el, b) {
      return el !== b && el.contains(b);
    }
  }
  return function(el, b) {
    if (el === b) return false;
    while (el != b && (b = b.parentNode) != null);
    return el === b;
  }
})();
```

提到**命名函数表达式**，很显然，指的就是有名字（技术上称为标识符）的函数表达式。在最前面的例子中，var bar = function foo(){};实际上就是一个以foo作为函数名字的函数表达式。对此，有一个细节特别重要，请大家一定要记住，即这个名字只在新定义的函数的作用域中有效——规范要求标识符不能在外围的作用域中有效：

```javascript
var f = function foo(){
  return typeof foo; // foo只在内部作用域中有效
};
// foo在“外部”永远是不可见的
typeof foo; // "undefined"
f(); // "function"
```

那么，这些所谓的命名函数表达式到底有什么用呢？为什么还要给它们起个名字呢？

原因就是有名字的函数可以让调试过程更加方便。在调试应用程序时，如果调用栈中的项都有各自描述性的名字，那么调试过程带给人的就是另一种完全不同的感受。

<h2 id="names-in-debuggers">调试器中的函数名</h2>

在函数有相应标识符的情况下，调试器会将该标识符作为函数的名字显示在调用栈中。有的调试器（例如Firebug）甚至会为匿名函数起个名字并显示出来，让它们与那些引用函数的变量具有相同的角色。可遗憾的是，这些调试器通常只使用简单的解析规则，而依据简单的解析规则提取出来的“名字”有时候没有多大价值，甚至会得到错误结果。（Such extraction is usually quite fragile and often produces false results. ）

下面我们来看一个简单的例子：

```javascript
function foo(){
  return bar();
}
function bar(){
  return baz();
}
function baz(){
  debugger;
}
foo();

// 这里使用函数声明定义了3个函数
// 当调试器停止在debugger语句时，
// Firgbug的调用栈看起来非常清晰：
baz
bar
foo
expr_test.html()
```

这样，我们就知道foo调用了bar，而后者接着又调用了baz（而foo本身又在expr_test.html文档的全局作用域中被调用）。但真正值得称道的，则是Firebug会在我们使用匿名表达式的情况下，替我们解析函数的“名字”：

```javascript
function foo(){
  return bar();
}
var bar = function(){
  return baz();
}
function baz(){
  debugger;
}
foo();

// 调用栈：
baz
bar()
foo
expr_test.html()
```

相反，不那么令人满意的情况是，当函数表达式复杂一些时（现实中差不多总是如此），调试器再如何尽力也不会起多大的作用。结果，我们只能在调用栈中显示函数名字的位置上赫然看到一个问号：

```javascript
function foo(){
  return bar();
}
var bar = (function(){
  if (window.addEventListener) {
    return function(){
      return baz();
    }
  }
  else if (window.attachEvent) {
    return function() {
      return baz();
    }
  }
})();
function baz(){
  debugger;
}
foo();

// 调用栈：
baz
(?)()
foo
expr_test.html()
```

此外，当把一个函数赋值给多个变量时，还会出现一个令人困惑的问题：

```javascript
function foo(){
  return baz();
}
var bar = function(){
  debugger;
};
var baz = bar;
bar = function() { 
  alert('spoofed');
}
foo();

// 调用栈：
bar()
foo
expr_test.html()
```

可见，调用栈中显示的是foo调用了bar。但实际情况显然并非如此。之所以会造成这种困惑，完全是因为baz与另一个函数——包含代码alert('spoofed');的函数——“交换了”引用所致。实事求是地说，这种解析方式在简单的情况下固然好，但对于不那么简单的大多数情况而言就没有什么用处了。

归根结底，只有**命名函数表达式才是产生可靠的栈调用信息的唯一途径**。下面我们有意使用命名函数表达式来重写前面的例子。请大家注意，从自执行包装块中返回的两个函数都被命名为了bar：

```javascript
function foo(){
  return bar();
}
var bar = (function(){
  if (window.addEventListener) {
    return function bar(){
      return baz();
    }
  }
  else if (window.attachEvent) {
    return function bar() {
      return baz();
    }
  }
})();
function baz(){
  debugger;
}
foo();

// 这样，我们就又可以看到清晰的调用栈信息了！
baz
bar
foo
expr_test.html()
```

在我们为发现这根救命稻草而欢呼雀跃之前，请大家稍安勿躁，再听我聊一聊大家所衷爱的JScript。

<h2 id="jscript-bugs">JScript的bug</h2>

令人讨厌的是，JScript（也就是IE的ECMAScript实现）严重混淆了命名函数表达式。JScript搞得现如今很多人都站出来反对命名函数表达式。而且，**直到JScript的最近一版——IE8中使用的5.8版——仍然存在下列的所有怪异问题**。

下面我们就来看看IE在它的这个“破”实现中到底都搞出了哪些花样。唉，只有知已知彼，才能百战不殆嘛。请注意，为了清晰起见，我会通过一个个相对独立的小例子来说明这些问题，虽然这些问题很可能是一个主bug引起的一连串的后果。

### 例1：函数表达式的标识符渗透到外部（enclosing）作用域中

```javascript
var f = function g(){};
typeof g; // "function"
```

还有人记得吗，我们说过：命名函数表达式的标识符在其外部作用域中是无效的? 好啦，JScript明目张胆地违反了这一规定——上面例子中的标识符g被解析为函数对象。这是最让人头疼的一个问题了。这样，任何标识符都可能会在不经意间“污染”某个外部作用域——甚至是全局作用域。而且，这种污染常常就是那些难以捕获的bug的来源。

<h3 id="example_2_named_function_expression_is_treated_as_both_function_declaration_and_function_expression">例2：将命名函数表达式同时当作函数声明和函数表达式</h3>

```javascript
typeof g; // "function"
var f = function g(){};
```

如前所述，在特定的执行环境中，函数声明会先于任何表达式被解析。上面这个例子展示了JScript实际上是把命名函数表达式当作函数声明了；因为它在“实际的”声明之前就解析了g。

这个例子进而引出了下一个例子：

### 例3：命名函数表达式会创建两个截然不同的函数对象！

```javascript
var f = function g(){};
f === g; // false

f.expando = 'foo';
g.expando; // undefined
```

问题至此就比较严重了。或者可以说修改其中一个对象对另一个丝毫没有影响——这简直就是胡闹！通过例子可以看出，出现两个不同的对象会存在什么风险。假如你想利用缓存机制，在f的属性中保存某个信息，然后又想当然地认为可以通过引用相同对象的g的同名属性取得该信息，那么你的麻烦可就大了。

再来看一个稍微复杂点的情况。

### 例4：只管顺序地解析函数声明而忽略条件语句块

```javascript
var f = function g() {
    return 1;
};
if (false) {
    f = function g(){
        return 2;
    };
}
g(); // 2
```

要查找这个例子中的bug就要困难一些了。但导致bug的原因却非常简单。首先，g被当作函数声明解析，而由于JScript中的函数声明不受条件代码块约束（与条件代码块无关），所以在“该死的”if分支中，g被当作另一个函数——function g(){ return 2 }——又被声明了一次。然后，所有“常规的”表达式被求值，而此时f被赋予了另一个新创建的对象的引用。由于在对表达式求值的时候，永远不会进入“该死的”if分支，因此f就会继续引用第一个函数——function g(){ return 1 }。分析到这里，问题就很清楚了：假如你不够细心，在f中调用了g（在执行递归操作的时候会这样做。——译者注），那么实际上将会调用一个毫不相干的g函数对象（即返回2的那个函数对象。——译者注）。

聪明的读者可能会联想到：在将不同的函数对象与arguments.callee进行比较时，这个问题会有所表现吗？callee到底是引用f还是引用g呢？下面我们就来看一看：

```javascript
var f = function g(){
  return [
    arguments.callee == f,
    arguments.callee == g
  ];
};
f(); // [true, false]
g(); // [false, true]
```

看到了吧，arguments.callee引用的始终是被调用的函数。实际上，这应该是件好事儿，原因你一会儿就知道了。

另一个“意外行为”的好玩的例子，当我们在不包含声明的赋值语句中使用命名函数表达式时可以看到。不过，此时函数的名字必须与引用它的标识符相同才行：

```javascript
(function(){
  f = function f(){};
})();
```

众所周知（但愿如此。——译者注），不包含声明的赋值语句（注意，我们不建议使用，这里只是出于示范需要才用的）在这里会创建一个全局属性f。而这也是标准实现的行为。可是，JScript的bug在这里又会出点乱子。由于JScript把命名函数表达式当作函数声明来解析（参见<a href="#example_2_named_function_expression_is_treated_as_both_function_declaration_and_function_expression">前面的“例2”</a>），因此在变量声明阶段，f会被声明为局部变量。然后，在函数执行时，赋值语句已经不是未声明的了（因为f已经被声明为局部变量了。——译者注），右手边的function f(){}就会被直接赋给刚刚创建的局部变量f。而全局作用域中的f根本不会存在。

看完这个例子后，相信大家就会明白，如果你对JScript的“怪异”行为缺乏了解，你的代码中出现“严重不符合预期”的行为就不难理解了。

明白了JScript的缺陷以后，要采取哪些预防措施就非常清楚了。首先，要注意防范标识符泄漏（渗透）（不让标识符污染外部作用域）。其次，应该永远不引用被用作函数名称的标识符；还记得前面例子中那个讨人厌的标识符g吗？——如果我们能够当g不存在，可以避免多少不必要的麻烦哪。因此，关键就在于始终要通过f或者arguments.callee来引用函数。如果你使用了命名函数表达式，那么应该只在调试的时候利用那个名字。最后，还要记住一点，一定要把**NFE（Named Funciont Expresssions，命名函数表达式）**声明期间错误创建的函数清理干净。

嗯，对于上面最后一点，我觉得还要再啰嗦两句：

<h2 id="jscript-memory-management">JScript的内存管理</h2>

熟悉上述JScript缺陷之后，再使用这些有毛病的结构，就会发现内存占用方面的潜在问题。下面看一个简单的例子:

```javascript
var f = (function(){
  if (true) {
    return function g(){};
  }
  return function g(){};
})();
```

我们知道，这里匿名（函数）调用返回的函数——带有标识符g的函数——被赋值给了外部的f。我们也知道，命名函数表达式会导致产生多余的函数对象，而该对象与返回的函数对象不是一回事。由于有一个多余的g函数被“截留”在了返回函数的闭包中，因此内存问题就出现了。这是因为（if语句）内部（的）函数与讨厌的g是在同一个作用域中被声明的。在这种情况下 ，除非我们**显式地断开对（匿名调用返回的）g函数的引用**，否则那个讨厌的家伙会一直占着内存不放。

```javascript
var f = (function(){
  var f, g;
  if (true) {
    f = function g(){};
  }
  else {
    f = function g(){};
  }
  // 废掉g，这样它就不会再引用多余的函数了
  g = null;
  return f;
})();
```

请注意，这里也明确声明了变量g，因此赋值语句g = null就不会在符合标准的客户端（如非JScript实现）中创建全局变量g了。通过废掉对g的引用，垃圾收集器就可以把g引用的那个隐式创建的函数对象清除了。

在解决JScript NFE内存泄漏问题的过程中，我运行了一系列简单的测试，以便确定废掉g能够释放内存。

<h2 id="tests">测试</h2>

这里的测试很简单。就是通过命名函数表达式创建10000个函数，把它们保存在一个数组中。过一会儿，看看这些函数到底占用了多少内存。然后，再废掉这些引用并重复这一过程。下面是我使用的一个测试用例：

```javascript
function createFn(){
  return (function(){
    var f;
    if (true) {
      f = function F(){
        return 'standard';
      }
    }
    else if (false) {
      f = function F(){
        return 'alternative';
      }
    }
    else {
      f = function F(){
        return 'fallback';
      }
    }
    // var F = null;
    return f;
  })();
}

var arr = [ ];
for (var i=0; i<10000; i++) {
  arr[i] = createFn();
}
```

通过运行在Windows XP SP2中的Process Explorer可以看到如下结果：

```javascript
IE6:

  without `null`:   7.6K -> 20.3K
  with `null`:      7.6K -> 18K

IE7:

  without `null`:   14K -> 29.7K
  with `null`:      14K -> 27K
```

这个结果大致验证了我的想法——显式地清除多余的引用确实可以释放内存，但释放的内存空间相对不多。在创建10000个函数对象的情况下，大约有3MB左右。对于大型应用程序，以及需要长时间运行或者在低内存设备（如手持设备）上运行的程序而言，这是绝对需要考虑的。但对小型脚本而言，这点差别可能也算不了什么。

有读者可能认为本文到此差不多就该结尾了——实际上还差得远呢 :)。我还想再多谈一点，这些内容涉及的是Safari 2.x。

<h2 id="safari-bug">Safari中存在的bug</h2>

在Safari较早的版本——Safari 2.x系列中，也存在一些鲜为人知的与NFE有关的bug。我在Web上看到<a href="_blank" href="//meyerweb.com/eric/thoughts/2005/07/11/safari-syntaxerror/">有人说Safari 2.x不支持NFE</a> 。实际上不是那么回事。Safari确实支持NFE，只不过它的实现中存在bug而已（很快你就会看到）。

在某些情况下，Safari 2.x遇到函数表达式时会出现不能完全解析程序的问题。而且，此时的Safari不会抛出任何错误（例如SyntaxError），只会“默默地知难而退”：

```javascript
(function f(){})(); // <== NFE
alert(1); // 由于前面的表达式破坏了整个程序，因此这一行永远不会执行
```

经过多次测试，我得出一个结论：**Safari 2.x 不能解析非赋值表达式中的命名函数表达式**。下面是一些赋值表达式的例子：

```javascript
// 变量声明
var f = 1;

// 简单赋值
f = 2, g = 3;

// 返回语句
(function(){
  return (f = 2);
})();
```

换句话说，把命名函数表达式放到一个赋值表达式中会让Safari“很高兴”：

```javascript
(function f(){}); // 失败

var f = function f(){}; // 没问题

(function(){
  return function f(){}; // 失败
})();

(function(){
  return (f = function f(){}); // 没问题
})();

setTimeout(function f(){ }, 100); // 失败

Person.prototype = {
  say: function say() { ... } // 失败
}

Person.prototype.say = function say(){ ... }; // 没问题
```

同时这也就意味着，在不使用赋值表达式的情况下，我们不能使用习以为常的模式返回命名函数表达式：

```javascript
// 以下返回命名函数表达式的常见模式，对Safari 2.x来说是不兼容的：
(function(){
  if (featureTest) {
    return function f(){};
  }
  return function f(){};
})();

// 在Safari 2.x中，应该使用以下稍麻烦一点的方式：
(function(){
  var f;
  if (featureTest) {
    f = function f(){};
  }
  else {
    f = function f(){};
  }
  return f;
})();

// 或者，像下面这样也行：
(function(){
  var f;
  if (featureTest) {
    return (f = function f(){});
  }
  return (f = function f(){});
})();

/* 
  可是，这样一来，就额外使用了一个对函数的引用，而该引用还被封闭在了返回函数的闭包中。
      为了最大限度地降低额外的内存占用，可以考虑把所有命名函数表达式都赋值给一个变量。
*/

var __temp;

(function(){
  if (featureTest) {
    return (__temp = function f(){});
  }
  return (__temp = function f(){});
})();

...

(function(){
  if (featureTest2) {
    return (__temp = function g(){});
  }
  return (__temp = function g(){});
})();

/*
  这样，后续的赋值语句通过“重用”前面的引用，达到了不过多占用内存的目的。
*/
```

如果兼容Safari 2.x非常重要，就应该保证源代码中不能出现任何“不兼容”的结构。虽然这样做不免会让人着急上火，可只要抓住了问题的根源，还是绝对能够做到的。

对了，还有个小问题必须说明一下：在Safari 2.x中声明命名函数时，函数的字符串表示不会包含函数的标识符：

```javascript
var f = function g(){};

// 看到了吗，函数的字符串表示中没有标识符g
String(f); // function () { }
```

这不算什么大问题。但正如我以前说过的，<a target="_blank" href="//thinkweb2.com/projects/prototype/those-tricky-functions/">函数的反编译结果是无论如何也不能相信的</a>。

<h2 id="spidermonkey-peculiarity">SpiderMonkey的怪癖</h2>

大家都知道，命名函数表达式的标识符只在函数的局部作用域中有效。但包含这个标识符的局部作用域又是什么样子的吗？其实非常简单。在命名函数表达式被求值时，**会创建一个特殊的对象**，该对象的唯一目的就是保存一个属性，而这个属性的名字对应着函数标识符，属性的值对应着那个函数。这个对象会被注入到当前作用域链的前端。然后，被“扩展”的作用域链又被用于初始化函数。

在这里（想象一下本山大叔在小品《火炬手》中发表获奖感言的情景吧。——译者注），有一点十分有意思，那就是ECMA-262定义这个（保存函数标识符的）“特殊”对象的方式。标准说**“像调用new Object()表达式那样”**创建这个对象。如果从字面上来理解这句话，那么这个对象就应该是全局Object的一个实例。然而，只有一个实现是按照标准字面上的要求这么做的，这个实现就是SpiderMonkey。因此，在SpiderMonkey中，扩展Object.prototype有可能会干扰函数的局部作用域：

```javascript
Object.prototype.x = 'outer';

(function(){
  
  var x = 'inner';
  
  /*
    函数foo的作用域链中有一个特殊的对象——用于保存函数的标识符。这个特殊的对象实际上就是{ foo: <function object> }。
    当通过作用域链解析x时，首先解析的是foo的局部环境。如果没有找到x，则继续搜索作用域链中的下一个对象。下一个对象
    就是保存函数标识符的那个对象——{ foo: <function object> }，由于该对象继承自Object.prototype，所以在此可以找到x。
    而这个x的值也就是Object.prototype.x的值（outer）。结果，外部函数的作用域（包含x = 'inner'的作用域）就不会被解析了。
  */
  
  (function foo(){
    
    alert(x); // 提示框中显示：outer
  
  })();
})();
```

不过，更高版本的SpiderMonkey改变了上述行为，原因可能是认为那是一个安全漏洞。也就是说，“特殊”对象不再继承Object.prototype了。不过，如果你使用Firefox 3或者更低版本，还可以“重温”这种行为。

另一个把内部对象实现为全局Object对象的是**黑莓（Blackberry）浏览器**。目前，它的活动对象（Activation Object）仍然继承Object.prototype。可是，ECMA-262并没有说活动对象也要“像调用new Object()表达式那样”来创建（或者说像创建保存NFE标识符的对象一样创建）。 人家规范只说了活动对象是规范中的一种机制。

好，那我们下面就来看看黑莓浏览器的行为吧：

```javascript
Object.prototype.x = 'outer';

(function(){
  
  var x = 'inner';
  
  (function(){
    
    /*
    在沿着作用域链解析x的过程中，首先会搜索局部函数的活动对象。当然，在该对象中找不到x。
    可是，由于活动对象继承自Object.prototype，因此搜索x的下一个目标就是Object.prototype；而
    Object.prototype中又确实有x的定义。结果，x的值就被解析为——outer。跟前面的例子差不多，
    包含x = 'inner'的外部函数的作用域（活动对象）就不会被解析了。
    */
    
    alert(x); // 提示框中显示：outer
    
  })();
})();
```

虽然这有点让人不可思议，但更令人匪夷所思的则是函数中的变量甚至会与已有的Object.prototype的成员发生冲突：

```javascript
(function(){
  
  var constructor = function(){ return 1; };
  
  (function(){
    
    constructor(); // 求值结果是{}（即相当于调用了Object.prototype.constructor()。——译者注）而不是1
    
    constructor === Object.prototype.constructor; // true
    toString === Object.prototype.toString; // true
    
    // ……
    
  })();
})();
```

<h2 id="solution">解决方案</h2>

```javascript
var fn = (function(){

  // 声明要引用函数的变量
  var f;

  // 有条件地创建命名函数
  // 并将其引用赋值给f
  if (true) {
    f = function F(){ }
  }
  else if (false) {
    f = function F(){ }
  }
  else {
    f = function F(){ }
  }

  // 声明一个与函数名（标识符）对应的变量，并赋值为null
  // 这实际上是给相应标识符引用的函数对象作了一个标记，
  // 以便垃圾回收器知道可以回收它了
  var F = null;

  // 返回根据条件定义的函数
  return f;
})();
```

最后，我要给出一个应用上述“技术”的实例。这是一个跨浏览器的addEvent函数的代码：

```javascript
// 1) 使用独立的作用域包含声明
var addEvent = (function(){

  var docEl = document.documentElement;

  // 2) 声明要引用函数的变量
  var fn;

  if (docEl.addEventListener) {

    // 3) 有意给函数一个描述性的标识符
    fn = function addEvent(element, eventName, callback) {
      element.addEventListener(eventName, callback, false);
    }
  }
  else if (docEl.attachEvent) {
    fn = function addEvent(element, eventName, callback) {
      element.attachEvent('on' + eventName, callback);
    }
  }
  else {
    fn = function addEvent(element, eventName, callback) {
      element['on' + eventName] = callback;
    }
  }

  // 4) 清除由JScript创建的addEvent函数
  //    一定要保证在赋值前使用var关键字
  //    除非函数顶部已经声明了addEvent
  var addEvent = null;

  // 5) 最后返回由fn引用的函数
  return fn;
})();
```

<h2 id="alt-solution">替代方案</h2>

不要忘了，如果我们不想在调用栈中保留描述性的名字，实际上还有其他选择。换句话说，就是还存在不必使用命名函数表达式的方案。首先，很多时候都可以通过声明而非表达式定义函数。这个方案只适合不需要创建多个函数的情形：

```javascript
var hasClassName = (function(){

  // 定义私有变量
  var cache = { };

  // 使用函数声明
  function hasClassName(element, className) {
    var _className = '(?:^|\\s+)' + className + '(?:\\s+|$)';
    var re = cache[_className] || (cache[_className] = new RegExp(_className));
    return re.test(element.className);
  }

  // 返回函数
  return hasClassName;
})();
```

显然，当存在多个分支函数定义时，这个方案就不能胜任了。不过，我最早见过<a target="_blank" href="//tobielangel.com/">托比·兰吉（Tobiel Langel）</a>使用过一个很有味道的模式。他的这种模式是**提前使用函数声明来定义所有函数，并分别为这些函数指定不同的标识符:**

```javascript
var addEvent = (function(){

  var docEl = document.documentElement;

  function addEventListener(){
    /* ... */
  }
  function attachEvent(){
    /* ... */
  }
  function addEventAsProperty(){
    /* ... */
  }

  if (typeof docEl.addEventListener != 'undefined') {
    return addEventListener;
  }
  elseif (typeof docEl.attachEvent != 'undefined') {
    return attachEvent;
  }
  return addEventAsProperty;
})();
```

虽然这个方案很优雅，但也不是没有缺点。第一，由于使用不同的标识符，导致丧失了命名的一致性。且不说这样好还是坏，最起码它不够清晰。有人喜欢使用相同的名字，但也有人根本不在乎字眼上的差别。可毕竟，不同的名字会让人联想到所用的不同实现。例如，在调试器中看到attachEvent，我们就知道addEvent是基于attachEvent的实现（即基于IE的事件模型。——译者注）。当然，基于实现来命名的方式也不一定都行得通。假如我们要提供一个API，并按照这种方式把函数命名为inner。那么API用户的很容易就会被相应实现的细节搞得晕头转向。（也许是因为inner这个名字太通用，不同实现中可能都会有，因此容易让人分不清这个API到底基于哪个实现。——译者注）

要解决这个问题，当然就得想一套更合理的命名方案了。但关键是不要再额外制造麻烦。我现在能想起来的方案大概有如下几个：

```javascript
'addEvent', 'altAddEvent', 'fallbackAddEvent'
// 或者
'addEvent', 'addEvent2', 'addEvent3'
// 或者
'addEvent_addEventListener', 'addEvent_attachEvent', 'addEvent_asProperty'
```

另外，托比使用的模式还存在一个小问题，即增加内存占用。提前创建N个不同名字的函数，等于有N-1的函数是用不到的。具体来讲，如果document.documentElement中包含attachEvent，那么addEventListener 和addEventAsProperty则根本就用不着了。可是，他们都占着内存哪；而且，这些内存将永远都得不到释放，原因跟JScript臭哄哄的命名表达式相同——这两个函数都被“截留”在返回的那个函数的闭包中了。

不过，增加内存占用这个问题确实没什么大不了的。如果某个库——例如Prototype.js——采用了这种模式，无非也就是多创建一两百个函数而已。只要不是（在运行时）重复地创建这些函数，而是只（在加载时）创建一次，那么就没有什么好担心的。

<h2 id="webkit-displayName">WebKit的displayName</h2>

WebKit团队在这个问题采取了有点儿另类的策略。囿于函数（包括匿名和命名函数）如此之差的表现力，WebKit引入了一个“特殊的”displayName属性（本质上是一个字符串），如果开发人员为函数的这个属性赋值，则该属性的值将在调试器或性能分析器中被显示在函数“名称”的位置上。<a target="_blank" href="//www.alertdebugging.com/2009/04/29/building-a-better-javascript-profiler-with-webkit/">弗朗西斯科·托依玛斯基（Francisco Tolmasky）详细地解释了这个策略的原理和实现</a>。

<h2 id="future-considerations">对未来的思考</h2>

将来的ECMAScript-262第5版（目前还是草案）会引入所谓的**严格模式（strict mode）**。开启严格模式的实现会禁用语言中的那些不稳定、不可靠和不安全的特性。据说出于安全方面的考虑，arguments.callee属性将在严格模式下被“封杀”。因此，在处于严格模式时，访问arguments.callee会导致TypeError（参见ECMA-262第5版的10.6节）。而我之所以在此提到严格模式，是因为如果在基于第5版标准的实现中无法使用arguments.callee来执行递归操作，那么使用命名函数表达式的可能性就会大大增加。从这个意义上来说，理解命名函数表达式的语义及其bug也就显得更加重要了。

```javascript
// 此前，你可能会使用arguments.callee
(function(x) {
  if (x <= 1) return 1;
  return x * arguments.callee(x - 1);
})(10);

// 但在严格模式下，有可能就要使用命名函数表达式
(function factorial(x) {
  if (x <= 1) return 1;
  return x * factorial(x - 1);
})(10);

// 要么就退一步，使用没有那么灵活的函数声明
function factorial(x) {
  if (x <= 1) return 1;
  return x * factorial(x - 1);
}
factorial(10);
```

<h2 id="credits">致谢</h2>

理查德· 康福德（Richard Cornford），是他率先<a target="_blank" href="//groups.google.com/group/comp.lang.javascript/msg/5b508b03b004bce8">解释了JScript中命名函数表达式所存在的bug</a>。理查德解释了我在这篇文章中提及的大多数bug，所以我强烈建议大家去看看他的解释。我还要感谢**Yann-Erwan Perio（这是中国人吗？——译者注）**和**道格拉斯·克劳克佛德（Douglas Crockford）**，他们早在2003年就在<a target="_blank" href="//groups.google.com/group/comp.lang.javascript/msg/03d53d114d176323">comp.lang.javascript论坛中提及并讨论NFE问题了</a>。

**约翰-戴维·道尔顿（John-David Dalton）**对“最终解决方案”提出了很好的建议。

**托比·兰吉**的点子被我用在了“替代方案”中。

**盖瑞特·史密斯（Garrett Smith）**和**德米特里·苏斯尼科（Dmitry Soshnikov）**对本文的多方面作出了补充和修正。

