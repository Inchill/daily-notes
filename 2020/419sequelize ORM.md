## 1.使用sequelize ORM

使用model映射数据库表时，应指明表的名称而且字段必须与表中一致，还要指明主键。同时如果没有created_at、updated_at的话，需要在模型中配置time stamps为false。一般来说实际项目中肯定要创建时间戳和更新时间戳，因为需要准确记录时间。

```js
const UserModel = sequelize.define('user', {
    user_id: {
        type: Sequelize.INTEGER,
        allowNull: false,
        primaryKey: true,
        autoIncrement: true,
        field: 'user_id'
    },
    username: {
        type: Sequelize.STRING,
        allowNull: false,
        field: 'username'
    },
    password: {
        type: Sequelize.STRING,
        allowNull: false,
        field: 'password'
    },
    email: {
        type: Sequelize.STRING,
        allowNull: false,
        field: 'email'
    },
    phone_num: {
        type: Sequelize.STRING,
        allowNull: false,
        field: 'phone_num'
    }
}, {
    timestamps: false,
    tableName: 'user_table'
});
```

## 2.使用nodemon热重启

安装nodemon

```js
npm i --save-dev nodemon
```

修改pakeage.json

