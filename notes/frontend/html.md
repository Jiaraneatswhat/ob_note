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
	- 