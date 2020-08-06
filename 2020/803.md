## Mac电脑打出英文名间隔圆点

```shell
shift + option + 9
```

## MacOS安装homebrew报错

MacOS安装homebrew报错：curl: (7) Failed to connect to raw.githubusercontent.com port 443: Connection refused.

1. 打开网站: `https://www.ipaddress.com/`, 查询一下 `raw.githubusercontent.com`对应的IP 地址.
2. 使用 sudo vim /etc/hosts 命令编辑hosts文件，添加以下内容并保存.

```txt
199.232.68.133  raw.githubusercontent.com
```

再次输入官网上面安装brew那一条命令居然发现能够下载了。不过下载速度有点慢，又提示超时：

curl: (7) Failed to connect to raw.githubusercontent.com port 443: Operation timed out

搜了一圈发现始终会报这个错，因此在掘金上面搜到了这个文章，用这个来下载[传送门](https://juejin.im/post/6844903549051076622).

