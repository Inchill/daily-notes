### 1.找不到babel-core

```js
ERROR in ./node_modules/ali-oss/dist/aliyun-oss-sdk.js     
Module build failed (from ./node_modules/babel-loader/lib/index.js):
Error: Cannot find module 'babel-core'
Require stack:
- D:\WebProjects\trading\node_modules\babel-loader\lib\index.js
- D:\WebProjects\trading\node_modules\loader-runner\lib\loadLoader.js
```

我的package.json是这样的："@babel/core": "^7.9.6"版本，babel-loader版本是7.1.5。然后我点进了babel-loader\lib\index.js查看，发现引入的是babel-core，并不是@babel/core，于是我修改了该文件：

```js
"use strict";

var babel = require("@babel/core");
var loaderUtils = require("loader-utils");
var path = require("path");
var cache = require("./fs-cache.js");
var exists = require("./utils/exists");
var relative = require("./utils/relative");
var read = require("./utils/read");
var resolveRc = require("./resolve-rc.js");
var pkg = require("../package.json");
var fs = require("fs");

```

重新执行命令npm run dev，就成功了。

**原因：这是因为babel-loader和babel-core版本不兼容，babel-loader里面引入的是babel-core，属于7以下版本。而babel-core版本7以后都是@babel/core形式，因此babel-loader就找不到babel-core。**

当然，上述直接修改babel-loader里面index.js文件是有很大隐患的。无论怎么修改，package.json版本始终不匹配，一旦npm install就会回到原版本。我的做法是是升级babel-loader:

```js
npm i --save-dev babel-loader@8
```

由于@babel/core还是7.9.5版本，低于babel-loader版本，因此这样项目就不会报错了。
