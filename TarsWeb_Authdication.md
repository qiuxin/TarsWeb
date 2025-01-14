## TARS 用户体系模块使用指引

TARS管理平台提供了与用户体系（包括单点登录系统和权限系统）对接的能力，若用户本身无相应的系统，TARS可提供了一个简易的用户体系模块，供用户选装。用户体系模块提供了单点登录和注册的功能，以及到服务层级的权限控制能力。用户也可以选择只使用其中一个功能。

### 1、用户体系模块安装

若用户想使用默认的用户体系模块，则需先安装web中的用户体系管理平台模块。相应的数据库名：db_user_system

在`TarsWeb/demo`下，修改`config/webConf.js`中的数据库配置为自己的数据库的配置（可以与TarsWeb一致），然后在`TarsWeb/demo`下运行命令
```sh
npm install
npm run prd
```
即可完成默认用户体系模块的安装并启动。

其中：

1. 默认登录模块：提供基础的注册和登录页面，对外提供获取用户信息的接口`getUidByTicket`和校验是否登录的接口`validate`,
    - `getUidByTicket`接口接收一个`ticket`参数，返回`uid`
    - `validate`接口接收`ticket`参数和`uid`参数，返回result (`true`或`false`)

2. 默认权限模块：权限模块数据库表中有三个字段，为`flag`, `role`和`uid`，分别对应标志（在TARS平台中为“应用+服务”表示一个标志），角色和用户。

权限模块对外提供6个接口和一个页面：
```
    /auth/addAuth：          批量新增权限接口，入参为[{flag: “”，role: “”，uid: “”}],
    /auth/deleteAuth：       删除权限接口，入参为flag，删除flag下所有权限信息
    /auth/updateAuth：       更新权限接口，入参为flag，role，uid，其中，uid为用户列表，表示更新某个flag和role下的所有用户信息。
    /auth/getAuthListByUid： 获取某用户具有的全部权限列表，入参为uid
    /auth/getAuth：          判断用户是否具有权限，入参为flag，role，uid。
    /auth/getAuthListByFlag：获取有某个flag权限的用户信息，入参为flag
    
    权限模块还提供了一个管理页面：
    /auth.html：             用于对权限进行增删改查。
```

**注意：** 默认的权限模块，为保证系统安全性，以上的6个接口，必须采用白名单的方式访问，不允许被其他人随意调用，管理页面`auth.html`需系统管理员才可使用。白名单和管理员的相关配置，可在`TarsWeb/demo/config/authConf.js`中配置。

### 2、TARS对接登录模块能力

默认用户体系模块已经安装好了，接下来我们要将TarsWeb对接到该模块的登录模块。

TARS 通过配置文件`TarsWeb/config/loginConf.js`与第三方登录体系或默认用户体系登录模块关联，提供允许用户登录的能力。

需要修改的登录配置文件参数
* `enableLogin`: 改为`true`
* `loginUrl`: `localhost`修改为用户模块部署机器的域名, 如`http://www.tars.com:3001/login.html`
* `cookieDomain`: 改为TarsWeb部署所在域名，如`www.tarsweb.com`

如果TarsWeb和用户体系模块部署在同一机器上，则`loginUrl`和`cookieDomain`域名一致；若不在同一机器上，除了这两个参数域名不一致外，还需要修改`getUidByTicket`和`validate`函数中的域名为登录模块域名。

修改完之后，重启TarsWeb即可进行用户注册和登录的操作
```sh
pm2 stop bin/www tars-node-web
npm run prd
```

其中，登录配置文件详细信息如下：
```js
module.exports = {
    enableLogin: false,                     //是否启用登录验证
    defaultLoginUid: 'admin',               //若不启用登录验证，默认用户为admin
    loginUrl: 'http://localhost:3001/login.html', //登录跳转url
    redirectUrlParamName: 'redirect_url',   //跳转到登录url的时带的原url参数名，如：***/login?redirect_url=***，默认是redirect_url
    logoutUrl: '',
    logoutredirectUrlParamName: 'url',
    ticketCookieName: 'ticket',             //cookie中保存ticket信息的cookie名
    uidCookieName: 'uid',                   //cookie中保存用户信息的cookie名
    cookieDomain: 'localhost',              //cookie值对应的域
    ticketParamName: 'ticket',              //第三方登录服务回调时候，url中表示st的参数名
    getUidByTicket: getUidByTicket,         //通过ticket从cas服务端校验和获取用户基本信息的url,或获取用户基本信息的方法
    getUidByTicketParamName: 'ticket',      //调用获取用户信息接口时候st的参数名
    uidKey: 'data.uid',                     //结果JSON里面取出用户名的位置，取到该用户名才认为成功,可以多层
    validate: validate,                     //通过token和用户名到cas服务端校验key和用户名是否匹配的url或方法
    validateTicketParamName: 'ticket',      //校验接口传入st参数名
    validateUidParamName: 'uid',            //校验接口传入用户参数名
    validateMatch: [
        ['data.result', true]
    ],                                      //校验通过匹配条件，可以从多层结果，多个情况
    ignore: ['/static'],                    //不需要登录校验的路径
    ignoreIps: [],                          //访问ip白名单
    apiPrefix: ['/pages/server/api'],       //接口相应的路径前缀，这类接口访问不直接跳转到登录界面，而只是提示未登录
    apiNotLoginMes: '#common.noLogin#',     //接口无登录权限的提示语
};

/**
 * 由用户直接定制通过ticket获取用户信息的方法
 * @param ctx
 */
async function getUidByTicket(ctx, ticket) {
    return new Promise((resolve, reject) => {
        try {
            request.get('http://localhost:3001/api/getUidByTicket?ticket=' + ticket).then(uidInfo => {
                uidInfo = JSON.parse(uidInfo);
                resolve(uidInfo.data.uid);
            }).catch(err => {
                reject(err);
            })
        } catch (e) {
            resolve(false)
        }
    })
}

/**
 * 由用户直接定制判断用户名校验方法
 * @param ctx
 */
async function validate(ctx, uid, ticket) {
    return new Promise((resolve, reject) => {
        try {
            request.get('http://localhost:3001/api/getUidByTicket?ticket=' + ticket).then(uidInfo => {
                uidInfo = JSON.parse(uidInfo);
                resolve(uidInfo.data.uid === uid);
            }).catch(err => {
                reject(err);
            })
        } catch (e) {
            reject(false)
        }
    })
}
```

