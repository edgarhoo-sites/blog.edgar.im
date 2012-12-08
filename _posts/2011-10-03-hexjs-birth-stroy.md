---
layout: post
title: HexJS诞生记
categories: [Javascript]
tags: [HexJS]
---


这篇是上周在方凳会上分享的同名[slides](http://slides.edgar.im/2011/hexjs-birth-stroy.html)的讲稿。本来应该先写这篇，再从中择几句拼成slides的。不料，现在反了一下。我始终觉得slides若作为资料存档总是不够份量的，不及文章能够沉淀

我对[HexJS](http://hexjs.edgarhoo.org/)的定位是一个页面级的module管理器，更重要的是在此基础之上所形成的一套js书写风格。那么，在只有一套书写风格的情况下，对同一功能的实现的思路、方法也许能够趋于相似，这对于团队开发、维护来说，至关重要。我对这一点深有感触，拿运营线的页面来说，无论哪位同学先开发，尔后哪位同学再接手，都不会觉得代码陌生，如同这是自己的代码一般，甚至一张页面多人同时开发再拼起来而它的代码风格也是一致的，因为大家都是基于[Pitaya](http://pitaya.edgarhoo.org/)系列约定来开发的。如此，一个团队就一套风格，不再是一人一风格

接下来讲HexJS产生前后的一些思路，或者说是产生HexJS的缘由以及原因

先来看一段代码：

{% highlight js %}
//代码1
var readyFunc = [
    function fn1(){}, //功能1
    function fn2(){}, //功能2
    function fn3(){} //功能3
];

$E.onDOMReady(function(){
    for ( var i = 0, len = readyFunc.length; i &lt; len; i++ ){
        try {
            readyFunc[i]();
        } catch(e) {
            if ( Ali.isDebug ){
                console.info( 'Error at No.' + i + '; ' + e.name + ':' + e.message );
            }
        } finally {
            continue;
        }
    }
});
{% endhighlight %}

不少同学对代码1肯定不陌生，曾经应该风行过，现在去style里搜还能搜出一大堆来。代码1的优点在于，通过try catch来保证各个功能互不干扰，哪怕某个功能失效了出bug了，也不影响其他功能的生效。这从体验上将失效的范围降到最低。有一点需要注意的是，这些功能只能是相互独立的，比较适合于一些小功能。这个风格我没使用过，因为我不喜欢每个文件底部都copy那段onDOMReady的代码

再看下一段代码：

{% highlight js %}
//代码2
(function( NS ){
    NS.xxx = {
        init: function(){
            this.fn1(); //功能1
            this.fn2(); //功能2
            this.fn3(); //功能3
        },
        fn1: function(){
            this._fn4();
        },
        fn2: function(){},
        fn3: function(){},
        _fn4: function(){}
    };

    $E.onDOMReady(function(){
        NS.xxx.init();
    }

})( NameSpace );
{% endhighlight %}

代码2的风格我经常用的，通过命名空间将所有页面都纳入到一个体系里，文件间也是经过命名空间的传递达到通信

后来新版旺铺，由琪君创造了下面这段风格：

{% highlight js %}
//代码3
(function( $, NS ){
    var ModName = { //功能1
        init: function(){
            this._fn1();
            this._fn2();
        },
        _fn1: function(){},
        _fn2: function(){}
    };

    NS.PageContext.register( 'mod-name', ModName );

})( jQuery, NameSpace );
{% endhighlight %}

当然新版旺铺的底层js还有很多功能，这里只是取一个风格。不容否认，琪君的旺铺js对我是有很大影响的，尽管我并没有真正使用过，主要还是看源码。这段js的其中一个优点在于应用层不再显式的出现DOMReady，将功能的执行统一的管理起来

至此，对我来说，已有的一定的风格累积

接下来讲转折点。其中一个是圈圈项目。从前端质量角度看，这并不是一个成功的项目，甚至因为使用了一个第三方组件，留着一个小问题就上线了。但于我而言，它的复杂程度刚好够我去思考js的写法

看一个现状，圈圈项目中的画报编辑页js：

{% highlight js %}
merge.js        //页面级出口文件
    base.js     //画报实例，全局标记变量
    util.js     //公共方法
    upload.js   //图片上传相关功能
    popup.js    //浮出层相关功能
    submit.js   //表单提交相关功能
    done.js     //页面初始化、功能执行
{% endhighlight %}

除了base.js和done.js，其他文件的行数在300至600之间，6个文件间通过FE.info.admin.editpictorial这个命名空间传递数据，采用类似代码2的风格，只不过分散在各个文件。DOMReady只在done.js中存在，其他文件的初始化、执行都依次在done.js运行

对此不满意的点有：

<ol>
    <li>文件拆分不完全
        <p>看上去文件根据功能拆分，还不错。但实际上，upload.js仍然会对util扩展方法，某些公共属性不得不在其他文件定义。</p>
    </li>
    <li>功能依赖不清晰
        <p>若其他同学要去改个东西，很难在短时间内理清楚。尽管已尽量让upload.js、popup.js、submit.js三者之间没有通信，但util的方法过多，而没有很好的分类。</p>
    </li>
    <li>某功能相关的方法跟功能主体间隔太远，不好理解
        <p>某些功能的主体和其输出接口分散在文件各处甚至两个文件，这么做的原因是要达到只允许util等公共变量才能传递数据。</p>
    </li>
    <li>曲折做到只在DOMReady之后取节点
        <p>不少节点在多文件使用，且无法在util初始化的去取，只能在done.js中取好再以入参的方式传给各文件的初始化函数。</p>
    </li>
    <li>难以做到一个节点只取一次
        <p>大部分节点可以做到只取一次，仍有不少不得不多取几次。</p>
    </li>
</ol>

**这个现状让我很不满意！**

项目时间基本上都是很紧的，根本没时间可以去重构或梳理，希望能在一开始就能够不产生这些问题

希望解决的问题：

<ul>
    <li>文件拆分可以完全</li>
    <li>文件间的信息通信、共享方便</li>
    <li>功能相互依赖清晰</li>
    <li>节点在DOMReady之后取</li>
    <li>一个节点尽可能只取一次</li>
</ul>

另一个转折点是alibar。差不多也在圈圈项目同时间，alibar的改动非常频繁，作为全站统一区块，稳定性是最重要的。在频繁改动的驱动下，原有的代码风格完全不能适应，期望功能彼此独立，连视觉上也期望彼此离得远些，于是改成如下风格：

{% highlight js %}
var checkSignIn = { //功能1
    init: function(){ this._fn(); },
    _fn: function(){ register( [ _addTip ] ); }
};

var changeDone = { //功能2
    init: function(){ this._fn(); },
    _fn: function(){}
};

var _addTip = { //功能3
    init: function(){}
};

register( [ checkSignIn, changeDone ] );
{% endhighlight %}

这样，多个功能的切换、更新换代，相对来说简单得多了，也放心得多

到这个时候，已经在想需要一套新的书写风格，但具体怎样还很模糊

若没有七月的首页改版，也许HexJS会来得晚些。不过，首页改版从四月开始闹起，势必是要改版的

对于这回首页改版，无论之前数回的胎死腹中，还是真正落地，我们都给予了很大的期待，我们想尝试的东西实在太多

那么对于js的这套书写风格也提上真正的日程

[CommonJS](http://commonjs.org/)的火爆应该没有一个前端不去关注下的，对于它所制定的一系列API规格，我是觉得也许真的会成为一个js的生态，实在没什么理由拒绝的

于是，便去找它的实现，看看哪个能直接拿来用的。其实根本不用找，[SeaJS](http://seajs.com/)、[OzJS](https://github.com/dexteryy/OzJS)、[RequireJS](http://requirejs.org/)这三个同样如雷贯耳的

OzJS文档缺失，首先排除掉，RequireJS实在有些庞大，只剩下SeaJS

从我的观点来看，SeaJS已经足够强大，这篇blog的产生也即意味着我并没有选择SeaJS，主要原因为以下三点：

<ol>
    <li>文件拆分过细
        <p>若一个module一个文件，当页面复杂时，碎文件过多。另一方面，从实际操作来讲，也难以做到一个文件只一个module，除非一个module包含数个功能，而这为我所不愿。</p>
    </li>
    <li>部署问题
        <p>上线的文件必然需要合并。虽SeaJS自有打包工具，效果似乎并不完美，况且我们已有较成熟的merge机制。对于动态加载，我认为只有组件级文件才有必要，页面级的文件实在没这个必要，而fdev-v4对组件的动态加载也同样处理得比较好了。</p>
    </li>
    <li>维护问题
        <p>在SeaJS未全站推广的情况下，其他同学来维护有使用SeaJS的页面时将比较麻烦。现在绝大部分同学仍然在windows系统下开发。</p>
    </li>
</ol>

从我的观察，觉得SeaJS不过是另一套Kissy，并不会甘心止于module loader

**其实我想要的是一种书写风格，而不是框架/库**

SeaJS更适合于初创网站，对中文站这庞大而古老且自有体系的站来说，它并不适合。提句题外话，既然fdev-v4没有选择SeaJS而选择了jQuery作为核心，那么现在fdev-v4已渐近成熟的情况下，也难以再回去使用它了

下面简单介绍下HexJS，它的特点有：

1. 专注于module机制，不涉及script加载
1. module定义与执行的分离
1. 各module的执行相互独立
1. 保留原有书写习惯
1. API简洁
1. 目前gzip之后只有766字节

HexJS只符合一部分CommonJS的Modules/1.1.1规范，具体API可以参考其[使用手册](http://hexjs.edgarhoo.org/manual.zh-cn.html)

最后分享下HexJS的命名由来：

{% highlight html %}
module -> 块 -> 积木 -> 蜂窝 -> 六边形 -> Hexagon -> HexJS
{% endhighlight %}

