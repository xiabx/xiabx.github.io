---
title: JQuery 笔记
author: XIA
date: 2019-05-30 19:18:22
categories:
- 前端
tags:
- js&JQuery
---
# $()方法
- 该方法接收一个css的选择符，通过该选择符获取对象。[css选择符](http://www.w3school.com.cn/cssref/css_selectors.asp)
- 获取到的对象如果不是一个单对象，可以自动遍历里面的结果。
- 通过$()方法获取到的对象是JQuery对象。可以通过[index]或.get(index)获取对应的DOM对象。
- 将DOM对象转化为Jquery对象：使用$()方法，参数传递一个DOM对象就可以将其包装为JQuery对象。
- JQuery对象和Dom对象的方法无法相互访问。
- filter()方法:过滤选择器的元素
- next(),nextAll(),prev(),prevAll()：选择对象的下一个或上一个。或者是下面所有或者是上面所有的同级元素。
- siblings():选择同级下所有元素。



# JQuery中操作元素的类

- 获取到元素的Jquery对象后，调用对应的方法可以直接增删元素的类
  - addClass():传入类名，增加类
  - removeClass()：当传入类名时删除相应的类，当不传参数时，移除元素上所有的类
  - toggleClass():传入类名，当元素上存在该类时则删除该类，当元素上不存在该类时则添加该类。
  - hasClass()方法：检测元素是否包含指定的类



# 事件

- 在加载后执行任务：通过$(document)对象调用ready()方法传入一个函数作为参数，即可注册在页面加载完以后执行的事件。
  ```js
    $(document).ready(function () {
        console.log("hello world");
    })
  ```
- JQuery注册事件:
  - .on()方法：两个参数，第一个为事件类型，第二个为事件执行的函数
  - .click()方法：可以直接注册点击事件，类似的还有.keydown()等方法，目的是减少使用on()方法注册事件的代码量。
    ```js
    //第一种事件注册方式
    $("#item1").on("click",function () {
        console.log("item1 has click");     
    })
    //第二种事件注册方式
    $("#item1").click(function () {
        console.log("spring coming");  
    })
    ```
- 由于js中事件执行时this默认执行触发的元素，所以使用`$(this)`可以获取触发事件元素的JQuery对象
- JQuery注册的事件都是冒泡类型的
- JQuery注册的事件函数中的event对象一样可以取消事件的冒泡行为和取消元素的默认事件
- .is()方法：传递一个选择符，检测对象是否匹配is方法中的选择符。
- .hasClass()方法：检测元素是否包含指定的类
- 取消元素绑定的事件：.off()方法，传递一个事件类型参数即可取消像相应的事件。当绑定同一类型的事件绑定了多个函数时，可以使用命名空间的方法:
  ```js
  //绑定事件，都是click类型事件，事件类型后面的点为名字空间。
  $("#xxx").on("click.name1",fun1);
  $("#xxx").on("click.name2",fun2);
  //取消事件，只取消一个事件
  $("#xxx").off("click.name1");
  ```
- 模拟事件：使用js触发事件，模拟用户触发的行为。
  - .trigger()方法:传递一个事件类型，模仿该事件触发
  - .click()方法：类似on方法的简写形式，该方法为trigger()方法的简写
  ```js
  //模仿#out元素的click事件被触发
  $("#out").trigger("click");
  //上面的简写形式
  $("#out").click();
  ```



# 样式和事件

- 获取元素的css属性：.css()方法。该方法可以获取和设置属性。
   - 获取属性，当获取一个属性时出入要获取的属性的名字。当获取多个属性时，传入要获取的属性所组成的数组，返回一个对象由属性名和属性值所组成。
   - 设置属性，当设置一个属性时传入两个参数，属性名和属性值。当设置多个属性时传入一个对象，该对象由要设置的属性名和属性值组成。
- 隐藏和显示元素：当方法传递参数为空时，立即隐藏或显示。当传递一个参数时可以设置一个速度，实现一个动画效果。
  - 隐藏：hide()方法，设置元素的display:none。
  - 显示：show()方法，设置元素的display属性为隐藏之前的值。
- 淡入淡出效果：也可以传递一个参数设置一个速度。
  - 淡入：fadeIn()方法，实现淡入
  - 淡出：fadeOut()方法，实现淡出
- 垂直滑上滑下效果：也可传递参数设置速度
  - 滑下：slideDown()方法，实现滑下
  - 滑上：slideUp()方法，实现滑上
- 切换可见性：toggle(),fadeToggle(),slideToggle()可以切换可见性，有的时候隐藏，没有时候显示



# 操作DOM



## 操作属性

- 获取和设置元素的属性：arrt()方法。当获取属性时传入一个属性名可以获得改属性名的属性值。当设置属性时可以传递两个参数，一个属性名一个属性值，或一个键值对象同时设置多个属性。
- attr()和prop()方法：主要区别在没有属性值的属性名上，prop()方法会返回true和false，而attr()方法当元素有该属性时返回属性名，没有该属性时返回undefined。
- 删除属性：.removeAttr()，传递属性名则可删除该属性，属性名和值一起删除。
- 获取表单控件的value值使用：.value()方法



## DOM树操作

- 创建新元素：$()函数，传入新元素的字符串表示，即可创建新元素。` $("<div id=\"d1\">hello insert</div>") `
- 对元素进行插入（正向）：
  - .insertBefore():在现有元素外部、之前添加内容
  - .insertAfter():在现有元素外部、之后添加内容
  - .prependTo():在现有元素内部、之前添加元素
  - .appendTo():在现有元素内部、之后添加元素
- 对元素进行插入（反向）：
  
  - before(),after(),prepend(),append()
- 移动元素，使用上述方法，操作位置。 
  ```js
        //a加入到b的外部、之前
        $("#a").insertBefore($("#b"));
        //a加入到b的外部、之后
        $("#a").insertAfter($("#b"));
        //a加入到b的内部、之前
        $("#a").prependTo($("#b"));
        //a加入到b的内部、之后
        $("#a").appendTo($("#b"));

        //b加入到a的外部、之前
        $("#a").before($("#b"));
        //b加入到a的外部、之后
        $("#a").after($("#b"));
        //b加入到a的内部、之前
        $("#a").prepend($("#b"));
        //b加入到a的内部、之后
        $("#a").append($("#b"));
  ```
- 显式迭代器：.each()方法，该方法接收一个函数，该方法会对匹配的元素集中每个元素都调用一次。
- 包装元素：.wrap()方法，接收一个节点的字符表示，对调用的元素集的每个元素每个包装。.wrapAll()对调用的元素集的元素全部包装包装
```html
    <div id="a">hello world</div>
    <div id="b">hello spring</div>

    $("div").wrapAll("<ul></ul>");
    $("div").wrap("<li></li>");

结果：
    <ul>
        <li><div id="a">hello world</div></li>
        <li><div id="b">hello spring</div></li>
    </ul>
```
- 复制元素：.clone()方法，接受一个jQuery对象，生成他的副本。
- .find()方法：在元素集中匹配相应的选择符表达式，返回匹配到的元素的集合。在之后调用.end()方法，返回find()方法之前的元素集。
- html()和text()方法：
  - 取值：html()取得包含html标签的值。text()只取得文本值，忽略html标签。
  - 赋值：html()会解析html标签，text()原样输出，包括html标签。
- 替换元素：replaceWith()和replaceAll()方法
  - replaceWith()：用括号中的字符替换所选择的元素
  - replaceAll()：用字符串替换括号中所选择的元素
  ```js
        $("#d1").replaceWith("spring");
        $("<span>summer</span>").replaceAll($("#d1"));
  ````
- .empty():移除匹配元素的子元素
- .remove()和datach():都是删除匹配的元素，方法执行后返回被删除元素的jquery对象。区别：前一个方法删除后返回的对象连同事件一起删除，后一个则会保留事件。



# AJAX

- 追加HTML：.load()方法，可以加载一个html文件，将其追加在元素中。
  ```js
  //在content元素内部，追加一个名为template.html的文件，这个文件中保存的是一个html片段。
  $("#content").load("template.html");
  ```
- 读取和解析json文件： $.getJSON()接收两个参数，一个为json文件的位置，另一个为回调函数（接受一个data参数，为解析后的json对象）。
- $.each():遍历方法，遍历对象或数组的值。接受两个参数一个是要遍历的数组或对象，另一个为回调函数。
  ```js
        //解析文件中的json，异步方法。后一个参数为回调函数
        $.getJSON("./o.json",function (data) {
            //遍历方法，遍历data的值
            //遍历的参数为：第一个为数组或对象的索引，第二个为对应的索引对应的值
            $.each(data,function (index,entry) {
                console.log(index+"  "+entry);
            })
        });
  ```
- 发送get或post请求,都是异步的。
  ```js
        var requestData = {
            name:"xia"
        }
        //发送get请求，第一个参数为链接，第二个参数为查询参数，第三个为回调方法
        $.get("http://www.baidu.com",requestData,function (data) {
            console.log(data);
        })
        //发送post请求，参数同get
        $.post("http://www.baidu.com",requestData,function (data) {
            console.log(data);
        })
  ```
- 序列化表单：.serialize()方法。表单对象调用该方法即可完成表单的序列化，序列化后的格式为`name1=value1&name2=value2`
- $.ajax(settings):发送ajax请求。settings为设置的对象。[settings配置](https://www.jquery123.com/jQuery.ajax/)
- $.ajaxSetup(settings):设置ajax的默认请求设置。除非在ajax()方法中明确覆盖设置，否则就是以此方法设置的请求设置进行请求。