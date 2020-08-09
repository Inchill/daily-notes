## node如何进行MVC分层开发

- 分离router

  ```tsx
  const router = require('koa-router')();
  
  module.exports = (app) => {
      router.get('/', async(ctx, next) => {
          await next();
          ctx.response.type = 'text/html';
          ctx.response.body = '<h1>hello world, i am chuck</h1>';
      });
      app.use(router.routes()).use(router.allowedMethods());
  };
  ```

- 分离controller

  将router中路由响应函数抽离出来，这一层属于业务逻辑层。

  ```tsx
  module.exports = {
      index: async (ctx, next) => {
          await next();
          ctx.response.type = 'text/html';
          ctx.response.body = '<h1>hello world, i am chuck</h1>';
      }
  }
  ```

- 分离service

  这一层用于操作数据，如操作数据库、调用第三方接口获取数据等，属于数据访问层。

  ```tsx
  module.exports = {
      login: async (username, password) => {
          let data;
          // 实际项目中这里要连接数据库并获取账号信息进行比对
          if(username === 'chuck' && password === '123456') {
              data = `hello, ${ username }`;
          }else {
              data = '账号信息错误';
          }
          return data;
      }
  }
  ```

  

------

### webpack打包出错

无法重新声明块范围变量“router”。ts(2451)

app.ts(3, 7): 'router' was also declared here.

*解决方式：配置tsconfig.json*
