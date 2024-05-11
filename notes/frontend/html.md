- `hyper-text markdown language`
# 1 基础
## 1.1 页面结构
```html
<!DOCTYPE html> 
<html lang="en">  
<head> 
    <meta charset="UTF-8">  
    <title></title>  
</head>  
<body>  
  
</body>  
</html>
```
- `<!DOCTYPE> html`
	- 所有 `HTML` 文档以 `<!DOCTYPE>` 声明开始
	- 浏览器据此得知自己将要处理的是 `HTML` 内容
- `<html></html>`
	- 表示 `HTML` 文档的根，表示 `HTML` 部分的开始
	- 是所有其他 `HTML` 元素的容器
- `<head></head>`
	- 元数据的容器，位于 `<html>` 和 `body` 之间
	- `<head>` 内可以放
		- `<title>`
		- `<style>`
		- `<base>`
		- `<link>`
		- `<meta>`
		- `<script>`
		- `<noscript>`
- `<meta>` 标签定义关于 `HTML` 文档的元数据
	- 用于指定字符集，页面描述，关键词，文档作者等
	- 属性
		- `charset` 指定 `HTML` 的字符编码
		- `content` 规定与 `http-equiv` 或 `name` 属性关联的值
		- `http-equiv` 为 `contnet` 属性的信息/值提供 `HTTP` 标头
			- `content-security-policy`
			- `content-type`
			- `default-style`
			- `refresh`
		- `name` 规定元数据的名称
			- `application-name`
			- `author`
			- `description`
			- `generator`
			- `keywords`
			- `viewpoint`
- `<title></title>` 定义文档的标题，显示在浏览器的标题栏或页面的选项卡中
- `<body></body>` 定义文档的主体
	- 紧跟在 `<head>` 之后
	- 包含 `HTML` 文档的所有内容
## 1.2 常用标签
### 1.2.1 标题标签
- `<h1></h1> - <h6></h6>`
```html
<h1>Heading 1</h1>
<h2>Heading 2</h2>
<h3>Heading 3</h3>
<h4>Heading 4</h4>
<h5>Heading 5</h5>
<h6>Heading 6</h6>
```
![[html1.png]]
### 1.2.2 段落标签
- `<p></p>`
- 浏览器会自动在每个 `<p>` 元素前后添加一个空行
```html
<p>这是一个段落。</p>
<p>这是一个段落。</p>
<p>这是一个段落。</p>
```
![[html2.png]]
### 1.2.3 行级元素和块级元素
- 行级元素不会另起一段，所占空间与内容大小有关
	- 包含 `<span>, <img>, <a>...`
- 块级元素会另起一段，占用当前整行
	- 包含 `<div>, <h>, <p>, <form>...`
### 1.2.4 文字显示相关标签
- `<strong></strong>` 或 `<b></b>` 可以加粗文字
```html
<p>这是普通文本 - <b>这是粗体文本</b>。</p>
<p>这是普通文本 - <strong>这是粗体文本</strong>。</p>
```
![[html3.png]]
### 1.2.5 图片与超链接
- 使用 `<img src="" alt="">` 将图片嵌入到网页中
	- `alt` 是图像的替代文本
	- 也可以指定图像的宽高
```html
<img src="jiaran.png" alt="jiaran" width="150" height="150"/>
```
![[html4.png]]
- 使用 `<a></a>` 搭配它的 `href` 属性从一张页面链接到另一张页面
- <font color = '#0000ff'><u>未访问过的链接是蓝色带下划线的</font></u>
- <font color = 'purple'><u>已访问过的链接是紫色带下划线的</font></u>
- <font color = 'red'><u>正在访问的链接是红色带下划线的</font></u>
```html
<p>在新窗口或选项卡中打开链接：<a href="https://www.jiaran.online" target="_blank">访问 jiaran.online！</a></p>
```
![[html5.png]]
- 也可以用图像作为链接
```html
<!-- 需要加协议-->
<a href="https://www.baidu.com">  
    <img src="jiaran.png" alt="jiaran" width="50" height="50"/>  
</a>
```
![[html6.png]]
- `<a>` 的属性
	- `download` 单击下载
	- `href` 的值可以是任何有效文档的相对或绝对 `URL`，包括 `JS` 片段
	- `ping` 以空格分隔的 `URL` 列表，链接被访问时，浏览器将发送带有 `ping` 正文的 `POST` 请求
	- `target` 规定在何处打开被链接文档
		- `_blank` 打开一个空白页
		- `_parent` 当前窗口打开
		- `_self` 同窗口打开
		- `_top` 顶端打开窗口
		- 网页没有框架时，除 `_blank` 外三者效果几乎相同