### 3、TARS对接权限模块能力
完成登录模块的对接，有登录和注册的功能，但是任何用户都可以注册并看到所有的服务，因此嗨哟啊对接权限模块限制用户权限。

TARS 通过配置文件`TarsWeb/config/authConf.js` 与第三方权限系统，或默认用户体系权限模块关联，提供权限控制的能力。

需要修改的参数
* `enableAuth`: 改为`true`
* 接口URL域名和端口: 改为用户体系模块机器域名和端口，如 `http://localhost/api/auth/addAuth`改为`http://www.tars.com:3001/api/auth/addAuth`

修改完即完成权限模块能力对接，重启TarsWeb即可。

另外，如果需要对用户权限进行操作，需要在`TarsWeb/demo/config/authConf.js`中添加相应的用户或IP，重启用户体系模块后，登录对应白名单内用户，将网页URL修改为auth页面地址，如`http://www.tars.com:3001/auth.html`，即可在页面上对用户权限进行修改；

或是直接在`db_user_system.t_auth`中插入管理员权限，如在页面注册账号`tarsadmin`后，在数据库中插入如下数据即可完成管理员权限的添加
```sql
INSERT INTO `db_user_system`.`t_auth` (`flag`, `role`, `uid`) VALUES ('', 'admin', 'tarsadmin')
```

其中，权限配置文件详细信息如下（接口入参和出参与上述第1点用户体系权限模块一致）：
```js
{
    /**
     * 是否启用自定义权限模块
     */
    enableAuth: false,

    /**
     * addAuthUrl             新增权限url
     * TARS平台会提供的参数
     * @param   {Array}    auth         权限对象列表，格式如 {"flag": "app-server", "role": "operator", "uid": "username"}
     */
    /**
     * 接口需要返回的参数
     * @param   {Number}    ret_code            返回码，200表示成功
     * @param   {String}    err_msg             错误信息
     */
    addAuthUrl: 'http://localhost/api/auth/addAuth',

    /**
     * deleteAuthUrl             删除权限url，用于服务下线时候删除权限
     * TARS平台会提供的参数
     * @param   {String}    flag                权限单位，在tars中为“应用-服务”
     */
    /**
     * 接口需要返回的参数
     * @param   {Number}    ret_code            返回码，200表示成功
     * @param   {String}    err_msg             错误信息
     */
    deleteAuthUrl: 'http://localhost/api/auth/deleteAuth',

    /**
     * updateAuthUrl             更新权限url
     * TARS平台会提供的参数
     * @param   {String}    flag                权限单位，在tars中为“应用-服务”
     * @param   {String}    role                角色，在tars中为operator或developer
     * @param   {String}    uid                 用户名
     */
    /**
     * 接口需要返回的参数
     * @param   {Number}    ret_code            返回码，200表示成功
     * @param   {String}    err_msg             错误信息
     */
    updateAuthUrl: 'http://localhost/api/auth/updateAuth',

    /**
     * getAuthListByUidUrl             通过用户名获取权限列表url
     * TARS平台会提供的参数
     * @param   {String}    uid                 用户名
     */
    /**
     * 接口需要返回的参数
     * @param   {Array}     data                服务列表，内容如下
     * @param   {String}    flag                权限单位，在tars中为“应用-服务”
     * @param   {String}    role                角色，在tars中为operator或developer
     * @param   {String}    uid                 用户名
     * @param   {Number}    ret_code            返回码，200表示成功
     * @param   {String}    err_msg             错误信息
     */
    getAuthListByUidUrl: 'http://localhost/api/auth/getAuthListByUid',

    /**
     * getAuthListByFlagUrl             通过应用名+服务名获取用户列表url
     * TARS平台会提供的参数
     * @param   {String}    flag                应用+服务名
     */
    /**
     * 接口需要返回的参数
     * @param   {Array}     data                服务列表，内容如下
     * @param   {String}    flag                权限单位，在tars中为“应用-服务”
     * @param   {String}    role                角色，在tars中为operator或developer
     * @param   {String}    uid                 用户名
     * @param   {Number}    ret_code            返回码，200表示成功
     * @param   {String}    err_msg             错误信息
     */
    getAuthListByFlagUrl: 'http://localhost/api/auth/getAuthListByFlag',

    /**
     * getAuthUrl             判断用户是否有相应角色的操作权限
     * TARS平台会提供的参数
     * @param   {String}    flag                权限单位，在tars中为“应用-服务”
     * @param   {String}    role                角色，在tars中为operator或developer
     * @param   {String}    uid                 用户名
     */
    /**
     * 接口需要返回的参数
     * @param   {Object}    data                服务列表，内容如下
     * @param   {Boolean}   result              是否有操作权限
     * @param   {Number}    ret_code            返回码，200表示成功
     * @param   {String}    err_msg             错误信息
     */
    getAuthUrl: 'http://localhost/api/auth/getAuth'
}
```