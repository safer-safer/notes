# js 高级技巧

[TOC]

Js学的也差不多了，该是来总结一下Js中一些比较高级的智慧结晶了。基于Js的动态性、对象都是易变的、函数是第一对象等等其他语言所不包含的特性，可以在使用Js的时候创造出更高效、组织性更好的代码。下面提到的一些概念，是不是很熟悉：
分支、惰性实例化、惰性载入函数、单例的两种模式、享元类、函数绑定（纠正函数一个执行上下文）、函数curry化、高级定时器、保护上下文的构造函数、函数节流、自定义事件……
js中的继承、原型、构造函数这些都是老生常谈的了。但是对于构造函数式继承和原型式继承的优缺点，还是有必要了解一下的。原型式继承的主要优点就是共享方法和属性，使得从原型对象中继承出来的子对象都可以在内存中共享全部的方法和属性，这如果是在大型的继承链中将会大大的改善性能和减少内存的使用量。
 
接下来看看上面所罗列的每一个所谓的“高级”技巧的具体细节咧：

#### 1.惰性实例化

>惰性实例化所要解决的问题是这样的：避免了在页面中js初始化执行的时候就实例化了类，如果在页面中没有使用到这个实例化的对象，那么这就造成了一定的内存的浪费和性能的消耗，那么如果将一些类的实例化推迟到需要使用它的时候才开始去实例化，那么这就避免了刚才说的问题，做到了“按需供应”，简单代码示例如下：

```javascript
var myNamespace = function(){
  var Configure = function(){
    var privateName = "someone's name";
    var privateReturnName = function(){
      return privateName;
    }
    var privateSetName = function(name){
      privateName = name;
    }
    //返回单例对象
    return {
      setName:function(name){
        privateSetName(name);
      },
      getName:function(){
        return privateReturnName();
      }
    }
  }
  //储存configure的实例
  var instance;
  return {
    getInstance:function(){
      if(!instance){
        instance = Configure();
      }
      return instance;
    }
  }
}();
//使用方法上就需要getInstance这个函数作为中间量了：
myNamespace.getInstance().getName();
```

>上面的就是简单的惰性实例化的示例，但是有一点缺点就是需要使用中间量来调用内部的Configure函数所返回的对象的方法（当然也可以使用变量来储存myNamespace.getInstance()返回的实例对象）。将上面的代码稍微修改一下，就可以只用比较得体的方法来使用内部的方法和属性：

```javascript
//惰性实例化的变体
var myNamespace2 = function(){
  var Configure = function(){
    var privateName = "someone's name";
    var privateReturnName = function(){
      return privateName;
    }
    var privateSetName = function(name){
      privateName = name;
    }
    //返回单例对象
    return {
      setName:function(name){
        privateSetName(name);
      },
      getName:function(){
        return privateReturnName();
      }
    }
  }
  //储存configure的实例
  var instance;
  return {
    init:function(){
      //如果不存在实例，就创建单例实例
      if(!instance){
        instance = Configure();
      }
      //将Configure创建的单例
      for(var key in instance){
        if(instance.hasOwnProperty(key)){
          this[key]=instance[key];
        }
      }
      this.init = null;
      return this;
    }
  }
}();
//使用方式：
myNamespace2.init();
myNamespace2.getName();
```
>上面修改了自执行函数返回的对象的代码，在获取Configure函数返回的对象的时候，将该对象的方法赋给myNamespace2，这样，调用方式就发生了一点改变了。

#### 2.分支
>分支技术解决的一个问题是处理浏览器之间兼容性的重复判断的问题。普通解决浏览器之间的兼容性的方式是使用if逻辑来进行特性检测或者能力检测，来实现根据浏览器不同的实现来实现功能上的兼容，但问题是，没执行一次代码，可能都需要进行一次浏览器兼容性方面的检测，这个是没有必要的，能否在代码初始化执行的时候就检测浏览器的兼容性，在之后的代码执行过程中，就无需再进行检测了呢？答案是有的，分支技术就可以解决这个问题（同样，惰性载入函数也可以实现Lazy Definied，这个在后面将会讲到），下面以声明一个XMLHttpRequest实例对象为例子：