### 1.2.6 列表标签与表格标签
- 通过 `<ul>` 搭配 `<li>` 来创建无序列表
```html
<ul>
  <li>咖啡</li>
  <li>茶</li>
  <li>牛奶</li>
</ul>
```
![[html7.png]]
- 也可以进行嵌套
```html
<ul>
  <li>咖啡</li>
  <li>茶
    <ul>
      <li>普洱</li>
      <li>绿茶</li>
    </ul>
  </li>
  <li>牛奶</li>
</ul>
```
![[html8.png]]
- 通过 `<ol>` 搭配 `<li>` 来创建有序列表
```html
<ol>
  <li>咖啡</li>
  <li>茶</li>
  <li>牛奶</li>
</ol>

<ol start="50">
  <li>咖啡</li>
  <li>茶</li>
  <li>牛奶</li>
</ol>
```
![[html9.png]]
- 通过 `<table>` 创建表格标签
	- 一个 HTML 表格由一个 `<table>` 元素和一个或多个
		- `<tr>` 元素定义行
		- `<th>` 元素定义标题单元格
		- `<td>` 元素定义单元格
		- 组成
	- 还可以包括以下元素：
		- `<caption>` 定义表的标题
		- `<colgroup> ` 规定表格中一列或多列分组的格式``
		- `<thead>, <tbody>, <tfoot>` 三者搭配对表格中的标题内容分组
```html
<table>
  <tr>
    <th>月份</th>  <!-- 标题单元格 -->
    <th>储蓄</th>
  </tr>
  <tr>
    <td>一月</td>
    <td>￥3400</td>
  </tr>
  <tr>
    <td>二月</td>
    <td>￥4500</td>
  </tr>
</table>
```
![[html10.png]]
### 1.2.7 表单与输入标签
- 通过 `<form>` 创建表单
- 用于接收用户输入创建 `HTML` 表单，可以包含一个或多个表单元素：
	- `<input>` 
	- `<textarea>`
	- `<button>`
	- `<select>`
	- `<option>`
	- `<optgroup>`
	- `<fieldset>`
	- `<label>`
	- `<output>`
#### 1.2.7.1 \<input>
- 用户可以在其中输入数据，显示方式取决于 `type` 属性：
##### 1.2.7.1.1 button
- `button` 在点击时激活 `js`
```html
<form>  
    <input type="button" value="click me!" onclick="msg()">  
</form>  
<script>  
    function msg() {  
        alert("Hello world!");  
    }  
</script>
```
![[button.png]]
##### 1.2.7.1.2 checkbox
- 创建一个复选框
```html
<form>
  <input type="checkbox">
  <label> 我要玩原神</label><br>
  <input type="checkbox">
  <label> 我要玩王者</label><br>
  <input type="checkbox">
  <label> 我要玩D5</label><br><br>
</form>
```
![[checkbox.png]]
##### 1.2.7.1.3 color
- `color` 从颜色选择器中选择一种颜色
```html
<form>
  <label>选择您最喜欢的颜色：</label>
  <input type="color" value="#ff0000">
</form>
```
![[color.png]]
##### 1.2.7.1.4 password
- `password` 用掩码隐藏字符
```html
<form>
  <label for="email">电子邮件：</label>
  <input type="email" id="email" name="email"><br><br>
  <label for="pwd">密码：</label>
  <input type="password" id="pwd" name="pwd" minlength="8"><br><br>
</form>
```
![[pwd.png]]
##### 1.2.7.1.5 radio
- `radio` 创建一个单选按钮
```html
<form action="/demo/action_page.php">

  <p>请选择您的年龄：</p>
  <input type="radio" id="age1" name="age" value="30">
  <label for="age1">0 - 30</label><br>
  <input type="radio" id="age2" name="age" value="60">
  <label for="age2">31 - 60</label><br>  
  <input type="radio" id="age3" name="age" value="100">
  <label for="age3">61 - 100</label><br><br>
</form>
```
![[radio.png]]
##### 1.2.7.1.6 text
- 文本输入
```html
<form action="/demo/action_page.php">
  <label for="phone">请输入电话号码：</label><br><br>
  <input type="tel" id="phone" name="phone" placeholder="138-1234-5678" pattern="[0-9]{3}-[0-9]{4}-[0-9]{4}" required><br><br>
  <small>格式：138-1234-5678</small><br><br>
  <input type="submit">
</form>
```
![[text.png]]
