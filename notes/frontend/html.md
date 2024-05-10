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
- 效果
![[html1.png]]
### 1.2.2 段落标签
- `<p></p>`
- 浏览器会自动在每个 `<p>` 元素前后添加一个空行
![[html2.png]]
### 1.2.3 行级元素和块级元素
- 行级元素不会另起一段，所占空间与内容大小有关
- 块级元素会另起一段，占用当前