```javascript
//分支
var XHR= function(){
  var standard = {
    createXHR : function(){
      return new XMLHttpRequest();
    }
  }
  var newActionXObject = {
    createXHR : function(){
      return new ActionXObject("Msxml2.XMLHTTP");
    }
  }
  var oldActionXObject = {
    createXHR : function(){
      return new ActionXObject("Microsoft.XMLHTTP");
    }
  }
  if(standard.createXHR()){
    return standard;
  }else{
    try{
      newActionXObject.createXHR();
      return newActionXObject;
    }catch(o){
      oldActionXObject.createXHR();
      return oldActionXObject;
    }
  }
}();
```
>从上面的例子可以看出，分支的原理就是：声明几个不同名称的对象，但是给这些对象都声明一个名称相同的方法（这个就是关键），并给这些来自于不同的对象但是拥有相同的方法进行浏览器之间各自的实现，接着就开始进行一次浏览器检测，并经过浏览器检测的结果来决定返回哪一个对象，这样不论返回的是哪一个对象，最后名称相同的方法都作为了对外一致的接口。

>这个是在Javascript运行期期间进行动态检测，并将检测的结果返回赋值给其他的对象，并提供相同的接口，这样储存的对象就可以使用名称相同的接口了。其实，惰性载入函数跟分支在原理是非常相近的，只是在代码实现方面有差异而已。
#### 3.惰性载入函数
>惰性载入函数就是英文中传说的“Lazy Defined”，它的主要解决的问题也是为了处理兼容性。原理跟分支类似，下面是简单的代码示例：

```javascript
var addEvent = function(el,type,handle){
  addEvent = el.addEventListener ? function(el,type,handle){
    el.addEventListener(type,handle,false);
  }:function(el,type,handle){
    el.attachEvent("on"+type,handle);
  };
  //在第一次执行addEvent函数时，修改了addEvent函数之后，必须执行一次。
  addEvent(el,type,handle);
}
```

