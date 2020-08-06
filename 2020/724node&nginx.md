## Linux升级node版本

`node --version` 6.x 版本过低，目标升级到8.0
`npm install -g n` 安装n模块
`n 8.0.0` 升级到了8.0.0
`node --version` 成功升级到8.0

## nginx部署前端项目的坑

### 场景描述

我自己开发个人项目的时候，是将前端项目打包后托管到后端静态资源文件夹下，即将`dist`文件夹放在后端项目的public静态资源目录下。这样我只需要跑后端项目即可，通过`pm2`进行进程管理，访问的时候输入后端地址就能显示前端界面。这样的做法对于我这种不会使用`nginx`的人来说，着实省去了配置的烦恼。

后来因为工作中的需要，一个项目基本是前后端分离开发，最后也是需要分开部署的，所以就试着通过配置`nginx`来部署前端项目。

### nginx拒绝访问

这个问题是花费时间最多的，主要原因是`nginx`配置不正确，正确配置如下：

```bash
server {
    listen       8080;
    server_name  0.0.0.0;

    location / {
        root   /root/zaixianshang-2020-1-7/view/dist;
        index  index.html index.htm;
    }
}
```

如果IP映射了域名，可以讲`server_name`改为域名。

### nginx提示404

配置路径不对，进入dist目录，再通过`pwd`获取路径之后，添加到`nginx.conf`里面。

### nginx提示403

将`nginx.conf`文件第一行的`#root nobody`注释去掉并改为`root root`。

## 使用云服务器拒绝访问

### 问题描述

我将前端项目上传至服务器，然后运行开发者模式，浏览器输入ip加端口号访问却拒绝访问。原因是云服务器是基于docker开发的，而docker的localhost并不是127.0.0.1，而是0.0.0.0。因此我修改`package.json`文件，添加`--host 0.0.0.0`之后再启动就能访问了。

## 服务器一直无法安装node

原因是后端同学再创建虚拟云服务器的时候，前后端机器用的是同一个镜像。后面问了其他组同学，他告诉我前端用`alphacloud/centos7_2-lnmp`镜像，创建的容器就自带node。

## 安装n模块后，切换node版本无效

**原因：**上述容器里的node安装路径，和我n模块路径不同，所以n命令无法操控系统自带node。

**解决：**

- 查看`node`安装路径：

```shell
which node
# /opt/soft/node-v8.4.0-linux-x64/bin/node
```

- 查看`n`安装路径：

```shell
which n
# /usr/bin/n
```

可以看出两者路径不同，接下来就是要将`n`模块安装`node`的路径和系统自带`node`修改一致。

- 使用vim编辑器`vi .bash_profile`文件，在结尾处添加两行：

```bash
export N_PREFIX=/opt/soft/node-v8.4.0-linux-x64
export PATH=$N_PREFIX/bin:$PATH
```

- 保存，刷新文件：

```shell
source .bash_profile
```

- 重新安装

```shell
n stable
```

之后就可以正常使用n模块管理node版本了。

## vue使用params传参刷新会丢失

因为我传递的是一个对象，页面跳转为了美观使用了params，但是众所周知params传参刷新会丢失。同时我又不想再让后端增加一个接口，通过query传递id再调用接口获取这个对象，所以这次就想找个办法彻底解决这个问题。

- 首先定义路由：

```typescript
{
    name: 'messageDetail',
    path: '/messageDetail/:message',
    component: () => import('./pages/MessageDetail.vue')
}
```

- 传递参数：

```typescript
this.$router.push({ path: '/messageDetail/' + encodeURIComponent(JSON.stringify(this.message)) });
```

- 接受参数：

```typescript
this.message = JSON.parse(decodeURIComponent(this.$route.params.message));
```

## vue子组件通知富组件

## axios的delete传参格式

使用axios的delete方法时，第二个参数直接传的数据，但是一直不起作用，后来发现和post，put不一样。

```javascript
axios.request(config)

axios.get(url[, config])

axios.delete(url[, config])

axios.head(url[, config])

axios.post(url[, data[, config]])

axios.put(url[, data[, config]])

axios.patch(url[, data[, config]])

```

`delete`需要通过`config`来传递数据：

```javascript
axios.delete(url, { data: MyData })
```

不过这样使用后，需要注意参数格式，后端明确的格式是表单格式，因此我这样写：

```javascript
axios.delete('/v1/delData', { data: new FormData(params) })
```

但是会报错，提示格式不对，ERROR TypeError: Failed to construct 'FormData': parameter 1 is not of type 'HTMLFormElement。因此换了一种写法：

```javascript
let formData = new FormData()
for (const key in params) {
  formData.append(key, params[key])
}
axios.delete('/v1/delData', { data: new FormData(params) })
```

这样写之后就可以成功发请求了，至于为什么第一种方式不行，是因为给FormData()构造函数传进去的是一个对象，而表单提交的数据是`index.html?username=chuckliuyu&useracc=2`这种键值对形式传递的，不会正确转换格式。

除了`FormData`，还有`Blob`等数据格式也是通过`append`键值对的形式添加数据，而不能直接初始化。

## axios实例设置请求数据格式

对于`post`请求我统一做了如下配置：

```typescript
// post请求头的设置
instance.defaults.headers.post['Content-Type'] = 'application/x-www-form-urlencoded;charset=UTF-8';
```

对于`axios post`数据格式，有以下4种方式：

- `application/x-www-form-urlencoded`

