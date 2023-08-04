# protoc-gen-doc 自定义模板规则详解
[配套演示工程](https://github.com/csuqiyuan/custom-protoc-doc-example)

此项目中所用 proto 文件位于 `./proto` 目录下，来源于 [官方proto示例](https://github.com/pseudomuto/protoc-gen-doc/tree/master/examples/proto)

此项目中所列所有模板case文件位于 `./tmpl` 目录下

此教程均基于 markdown 文本演示
## 前言
最近有通过 proto 文件生成其接口文档的需求，而 `protoc-gen-doc` 所生成的格式不能很好地满足需要。看到它支持自定义模板，但几乎没有一篇文章能详细解释该如何自定义，该用什么字段自定义，里面各种遍历、条件语句怎么写，如何取用 ServiceMethod Options 等。

经过近一天的梳理和踩坑，产出本文，记载了自定义模板所需的常用语法。

## protoc-gen-doc 介绍

protoc-gen-doc 是一个用于生成 proto 的 protoc 插件，通过解析 proto 文件定义，快速生成多种类型的接口文档。

生成文档的原理，就是基于一个文件模板，通过解析 proto 文件的属性，填充到模板中指定位置。可以基于内置模板生成，也可以自定义模板使用。

内置模板支持：
- json
- html
- markdown
- docbook

### 使用方式：

1. 首先通过 go get 下载 `protoc-gen-doc` 二进制文件，使用时将传入此文件的路径。。为了演示方便，我在配套代码工程的根目录下放了一个 `protoc-gen-doc` 文件。 

```shell
go get -u github.com/pseudomuto/protoc-gen-doc/cmd/protoc-gen-doc
```

1. 使用 protoc 命令
```shell
protoc \
--plugin=protoc-gen-doc=./protoc-gen-doc \
--proto_path=./proto \
--proto_path=./third_party \
--doc_out=./out \
--doc_opt=markdown,index.md \
./proto/*.proto
```

参数解析：
- `--plugin`: 指定 protoc-gen-doc，并指定 protoc-gen-doc 可执行文件的路径
- `--proto_path`: 包含目标 proto 文件的目录路径,如果依赖了其他目录的 proto 文件，也要使用此参数指定其目录。例如 `third_party` 目录。
- `--doc_out`: 接口文档的输出目录
- `--doc_opt`: 指定待输出的文档模板和文件名
- 最后一行: 指定目标 proto 文件

### 指定自定义模板

如果要指定自定义模型，则可修改 `--doc_opt` 的第一个参数。

例如，我在 `./tmpl` 目录下自定义了模板文件 `case1.tmpl`,则 `--doc_opt` 应设置为：
```shell
--doc_opt=./tmpl/case1.tmpl,index.md
```

## 官方自定义模板示例

官方在这里有两个简单的自定义模板，里面可见常用用法，但没有太多解释，想编写自己的自定义模板还是难度颇大。可供参考。

[官方自定义模板示例](https://github.com/pseudomuto/protoc-gen-doc/tree/master/examples/templates)

## 数据结构与函数定义

想要理解模板，就要将其中每个占位的参数所代表的含义。

模板中所有的占位参数，均可以在源码中找到：[templates.go](https://github.com/pseudomuto/protoc-gen-doc/blob/master/template.go)

这里不需要理解源码做了什么，不需要知道它是如何运行的，只需要看一个结构体：

```golang
type Template struct {
	// The files that were parsed
	Files []*File `json:"files"`
	// Details about the scalar values and their respective types in supported languages.
	Scalars []*ScalarValue `json:"scalarValueTypes"`
}
```

这就是从 proto 解析出的结构 Files 为一个文件，从 File 进入，可以见更多细节，亦可更深入 Services、Messages 查看。

```golang
type File struct {
	Name        string `json:"name"`
	Description string `json:"description"`
	Package     string `json:"package"`

	HasEnums      bool `json:"hasEnums"`
	HasExtensions bool `json:"hasExtensions"`
	HasMessages   bool `json:"hasMessages"`
	HasServices   bool `json:"hasServices"`

	Enums      orderedEnums      `json:"enums"`
	Extensions orderedExtensions `json:"extensions"`
	Messages   orderedMessages   `json:"messages"`
	Services   orderedServices   `json:"services"`

	Options map[string]interface{} `json:"options,omitempty"`
}
```

与官方示例进行对比便可发现，那些占位的字段，皆为这里所定义的字段。这里字段名名如其意，应该无需详解。

其中`Description`字段，对应的是 proto 文件中的注释。当注释写在 service 上方时，便可从 Service 结构的 `Desctiption`字段读取。

另外，每个结构体还定义了诸多方法，也可以用于在自定义模板中调用。详情见后文。

## 基本语法

### 注解

主要以要生成的文档类型决定。

比如在 markdown 与 html 中，`<!-- -->`无法被渲染出来，则可以用 `<!-- -->` 写注解

### 输出字段值

想获取到某个字段值，可以使用 `{{}}` 符号。在一个自定义模型文件中，

`{{.}}` 初始表示 Template 结构

`{{.Files}}` 表示输出 Template 的 Files 字段。

如果想要拿到单独一个 File 进行操作，就要使用**遍历语句**进行遍历。

### 遍历语句

从 Template 中遍历 Files 列表，如下所示：

```
<!-- case1.tmpl -->
{{range .Files}}
# {{.Name}}
{{.Description}}

{{.}}
{{end}}
```

上面例子，使用 `{{Range .Files}}` 表示遍历 Files 列表，到 `{{end}}` 处结束。在 range 和 end 之间，`{{.}}` 表示 File 的结构，`{{.Name}}` 即为 File.Name。

修改 `--doc_opt=./tmpl/case1.tmpl,case1.md` 参数，查看运行结果：

![case1](https://github.com/csuqiyuan/custom-protoc-doc-example/blob/main/assert/images/case1.png)

遍历语句非常常用，它是可以嵌套的，不光 File，连 Service、Message、ServiceMethod 等结构，都需要使用 Range 遍历进入，获取其内部结构。

例如，进入File，并进入 File.Services，打印 Service 的信息：
```
<!-- case2.tmpl -->
{{range .Files}}
# {{.Name}}
{{.Description}}

{{.}}

{{range .Services}}
## ServiceName: {{.Name}}
{{.Description}}

{{.}}
{{end}} <!-- end Services -->
{{end}} <!-- end Files -->

```

这里我们遍历打印了 Service.Name、Service.Description 和 Service 结构体

修改 `--doc_opt=./tmpl/case2.tmpl,case2.md` 参数，查看运行结果：

![](https://github.com/csuqiyuan/custom-protoc-doc-example/blob/main/assert/images/case2.png)

### 参数赋值

可以看到，每次 range 内部，`{{.}}` 都变成了 range 的目标列表的一个 Item。当 range 嵌套较多时，容易混淆。可以将 Item 赋值给一个参数，在之后使用参数进行调用。

基于 case2 进行改造：
```
<!-- case3.tmpl -->
{{range .Files}}
{{$file := .}}

# {{$file.Name}}
{{$file.Description}}

{{$file}}
{{range $file.Services}}
{{$service := .}}
## ServiceName: {{$service.Name}}
{{$service.Description}}

{{$service}}
{{end}} <!-- end Services -->
{{end}} <!-- end Files -->
```

查看运行结果，可以看到用了参数的 case3 结果与 case2 完全一致：

![](https://github.com/csuqiyuan/custom-protoc-doc-example/blob/main/assert/images/case3.png)

### 条件语句
使用 `{{if}} {{end}}`，也可以在其中加入 `{{else}}`

`{{if}}` 目前发现有两种用法：
```
{{if eq param1 param2}}
{{end}}

{{if boolParam}}
{{end}}
```
前者对比两个参数是否相等。后者判断 boolParam 是否为 true。

下面示例基于 case3，将判断 file.Name 是否等于 `Booking.proto`，若是，则输出相关信息，否则输出`忽略 xxx`

在 `Booking.proto` 中继续判断 HasServices 是否存在 service，如果存在则输出相关信息

```
<!-- case4.tmpl -->
{{range .Files}}
{{$file := .}}
{{if eq $file.Name "Booking.proto"}}
# {{$file.Name}}
{{$file.Description}}

{{$file}}
{{if $file.HasServices}}
{{range $file.Services}}
{{$service := .}}
## ServiceName: {{$service.Name}}

{{$service.Description}}

{{$service}}
{{end}} <!-- end Services -->
{{end}} <!-- end if -->
{{else}}
# 忽略 {{$file.Name}}
{{end}} <!-- end if else -->
{{end}} <!-- end Files -->
```

![case4](https://github.com/csuqiyuan/custom-protoc-doc-example/blob/main/assert/images/case4.png)

### 调用函数
想知道某个函数的功能，还要亲自阅读一下源码关于各结构体函数的实现：[templates.go](https://github.com/pseudomuto/protoc-gen-doc/blob/master/template.go)

使用方式：
```
{{functionName params... }}
```

函数大多是用来读取 Options 内容的。几乎每个结构都有 Options，为 `map[string]interface{}` 结构。

比较常用的 Options，如下图对 Service 进行 http 的定义：

![](https://github.com/csuqiyuan/custom-protoc-doc-example/blob/main/assert/images/options.png)

为了取出这里的 Options，可以使用 ServiceMethod 的 `Option` 方法。

基于 case3 进行改造：

```
<!-- case5.tmpl -->
{{range .Files}}
{{$file := .}}
# {{$file.Name}}
{{$file.Description}}

{{$file}}
{{range $file.Services}}
{{$service := .}}
## ServiceName: {{$service.Name}}
{{$service.Description}}
{{range $service.Methods}}
{{$method := .}}
### ServiceMethod: {{$method.Name}}
{{range ($method.Option "google.api.http").Rules}}
- {{.Method}}
- {{.Pattern}}
- \{{.Body}} <!-- body 为 * 时会被认为是 markdown 语法 -->
{{end}} <!-- end Rules -->
{{end}} <!-- end Methods -->
{{end}} <!-- end Services -->
{{end}} <!-- end Files -->
```

运行后结果:

![](https://github.com/csuqiyuan/custom-protoc-doc-example/blob/main/assert/images/case5.png)

`($method.Option "google.api.http").Rules` 对取出的 `interface{}` 类型做了类型转换。为什么用 Rules 来转换，可以在[源码](https://github.com/pseudomuto/protoc-gen-doc/blob/master/extensions/google_api_http/google_api_http.go)中找到：

![](https://github.com/csuqiyuan/custom-protoc-doc-example/blob/main/assert/images/rules.png)

### 其他

```
<!-- 设置默认值 -->
{{ .XXX | default ""}} 

<!-- 取值小写 -->
{{ .XXX | lower}} 

<!-- 替换 a 字符为空 -->
{{ .XXX | replace "a" ""}} 

<!-- 取值小写并替换 a 字符为空 -->
{{ .XXX | lower | replace "a" ""}} 
```

除此之外，还有不少其他用法，暂时没有更深入探索，但掌握以上已经能够满足自定义模板之需求。

如看者有其他补充，不胜感激。

# 参考文档汇总

1. 配套演示工程 [https://github.com/csuqiyuan/custom-protoc-doc-example](https://github.com/csuqiyuan/custom-protoc-doc-example)
2. proto 文件官方示例 [https://github.com/pseudomuto/protoc-gen-doc/tree/master/examples/proto](https://github.com/pseudomuto/protoc-gen-doc/tree/master/examples/proto)
3. 自定义模板官方示例 [https://github.com/pseudomuto/protoc-gen-doc/tree/master/examples/templates](https://github.com/pseudomuto/protoc-gen-doc/tree/master/examples/templates)
4. templates.go [https://github.com/pseudomuto/protoc-gen-doc/blob/master/template.go](https://github.com/pseudomuto/protoc-gen-doc/blob/master/template.go)
5. google.api.http Option 类型转换 [https://github.com/pseudomuto/protoc-gen-doc/blob/master/extensions/google_api_http/google_api_http.go](https://github.com/pseudomuto/protoc-gen-doc/blob/master/extensions/google_api_http/google_api_http.go)