## 译文：JS中哪种循环最快？

原文链接：https://javascript.plainenglish.io/which-type-of-loop-is-fastest-in-javascript-ec834a0f21b9

作者：KushSavani

如有翻译不准确，请多指正。

本文主要想让读者了解：哪种循环能够匹配你的需求，并防止你再犯一些影响应用性能的低级错误。

JavaScript是web开发的一种全新代码开发体验，无论是js框架，比如Node.js、React、Angular、Vue等等，还是原生JS都拥有庞大的粉丝基础。而现代的JavaScript，循环往往是编程语言中最大的部分。这种现代JS为你提供了许多方法，来循环你要带入的数值。

但问题是，你真的知道哪种循环方式才是最适合你的吗？有许多for循环方式供你选择，比如for、for（倒序）、for...of、forEach、for...in、for...await。本文将针对以上的循环类型进行讨论。

## 哪种循环更快？

> 答案是： for...（倒序)

最令人惊讶的是，当我把它在本地计算机中进行测试时，我开始相信for（倒序）是所有循环中最快的。下面我会分享一个关于对一个包含超过一百万项元素的数组执行一次循环遍历的例子。

**声明：**console.time（）结果的准确性在很大程度上取决于你的系统配置。



![图片](https://mmbiz.qpic.cn/mmbiz_png/I7I6eaS2zG9agd5giaiaS3D7mzlvPXNKXW9QOicQ4X4dbOCNajVSrxLxQQwTOfp8RaJB8on2iba8HIEeoybl84q1ew/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**原因：**在这里，正序和反序的for循环几乎花费一样的时间，只差0.1毫秒，

因为for（倒序）计算一个起始变量，`let i = arr.length`仅一次。

当正序for循环在每一个变量的增量之后都会检查条件`i< arr.length`，这种细微的差异不必在意，你可以忽略掉。

而另一边，forEach是Array原型的方法。相较于常见的for循环，forEach和for...of需要在数组的迭代中花费更长时间。

## 循环的类型，以及在何处使用它们

大家应该都不会对这个循环感到陌生，你可以在任何需要的地方使用for循环，按照核定次数运行代码。基础的for循环是最快的，所以我们每次都应该用它？当然不是，性能不仅仅是唯一需要考虑的因素，代码可读性往往要比代码的性能更加重要，所以，就让我们选择适合的应用程序即可。

这种方法接受回调函数作为输入参数，遍历数组的每一个元素，并执行回调函数。同样的，forEach回调函数接受当前值和相应的索引。forEach允许this在回调函数中使用可选参数。



![图片](https://mmbiz.qpic.cn/mmbiz_png/I7I6eaS2zG9agd5giaiaS3D7mzlvPXNKXW9N9bVyJVHNPiaTkr4VRXNNsPibrmic4wLSoicTesjb7v8JgYxAqbgPxXUg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



注意：如果使用forEach，就无法利用JavaScript中的短路运算符。

## For...of

for...of是在ES6（ECMAScript中6）中实现标准化的。它会基于一个可迭代对象，比如array，map，set，string等创建一个循环，并且拥有更优秀的可读性。

![图片](https://mmbiz.qpic.cn/mmbiz_png/I7I6eaS2zG9agd5giaiaS3D7mzlvPXNKXWPvSoKT2VYic01pl86iat7MkNpF7kia0LILhNNe8qEUolcsk6CaiaSoP71Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



注意：即使for...of提前终止，也不要在生成器上重用for...of。应该在退出循环后关闭生成器，并再次尝试重复该生成器将不会产生进一步的结果。

## For...in

在for...in对象的所有可以列举出来的属性上迭代指定的变量。对于每个不同的属性，for...in语句除返回数字索引外，还将返回用户定义的属性的名称。

所以，for在遍历数组时最好使用带有数字索引的传统for循环。因为for...in语句会迭代除数组元素之外的用户定义属性，所以即使你已经修改了数组对象也是如此（比如添加自定义属性或方法）。



![图片](https://mmbiz.qpic.cn/mmbiz_png/I7I6eaS2zG9agd5giaiaS3D7mzlvPXNKXW1r0zdhqrdfS2pNgFJ0hVUwHkhibNCWiaT1DEsKA1SxI6ptXaBXMcDx6w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## for..of和for...in之间的区别

for..of和for...in之间的主要区别是它们迭代的内容。上面所说的for...in环路迭代的是对象的属性，而for...of循环遍历一个迭代对象的值。



![图片](https://mmbiz.qpic.cn/mmbiz_png/I7I6eaS2zG9agd5giaiaS3D7mzlvPXNKXWvTwTP5dTrNkSibn3ibSeSaAuFUwqItJd7ryFk71jWaWo2Us6G0Xcib8YQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **结论：**

for最快，但是可读性较差；

forEach比较快，可以控制内容；

for...of比较慢，但是更好用；

for...in相对较慢，也不太方便使用。



最后，建议你将代码的可读性放在优先位置。当你开始开发一个复杂的结构程序时，代码的可读性是必不可少的，但是也要考虑性能的问题。尽量避免在代码中添加多余的、不必要的代码，有时这会减慢应用程序的性能。