>从代码上看，惰性载入函数也是在函数内部改变自身的一种方式，这样之后，当重复执行的时候，就不会再进行兼容性方面的检测了。
#### 4.单例的两种模式
>单例模式是家喻户晓的了，也是当前最流行的一种编写方式，注明的模块模式的编写方式也是从这个思想中衍生出来的。单例模式有两种方式：一种是所谓的“门户大开型”，另外一种就是使用闭包来创建私有属性和私有方法的方式。第一种方式跟构造函数的“门户大开型”是一个样的，声明的方法和属性对外都是开放的，可以通过实例来调用。但是使用闭包来实现的单例模式，可以在一个“封闭”的作用域内声明一些不为外部所调用的私有属性和私有方法，而且还有一个很主要的功能，就是私有属性可以作为一个“数据存储器”，在闭包内声明的方法都可以访问这些私有属性，但是外部不可访问。那么重复添加、修改、删除这些私有属性所储存的数据是安全的，不受外部其他程序的影响。
#### 5.享元类
>顾名思义，“享”、“元”。就是共享通用的方法和属性，将差异比较大的类中相同功能的方法集中到一个类中声明，这样需要这些方法的类就可以直接从享元类中进行扩展，这样使得这样通用的方法只需要声明一遍就行了。这个从代码的大小、质量来说还是有一定的效益的。
#### 6.函数绑定
>函数绑定就是为了纠正函数的执行上下文，特别是函数中带有this关键字的时候，这点尤其显得重要，稍微不小心，使得函数的执行上下文发生了跟预期的不同的改变，导致了代码执行上的错误（有时候也不会出现错误，这样调试起来，会很变态）。对于这个问题，bind函数是再熟悉不过的了，bind函数的功能就是提供一个可选的执行上下文传递给函数，并且在bind函数内部返回一个函数，来纠正在函数调用上出现的执行上下文发生的变化。最容易出现的错误就是回调函数和事件处理程序一起使用了，下面是摘自《Javascript高级程序设计第二版》的一个示例：
```javascript
var handler = {
  message:"Event handler",
  handlerClick:function(e){
    alert(this.message);
  }
}
var btn = document.getElementById("my-btn");
//这句就造成了回调函数执行上下文的改变了
EventUtil.addHandler(btn,"click",handler.handlerClick);
```
>解决的办法之一，就是纠正一下handler.handlerClick执行的上下文环境，改为：
```javascript
//这样运行的很好
EventUtil.addHandler(btn,"click",function(e){
  handler.handlerClick(e);
});
```
>上面就很好的纠正了回调函数的执行上下文了。而且，也可以使用传说中的bind函数来解决：
```javascript
var bind = function(fn,context){
  return function(){
    return fn.apply(context || this,arguments);
  }
}
EventUtil.addHandler(btn,"click",bind(handler.handlerClick));// So Good！
```
#### 7.函数curry化
>函数curry化的主要功能就是提供了强大的动态函数创建的功能。通过调用另一个函数并为它传入要curry的函数和必要的参数。说白点就是利用已有的函数，再创建一个动态的函数，该动态的函数内部还是通过该已有的函数来发生作用，只是传入更多的参数来简化函数的参数方面的调用。具体示例：
```javascript
//curry function
function curry(fn){
  var args = [].slice.call(arguments,1); //这个就相当于一个存储器了。
  return function(){
    return fn.apply(null,args.concat([].slice.call(arguments,0)));
  }
}
//Usage:
function add(num1,num2){
  return num1+num2;
}
var newAdd = curry(add,5);
alert(newAdd(6));
```
>在curry函数的内部，私有变量args就相当于一个存储器，来暂时的存储在curry函数调用的时候所传递的参数值，这样跟后面的动态创建的函数调用的时候的参数合并，并执行，就得到了一样的效果了。
#### 8.高级定时器
>提到定时器，无非就是利用setTimeout/setInterval了。但问题是定时器并不是相当于新开一个线程来执行js程序，也不会说是在指定的时间间隔内就会一定执行。指定的时间间隔表示何时将定时器的代码添加到浏览器的执行队列，而不是合适实际执行代码。对此，就有这样的一个问题了：如果代码执行时间超过了定时器指定的时间间隔，那么在指定的时间里代码还是加入的执行队列，但是并没有执行，这样就会造成了无意义的代码执行，这也是使用setInterval的弊端。为此，使用setTimeout才能更好的避免这个问题，在代码本身中执行完毕了，再通过setTimeout来重新设定定时器，把代码加入到执行队列。比如：
```javascript
setTimeout(function(){
  //many code here...
  setTimeout(arguments.callee,100); //Key
},100);
```
>当然了，定时器还有很多其他的技巧和实际作用，看需求而定，更详细的解释可以查看《Javascript高级程序设计第二版》（第467页）。
#### 9.保护上下文的构造函数
>这个主要是避免构造函数在没有使用new来实例化的时候，内部的this指向错误问题。通常没有使用new的话，this一般执行window去了，因此造成了执行错误，给代码带来了灾难。使用下面的方式就可以避免这个问题：
```javascript
function myClass(name,size){
  if(this instanceof myClass){ //Key,使用instanceof来检测当前实例是否是myClass的实例化对象
    this.name = name;
    this.size = size;
  }else{
    return new myClass(name,size);
  }
}
```
>但是上面通过instanceof的方式，给继承造成了一定的困扰，因为子类并不是myClass的实例对象，所以会出现属性和方法无法被继承的方式。在说解决办法之前，先来了解一下instanceof操作符的原理：它首先会检测对象当前的原型是否指向右边的构造函数，如果找不到，就会往上一级的原型去查找，直到找到为止，并返回true，否则就返回false。

>基于上面的instanceof的原理，在继承的时候，就可以给子类的prototype原型赋于一个父类的实例化对象就行了，这样就可以在子类继承的时候绕过instanceof的检测。
#### 10.函数节流
>函数节流函数节流解决的问题是一些代码（特别是事件）在无间断的执行，这严重的影响了浏览器的性能，再没有给它设定间断来执行的话，可能造成浏览器反应速度变慢或者直接就崩溃了。比如：resize事件、mousemove、mouseover、mouseout等等事件。
>
>这个时候，就可以加入定时器的功能了，将事件进行“节流”，即是：在事件触发的时候，设定一个定时器来执行事件处理程序，这样可以很大的程度上缓解浏览器的负担，又缓冲的余地去更新页面。具体的实例可以查看支付宝中部“导购场景”的导航：http://life.alipay.com/?src=life_alipay_index_big，以及当当网首页左边的导航栏：http://www.dangdang.com/等等，这些都是为了解决mouseover和mouseout移动过快的时候加大浏览器处理的负担，特别是在涉及到有Ajax调用，而且Ajax调用是么有缓存的情况下，给服务器也造成了很大的负担。为此，函数节流就派上用场了。比如简单的示例如下（出自本人写的：http://www.ilovejs.net/lab/tween/tweener_tab_modify.html）：
```javascript
oTrigger.onmouseover=function(e){
  //如果上一个定时器还没有执行，则先清除掉定时器
  oContainer.autoTimeoutId && clearTimeout(oContainer.autoTimeoutId);
  e = e || window.event;
  var target = e.target || e.srcElement;
  if((/li$/i).test(target.nodeName)){
    oContainer.timeoutId = setTimeout(function(){
      addTweenForContainer(oContainer,oTrigger,target);
    },300);
  }
}
```
#### 11.自定义事件

