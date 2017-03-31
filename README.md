## Web安全之模拟XSS攻击

跨站脚本（XSS, Cross Site Script）攻击指的是，攻击者可以让某网站执行一段非法脚本。这种情况很常见，比如提交一个表单用于修改用户名，我们可以在文本框中输入一些特殊字符，比如`<`,`>`,`'`,`"`等，检查一下用户名是否正确修改了。

### XSS的攻击方式
- 反射型
> 发出请求是,XSS代码出现在URL中,作为输入提交到服务器端,服务器端解析后响应,XSS代码随着响应内容一起传回浏览器,最后浏览器解析执行XSS代码。这个过程像一次反射,故叫做反射型XSS。
- 存储型
> 存储型XSS和反射型XSS的差别在于,提交的代码会存储在服务器中(例如数据库,内存,文件系统等),下次请求页面是不用再提交XSS代码。

`XSS`一定是由用户的输入引起的，无论是提交表单、还是点击链接（参数）的方式，只要是对用户的输入不做任何转义就写到数据库，或者写到`html`，`js`中，就很有可能出错。

从一个请求发出开始，到浏览器显示内容，与`XSS`相关的有三个地方`URL、HTML、JavaScript`。至于后台方面，它分两个功能，一个是将数据写到数据库，这时候也要对数据进行转义，但不是XSS的范畴，它更多是防止数据破坏`SQL`语句的结构；另一个是从数据库读取数据，直接生成`HTML`或者以`JSON`的方式传给前端，这些数据都必须转义后才能显示到浏览器中。

## HTML字符

`HTML`本身是一个文本文档，但在浏览器中却可以显现得花样百出，是因为很多字符对于浏览器来说是有特殊含义的，比如在`<script>`中的内容，浏览器会做一些动画等等。那么对这些特殊字符进行转义，就意味着让浏览器对待它们的时候，就像普通字符一样，比如`&lg;script&gt;`这段文字在浏览器中就会正常显示为`<script>`。

### 简单的用来转义`HTML`的`JavaScript`方法

```js
function encodeHTML (a) {
  return String(a)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#39;");
};
```

那么有哪些字符需要转义呢？这里列了一些常见的。

`" --> &#34;`
`# --> &#35;`
`$ --> &#36;`
`& --> &#38;`
`' --> &#39;`
`( --> &#40;`
`) --> &#41;`
`; --> &#59;`
`< --> &#60;`
`> --> &#62;`


在 escapeHTML 方法中，我使用了别名的方式转义，因为它比较容易记一点。无论是别名还是十六进制，它们表示的含义都是一样的，比如`&amp;`和`&#38;`都表示`&`符号。想要看更具体的列表可以参考[这个网站](http://ascii.cl/htmlcodes.htm)

在浏览器收到 HTML 之后，首先会对所有的内容进行解码，它会把所有能识别的编码符号，解码成字面值。比如有

```html
<p>my name is&#58;&#32;<a href="http&#58;&#47;&#47;www.jchen.cc">名一</a></p>
```

经过浏览器解码就变成

```html
<p>my name is: <a href="http://www.jchen.cc">名一</a></p>
```

这里要说的是，浏览器只会对两个地方解码，一个是标签的**内容**（即`textContent`，除了`<script>`和`<style>`标签），另一个是标签的**属性值**。对于属性名是不会解码的。

## URL

早些时候，服务端还不支持在`URL`中直接传输`Unicode`，比如`https://i.jakeyu.top/search?q=你好`这样的地址，服务端无法识别“你好”这个值，所以必须编码之后进行传输。

那么对于 URL，我们只需要对参数的值进行编码就可以了。比如上面这个链接，编码之后就是`https://i.jakeyu.top/find?q=%E4%BD%A0%E5%A5%BD`。

如果对整个 URL 编码，那么链接就无效了。

编码的方式很简单，浏览器提供了全局的`encodeURI`方法，调用之后就可以实现转义了。

有一点很重要`encodeURI`是不会转义`:`,`/`,`?`,`&`,`=`这些在`URL`中有特殊含义的字符的，那么如果有个参数正好包含了这些字符，就不会转义，比如

```js
encodeURI('https://i.jakeyu.top/login?name=名一&from=http://other.com'); 

// -> https://i.jakeyu.top/login?name=%E5%90%8D%E4%B8%80&from=http://other.com
```


from 参数的值并没有转义，这时候，就需要用到另一个方法`encodeURIComponent`

```js
var param = encodeURIComponent('http://other.com');
encodeURI('https://i.jakeyu.top/login?name=名一&from=') + param;

// -> https://i.jakeyu.top/login?name=%E5%90%8D%E4%B8%80&from=http%3A%2F%2Fother.com
```

所以结论就是，如果要对整个 URL 进行转义，使用 encodeURI，如果对参数的值进行转义，使用 encodeURIComponent。

当动态生成的链接地址需要赋值给`href`或者`src`属性时，需要对这些地址进行`URL`转义。当然，如果服务端支持在`URL`中包含`UTF-8`的字符的话，其实不转义也不会错，这就是为什么我们平时不会太注意对表单和`URL`参数进行转义的原因，因为服务端表现良好。

## JavaScript 特殊字符

JS 中的转义都是通过反斜杠完成，有三种类型，以`'`和`"`为例

* 直接反斜杠 --> \'\"
* 十六进制 --> \x22\x27
* Unicode --> \u0022\u0027

一般情况下可以直接通过反斜杠转义，但有些字符我们不知道怎么输入，很常见的比如 Web Font，在 CSS 中可以看到类似这样的代码

```css
.glyphicon-home::before {
    content: "";
}
```
那个 content 中的值可以通过十六进制或者 Unicode 的方式来代替。

JS 转义一般用于显示用户输入的时候，比如用户输入了反斜杠，需要显示时，就必须`alert('\\');`。

## 解码顺序

当浏览器进行绘制时，首先会对 HTML 进行解码，然后是 URL，最后是执行 JS 时对它进行解码。

现在考虑这三种编码同时存在的情况

```js
<a href="javascript&#58;&#32;alert('\<https&#58;&#47;&#47;i.jakeyu.top/find?q=%E4%BD%A0%E5%A5%BD\>');">click</a>
```

首先是`HTML`解码，结果为

```html
<a href="javascript: alert('\<https://i.jakeyu.top/find?q=%E4%BD%A0%E5%A5%BD\>');">click</a>
```
然后是`URL`解码，结果为

```html
<a href="javascript: alert('\<https://i.jakeyu.top/find?q=你好\>');">click</a>
```

最后是`JS`解码，结果为

```js
<a href="javascript: alert('<https://i.jakeyu.top/find?q=你好>');">click</a>
```

单击链接后，应该会出现一个弹窗，内容是`<https://i.jakeyu.top/find?q=你好>`。

本文更多的是介绍如何防止XSS的发生，而不是它的危害。核心就是用适当的方法对 HTML, JS 进行转义。

> 来自[[Web 安全]了解XSS与防范](https://segmentfault.com/a/1190000003874852)