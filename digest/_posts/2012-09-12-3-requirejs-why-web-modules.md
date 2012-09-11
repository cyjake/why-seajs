---
title: RequireJS 其一
subtitle: 为何用 Web 模块的方式？
klass: why-web-modules
layout: post
---

### 译者序

前一阵围观 [RequireJS 的文档](http://requirejs.org/)，做得很好看，发现两篇讲设计思想的文章，
很符合我这《[Why SeaJS]({{ site.baseurl }})》的开篇立意，所以不妨拿来翻译一下，拓宽点思路。以下译文。

本文讨论为何网站模块化很有用，以及模块化的各种实现方式的可行性。同时，还有个
[独立页面]({{ site.baseurl }}/digest/#why-amd) 讨论 RequireJS
采用的函数封装的设计驱使。

### 问题

 - 网站正在变成网络应用
 - 代码复杂度随着网站变大而增加
 - 代码组织变难了
 - 开发者希望 JS 文件模块化
 - 部署时又希望将代码优化进一到数次 HTTP 请求

### 解决办法

前端开发者需要一个满足以下条件的解决方案：

 - 类似 `#include`、`import`、`require` （译注：分别对应 C、Python、node.js）
 - 能够加载嵌套的依赖
 - 对开发者友好并且能提供帮助部署的优化工具

### 脚本加载的 API

首要任务是厘清加载脚本的 API。有以下备选方案：

 - Dojo：`dojo.require('some.module')`
 - LABjs：`$LAB.script('some/module.js')`
 - CommonJS: `require('some/module')`

以上所有都将映射模块到 `some/path/some/module.js`。因为 CommonJS 的语法用者越来越广，
而我们希望代码复用，所以理想地，我们选择它。

我们还想要某种语法，使得当前的 JavaScript 文件不经修改即可被加载 —— 开发者不需要因为想用脚本加载的方式而重写所有的 JavaScript 文件。

但是，我们还需要这语法能够在浏览器中执行良好。CommonJS 里的 `require()` 是个同步的调用，意味着它得立即返回模块。
在浏览器里头，这可不太好使。

### 异步对同步

下例说明了浏览器（实现模块化）的基本问题。假设我们有个 `Employee` （职工）对象，
我们想要 `Manager`（经理）继承自 `Employee` 对象。
[以此为例](https://developer.mozilla.org/en-US/docs/JavaScript/Guide/Obsolete_Pages/The_Employee_Example/Creating_the_Hierarchy?redirectlocale=en-US&redirectslug=Core_JavaScript_1.5_Guide%2FObsolete_Pages%2FThe_Employee_Example%2FCreating_the_Hierarchy)
，用我们的脚本加载 API，我们可能会把它写成这样：

{% highlight js %}
var Employee = require("types/Employee");

function Manager () {
    this.reports = [];
}

// 如果 require 调用是异步的，这里就出错了
Manager.prototype = new Employee();
{% endhighlight %}

如上边注释所示，如果 `require()` 是异步的，这段代码就跑不起来。然而，在浏览器里同步加载脚本又严重影响性能。
那该怎么办呢？

### 脚本加载：XHR

用 `XMLHttpRequest`（XHR）加载脚本看起来很靠谱。如果用了 XHR，我们就可以调戏上例中的代码 ——
我们可以用正则表达式找出所有的 `require()` 调用，确保加载了这些脚本之后，再使用 `eval()` 或者动态创建 `script`
节点将 XHR 获取的代码塞进去来执行上例代码。

用 `eval()` 来执行模块代码很糟糕：

 - 开发者们已经被教育了 `eval()` 很糟糕
 - 有些环境不允许 `eval()`
 - 调试很困难。WebKit 的查看器，和 Firebug，都有个 `//@ sourceURL=convention` 来帮你给执行的文本命名，
   但是这个特性并不是所有浏览器都有的
 - 不同浏览器中，`eval()` 的上下文也是不同的。你可以在 IE 里改用 `execScript`，但这意味着更多的不确定部分。

用 `script` 节点插入模块代码来执行模块也不好：

 - 调试的时候，错误消息中的行号并不是指向到实际文件的

XHR 还有个问题，它不支持跨域的请求。有些浏览器支持跨域 XHR 请求，但这个特性并不是所有浏览器都支持的，
而 IE 又创造了一个不同的专门用来处理跨域请求的 API 对象，叫做 `XDomainRequest`。
越多的不确定部分意味着事情越容易变糟。特别是，你要确保不发送任何非标准的 HTTP 头信息，不然会有个预请求，以确保是可以跨域访问的。

Dojo 用的是基于 XHR 与 `eval()` 的加载器。尽管它管用，但成了开发者的沮丧之源。Dojo 有个跨域的加载器，
但是它需要模块在发布阶段改成函数包装的形式，从而可以用 `script src=""` 来加载模块。
此外，还有不少边界情况和不确定部分给开发者增加了额外的负担。

如果我们要创造一个新的脚本加载器，我们可以做得更好。

### 脚本加载：Web 工作线程

Web 工作线程或许可以作为加载脚本的另一种方式，不过：

 - 它没有很好的跨浏览器支持
 - 它是个消息传递的 API，而脚本基本上都要与 DOM 交互的，所以这意味着工作线程直通用来获取脚本内容，
   传送文本到主 `window` 然后使用 `eval`、`script` 来执行脚本。于是 XHR 会有的问题，它也都会有。

### 脚本加载：`document.write()`

`document.write()` 也可以用来加载脚本 —— 它可以从其他域名加载脚本，并且它的行为与浏览器正常加载脚本一致，
所以它是易于调试的。

但是，在异步对同步的例子中，我们的例子是不能直接执行的。理想情况是，我们在执行脚本之前就知道 `require()` 的依赖，
并保证这些依赖先被加载。但在此法中，我们不能在脚本被执行之前获取它的内容。

并且，`document.write()` 在页面加载完毕之后不管用。而取得大幅性能提升的好办法之一就是按需加载，
因为用户在下一步操作时才需要它（译注：即在可能页面加载完毕之后才加载，因此 `document.write()` 不好使）。

最后，通过 `document.write()` 加载脚本会阻滞页面渲染。当你专注于网站性能极限的时候，这种方法是不可容忍的。

### 脚本加载：`head.appendChild(script)`

我们可以按需创建节点，并插入到 `head` 中：

{% highlight js %}
var head = document.getElementsByTagName('head')[0],
    script = document.createElement('script');

script.src = url;
head.appendChild(script);
{% endhighlight %}

这段小代码还是不够的，但它已经说明了基本的想法。这个方式比 `document.write` 的优势在于它不会阻碍页面渲染，
同时在页面加载完毕之后也是可用的。

但是，它仍然有异步对同步里提到的问题：理想情况是，我们可以在执行脚本之前知道 `require()` 依赖，并确保它们先被加载好。

### 函数封装

所以我们需要知道依赖，并且保证在执行脚本之前先加载它们。最好的办法就是将我们的模块加载 API 用匿名函数封装起来。
像这样：

{% highlight js %}
define(
    // 模块名称
    "types/Manager",

    // 依赖数组
    ["types/Employee"],

    // 模块依赖加载完毕之后再执行的函数。
    // 这个函数的参数是依赖数组。
    function (Employee) {
        function Manager () {
            this.reports = [];
        }

        Manager.prototype = new Employee();

        // 返回 Manager 构造器，使其能为人所用
        return Manager;
    }
);
{% endhighlight %}

这就是 RequireJS 所采用的语法。还有一个简化的语法，方便你用来加载未使用 `define` 包装的 JavaScript 文件：

{% highlight js %}
require(["some/script.js"], function() {
    //This function is called after some/script.js has loaded.
});
{% endhighlight %}

选择这种语法的原因是它简洁，同时又允许加载器使用 `head.appendChild(script)` 这种加载方式。

为了能够在浏览器里运行良好，它与 CommonJS 的语法的不同是必须的。坊间也有建议说常规的 CommonJS
语法也可以用来以 `head.appendChild(script)` 方式加载，只要服务端能够自动将模块代码以函数封装的形式包装起来。

我相信有一点很重要，不能强制用户使用后端服务来改变代码：

 - 调试的时候会很怪，因为服务端插入了函数封装，行号会和源文件有偏差。
 - 增加了技术负担。前端开发应该只需静态文件就好。

更多关于函数封装格式的设计驱使与用例的细节，称作异步模块定义（Asynchronous Module Definition，AMD），
可以在《[为何 AMD？]({{ site.baseurl }}/digest/#why-amd)》页面找到。