>首先要说的是，这里并不是说自定义事件可以真的自定义跟mouseout、click等一样性质的“事件”。这里的自定义事件在执行的时候还是需要依赖已有的键盘、鼠标、HTML等事件来执行，或者又其他函数“触发”执行，这里的“触发”是指直接调用自定义事件中声明的某个接口方法，来轮询的执行全部相关的添加到自定义事件中的函数。

>自定义事件内部有一个“事件”存储器，根据添加的事件的类型的不同，来储存各类的事件执行函数，这样再出发这类事件的时候，就轮询执行添加到该类型下的函数。“自定义事件背后的概念是创建一个管理事件的对象，让其他对象监听那些事件”来自《Javascript高级程序设计第二版》的解释。基于自定义事件的原理，可以想象自定义事件很多时候是用于“订阅—发布—接收”性质的功能。

>文章写的有点多了，但是上面介绍的Javascript高级技巧还远不止这些，特别是在Ajax方面的一些模式和技巧都还没有介绍，何况是客户端和服务端结合的一些技巧（比如压缩、Minify、服务器“推技术”等等）。更多的有待以后了解了介绍一二。
>在前端基本技术方面，Javascript、HTML、CSS等大家都是已经掌握的差不多了，但是利用这些已有的基本技术，能否创造出不一般的应用和模式呢？这个才是在掌握了基本的能力之后接下来需要掌握的，比如下面是本人在每天晚上睡觉之前所总结的几点：

>1.学会重构代码的技术，有计划性的重构下自己之前所编写过的一些代码，加强自己掌控代码的能力。

>2.Code review。这里说的review，并不是指个人，而是对团队来说的，一个人编写的代码的想象空间有限，如果在自己编写代码完成之后，邀请其他团队内的伙伴来查看你的代码，及时发现问题以及提出更好的解决方案，这也不失为一种即时重构的方式，提高代码的质量。

>3.编写具体的功能代码之前，首先设计代码、规划代码、组织代码的模式。

>4.在代码的质量、性能、大小之间能作出合理的权衡。

>5.编写阅读性良好、一目了然、扩展性、可维护性良好、重复利用的代码，也是一门艺术。

>6.关注web前端的性能优化，包括Javascript、HTML、CSS、客户端、服务端、前端、后端等整体性的优化。

>7.最后一点或许也是最重要的：善于总结。这点比上面的任何一点都来的重要，因为上面的每一点都是出自这点的积累。

>上面我总结的几点，也是后期自己要着重提高的能力，当然了，在实际的编码方面，还有很多的东西还需要去挖掘和了解。继续革命吧，将互联网革命进行到底……

#### 12.动态绑定事件

```js
$(document).ready(function(){
    $("#createElement").click(function(){
        //统计当前页面中使有以newButton_开头的元素个数，生成ID
        id = $("[id^='newButton_']").size()+1;
        //生成新元素，追加到ID值为newElement的元素中
        $(box.getButton(id)).appendTo($("#newElement"));
        //绑定click事件,其它change事件类似
        $("#newButton_"+id).click(function(){
            box.getClick();
        });
    });
});
 
//生成一个对象盒子,面向对象思想，封装我们的函数，强烈推荐这种方法
var box = {};

box.getButton = function(id){
    return '<input type="button" value="新按钮" id="newButton_'+id+'" /><br />';
    //返回任何你需要生成的新元素
}
 
box.getClick = function(){
    alert('事件生效啦！你点击了新按钮');
    //添加任何你需要的代码
}
```