这是常见的`post`数据格式，浏览器的原生 <form> 表单，如果不设置 `enctype` 属性，那么最终就会以 application/x-www-form-urlencoded 方式提交数据。

- ### multipart/form-data

这又是一个常见的 POST 数据提交的方式。我们使用表单上传文件时，必须让 <form> 表单的 `enctype` 等于 multipart/form-data。

这种方式一般用来上传文件，各大服务端语言对它也有着良好的支持。

上面提到的这两种 POST 数据的方式，都是浏览器原生支持的，而且现阶段标准中原生 <form> 表单也只支持这两种方式（通过 <form> 元素的 `enctype` 属性指定，默认为 `application/x-www-form-urlencoded`。其实 `enctype` 还支持 `text/plain`，不过用得非常少）。

- ### application/json

如果使用这种编码方式，那么传递到后台的将是序列化后的json字符串。application/x-www-form-urlencoded上传到后台的数据是以key-value形式进行组织的，而application/json则直接是个json字符串。

- text/xml

这种格式一般用的很少，因为它的标签会占用不少内容，所以使得有效信息比很低。

## axios中请求变成OPTIONS的几种解决方案

在使用axios发请求的时候，发现一个接口调用了两次，虽然是一个小项目，但是对于我这种有强迫症的人来说着实受不了。如果一个项目很大，每一次请求接口都调用2次的话，是很损耗网络资源的，所以我就想着去解决它。

### 为什么会变成`OPTIONS`?

根本原因是由于跨域导致的，跨域资源共享（ [CORS](https://developer.mozilla.org/zh-CN/docs/Glossary/CORS) ）机制允许 Web 应用服务器进行跨域访问控制，从而使跨域数据传输得以安全进行。现代浏览器支持在 API 容器中（例如 [`XMLHttpRequest`](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest) 或 [Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) ）使用 CORS，以降低跨域 HTTP 请求所带来的风险。

跨域资源共享标准新增了一组 HTTP 首部字段，允许服务器声明哪些源站通过浏览器有权限访问哪些资源。另外，规范要求，对那些可能对服务器数据产生副作用的 HTTP 请求方法（特别是 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET) 以外的 HTTP 请求，或者搭配某些 MIME 类型的 [`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST) 请求），浏览器必须首先使用 [`OPTIONS`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/OPTIONS) 方法发起一个预检请求（preflight request），从而获知服务端是否允许该跨域请求。服务器确认允许之后，才发起实际的 HTTP 请求。在预检请求的返回中，服务器端也可以通知客户端，是否需要携带身份凭证（包括 [Cookies ](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies)和 HTTP 认证相关数据）。

具体可参见[CORS预检请求]([https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS#%E9%A2%84%E6%A3%80%E8%AF%B7%E6%B1%82](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS#预检请求))

- 使用`URLSearchParams`

**`URLSearchParams`** 接口定义了一些实用的方法来处理 URL 的查询字符串,一个实现了 `URLSearchParams` 的对象可以直接用在 [`for...of`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/for...of) 结构中。

```typescript
var paramsString = "q=URLUtils.searchParams&topic=api"
var searchParams = new URLSearchParams(paramsString);

for (let p of searchParams) {
  console.log(p);
}

searchParams.has("topic") === true; // true
searchParams.get("topic") === "api"; // true
searchParams.getAll("topic"); // ["api"]
searchParams.get("foo") === null; // true
searchParams.append("topic", "webdev");
searchParams.toString(); // "q=URLUtils.searchParams&topic=api&topic=webdev"
searchParams.set("topic", "More webdev");
searchParams.toString(); // "q=URLUtils.searchParams&topic=More+webdev"
searchParams.delete("topic");
searchParams.toString(); // "q=URLUtils.searchParams"
```

`URLSearchParams` 构造函数*不会*解析完整 URL，但是如果字符串起始位置有 `?` 的话会被去除。详情可参考[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/URLSearchParams)

## `axios`传输数据序列化

- 使用`qs.stringify`

1. 安装

```shell
npm i qs --save
```

2. `axios`配置

```typescript
import axios from 'axios'
import qs from 'qs'

// 添加响应拦截器
axios.interceptors.request.use(
	config => {
		if (config.method === 'post') {
			config.data = qs.stringify(config.data)
		}
		return config
	},
	error => {
		console.log(error)
		Promise.reject(error)
	}
)
```

这里我们看一下，其实就是在 `axios` 发起请求的前拦截请求，当请求类型是 `post` 的时候，把请求的入参数据 `qs.stringify` 一下，看一下前后数据格式的对比。

```typescript
// qs.stringify 前
{
    "userId": "520b0ec329164dd9a4f216ba8d209029",
    "startTime": "1548950400000",
    "endTime": "1551369599999"
}
// qs.stringify 后
"userId=520b0ec329164dd9a4f216ba8d209029&startTime=1548950400000&endTime=1551369599999"
```

**注意：前端使用字符串格式化参数，需要后端配合，这样才能拿到正确的参数。**

说到`stringify`，这里就不得不谈谈`JSON.stringify`了，那么他们之间的区别是什么呢？

```typescript
var obj = {
  name: 'jack',
  age: 20
};
console.log(qs.stringify(obj));
// 'name=jack&age=20'
console.log(JSON.stringify(obj));
// '{ "name":"jack", "age":20 }'
```

所以`qs.stringify()`将对象 序列化成URL的形式，以&进行拼接，`JSON`是正常类型的JSON数据。
