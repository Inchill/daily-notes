## axios获取本地json报404

不建议通过axios获取，用import引入：

```js
import mapJson from '../../../../static/json/map.json';
```

## vue子组件传值给父组件

```js
<address-input @sendMsg="getAddress"></address-input>
```

父组件用getAddress，子组件触发sendMsg

## p标签文字溢出

```css
p {
    word-wrap break-word;
}
```

## flex布局子元素居右

```css
flex: 5;
text-align: right;
```

## vue路由跳转通过params传参，刷新页面参数丢失

百度之后有人说改为query，改为query后发现参数变为object。我想到存储的是待结算商品列表，因此不能通过路由传参来解决，就想到了vuex。因此我新建了一个store，跳转到结算页面前将其存储在store里，但是刷新还是会丢失数据。此时我百度了一下，找到了vuex也失效的原因：store里的数据是保存在运行内存中的，当页面刷新时，页面会重新加载vue实例，store里面的数据就会被重新赋值初始化。

解决思路是数据持久化，可以将数据保存在localStorage、sessionStorage和cookie中。

1. localStorage的生命周期是永久的，页面关闭或浏览器关闭也不会失效，除非主动删除。
2. sessionStorage的生命周期仅在当前会话下有效。sessionStorage引入了一个“浏览器窗口”的概念，sessionStorage是在同源的窗口中始终存在的数据。只要浏览器窗口没有关闭，即使刷新页面或者进入同源的另一个页面，数据依然存在，但是关闭窗口后sessionStorage就会被销毁。同时独立地打开同一个窗口同一个页面，sessionStorage也是不一样的。
3. cookie在设置的过期时间之前一直有效，即使窗口关闭或浏览器关闭。存储数据大小为4K左右，而且浏览器一般有个数限制，不能超过20个。缺点是不能存储大数据且不易读取。

由于vue是单页应用，操作都是在一个页面跳转路由，因此sessionStorage较为合适。理由如下：

1. sessionStorage可以保证打开页面时数据为空。
2. 每次打开页面时localStorage存储着上一次页面打开时的数据，因此需要手动清空。

因此结合store和sessionStorage，每次生成待结算商品数组前，先清空store存储的上一次用户提交的待结算商品数组，保证每次统计当前待结算商品。为了防止刷新待结算页面数据丢失，再用sessionStorage做一下数据持久化。

