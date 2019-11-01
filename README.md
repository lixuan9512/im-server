# im-server
IM application, including server and front-end.

## 介绍
总结下在IM这块的落地实践，使用`socket.io`作为IM的核心部分，不掺杂公司业务的IM通信示例，包含前后端，前端部分`Vue`和`React`都有实现。

## 功能简介
1. 角色、权限管理
2. 群组管理
3. 用户注册，登录，个人信息编辑
4. 个人、群组聊天
5. 聊天名单记录
6. 表情，图片，定位
7. 机器人自动回复
8. 在线离线显示
9. 支持多点登录
10. 支持消息漫游

## 框架简介

### server
选用阿里的egg框架做支撑，看中的原因是他们内部大规模的落地和安全方面做得比较好。

### front-end
我本身对`Vue`是比较熟悉的了，最近想使用下`React`感受下不同的设计思路，所以准备先使用`Vue`实现出所有功能，然后再使用`React`实现一次。

## server端详细说明

### passport
这个章节的官方文档，要你的命，一定要去看源码，太坑人了，我研究了一整天才弄明白是怎么回事。因为我想更自由的控制账户密码登录，所以账号密码登录并没有使用passport，使用的就是普用的controller控制的。

下面详细说下使用第三方平台（我选用的是GitHub）登录的过程：
1. 在[GitHub OAuth Apps](https://github.com/settings/applications/new)新建你的应用
1. 在项目安装egg-passport和egg-passport-github
2. 开启插件：
```
// config/plugin.js
module.exports.passport = {
  enable: true,
  package: 'egg-passport',
};

module.exports.passportGithub = {
  enable: true,
  package: 'egg-passport-github',
};
```
3. 配置：
```
// config/default.js
config.passportGithub = {
  key: 'your_clientID',
  secret: 'your_clientSecret',
  callbackURL: '/v1/passport/github/callback' // 注意这里非常的关键，这里需要和你在github上面设置的Authorization callback URL一致，我开发的时候想换github上面设置的Authorization callback URL一直报错，后来发现需要在这里配置
};
```
4. 需要设置两个passport的get请求路由，第一个是我们在login页面点击的请求，第二个是我们在上一步设置的callbackURL，这里主要是第三方平台会给我们一个可用的code，然后根据OAuth2授权规则去获取用户的详细信息
```
const github = app.passport.authenticate('github', { successRedirect: '/v1/passport' });// successRedirect就是最后校验完毕后前端会跳转的路由
router.get('/v1/passport/github', github);
router.get('/v1/passport/github/callback', github);
```
5. 获取到详细信息后，我们需要在app.js里面的app.passport.verify去验证用户信息，并且和我们自身平台的用户信息做关联，也要授权给session
```
// 在verify函数我们只能获取ctx和user
module.exports = async (ctx, user) => {
  const { provider, id, name, photo } = user;
  if (provider === 'github') {
    // 查询auth是否已经注册
    const auth = await ctx.model.Authorization.findOne({
      where: {
        provider,
        uid: id
      }
    });
    if (!auth) {
      // 如果没有注册的话就需要使用拿到的user信息注册下
      const newAuth = await ctx.model.Authorization.create({
        provider,
        uid: id,
        username: name
      });
      const user = await ctx.model.User.create({
        nickname: name,
        photo
      });
      newAuth.setUser(user);
      // 这步和下面的功能一样，添加auth到session
      ctx.session.auth = newAuth;
    } else {
      ctx.session.auth = auth;
    }
  }
  return user;
};
```

### 消息
message的结构设计参考了几家第三方服务的设计，也结合本项目自身的情况做如下说明：
```
const Message = app.model.define('message', {
  /**
    * 消息类型：
    * 0:单聊
    * 1:群聊
    */
  type: {
    type: STRING
  },
  // 消息体
  body: {
    type: JSON
  },
  fromId: { type: INTEGER },
  toId: { type: INTEGER }
});
```
body里面存放的是消息体，使用json用来存放不同的消息格式：

```
// 文本消息
{
  "type": "txt",
  "msg":"哈哈哈" //消息内容
}
```

```
// 图片消息
{
  "type": "img",
  "url": "http://nimtest.nos.netease.com/cbc500e8-e19c-4b0f-834b-c32d4dc1075e",
  "ext":"jpg",
  "w":360,    //宽
  "h":480,    //高
  "size": 388245
}
```

```
// 视频消息
{
  "type": 'video',
  "url": "http://nimtest.nos.netease.com/cbc500e8-e19c-4b0f-834b-c32d4dc1075e",
  "ext":"mp4",
  "w":360,    //宽
  "h":480,    //高
  "size": 388245
}
```

```
// 地理位置消息
{
  "type": "loc",
  "title":"中国 浙江省 杭州市 网商路 599号",    //地理位置title
  "lng":120.1908686708565,        // 经度
  "lat":30.18704515647036            // 纬度
}
```


## 部署
docker
