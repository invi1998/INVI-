<!-- 安装插件
 Markdown All in One
 Markdown Preview Enhanced
 Markdown Shortcuts
 markdown pdf
 markdown toc
简介：这个插件是用来生成目录
配置：
markdown-toc.depthFrom: 生成目录的标题最低级别，默认h1
markdown-toc.depthTo: 生成目录的标题最高级别，默认h6
markdown-toc.insertAnchor: 自动插入链接地址，默认false
markdown-toc.withLinks: 自动插入链接，默认true
markdown-toc.updateOnSave: 自动更新

常用快捷键
column0	column1
Key	Command
**Ctrl + B	粗体**
_Ctrl + I	斜体_
~~Alt + S	删除线~~
# Ctrl + Shift + ]	标题(uplevel)
Ctrl + Shift + [	标题(downlevel)
Ctrl + M	Toggle math environment
Alt + C	Check/Uncheck task list item 
ctrl+k v: 这个意思是先同时按住ctrl和k键，放开后再按一个v键

2. 使用技巧总结(MarkDown语法)
2.1. 标题设置（:后面需要空格）
几级标题需要使用多少个‘#’。

2.2. 段落or换行
使用空行隔开自动形成段落
在结束处使用2个空格键即可
2.3. 简单字体设置
粗体：安装完插件后，使用ctrl+b设置选中的文字，或者使用代码在** **，在之间的空格处插入文字。
斜体：使用ctrl+i，可以设置斜体，在两个‘*’之间设置。
下划线：左右个2个++即可

2.4. 引用
使用“> “，即可。

2.5. 分割线
使用三个"***"，或者

2.6. 页眉分割线
使用三个"-"就可以设置页眉分割线

2.7. 插入图片
图片的插入比较负责可以使用"![]（）“这个是比较负责的在之间插入本地图片的地址
-->

# 开发笔记

---

## 用户登录

* ### 登录流程分析

    **_前端_**：用户登录界面->输入用户名和密码->点击登录————>**后端**：用户授权模块->mysql数据库中校验用户名和密码是否正确->正确之后->通过jwt生成token->将token返回给前端->前端保存token->之后请求用户信息的时候->会将token附带在http header中->传给后端->后端拿到数据->进行token校验->校验成功->会解析token获得用户名并查询用户信息->然后将用户信息返回给前端->前端拿到信息->会去做一个路由的认证->重定向到/根路径->根据用户权限生成菜单
    ![登录流程](image/登录流程分析.png)

* ### 路由和权限校验
  
  首先，我们访问一个路由，假定我们访问的路由是/login，可以看见在这个项目中（vue-element-admin)框架定义了一个全局的路由守卫，所有经过这个路由的都需要访问这个方法。
  **_那这个方法具体做了哪些事情？_**
  首先它从cookie中获取token，判断token是否存在，其实我们第一次访问login的时候，token是不存在的，然后没有token会进入到否这个环节
  <h5 style="color:red">token不存在</h5>
  在这个环节，它首先会判断路由是否在白名单中，如果在白名单中，他就可以直接访问，我们的login就是在白名单中的。<img src="./image/白名单.png">访问到路由的时候，在界面上就会做一个事情，把app.vue里的\<_**router-view**_/>替换为我们的login组件。如果我们访问的路由不在白名单中，那么我们实际访问的路由会是/login?redirect=/xxx，即在登录之后会被重定向到这个路由中。以上即为token不存在的情况.

  <h5 style="color:green">token存在</h5>

  这时候有token（已登录过）还访问login，会走到左边token存在这个分支。token存在的时候，他会做一个校验，路由是否为/login？如果是，那么它会重定向到根路径下（/），而根路径对应的就是项目里的deshboard。如果不是/login那么它就会根据token获取角色信息，生成动态路由，去做权限校验和匹配。如果匹配到这个路由，它会通过replace模式来访问这个路由。如果在右边这些过程中任何一步发生异常，就会进入异常处理逻辑（使用try catch）。异常处理逻辑主要有两步，第一：重置token；第二：访问/login，其实就是回到登录页面。
![路由处理逻辑](image/路由处理逻辑.png)

* ##### 路由实现
  
  下面这个是一个纯样式，表示在页面加载时在页面顶部显示的加载页面进度条，因为是纯样式，所以需要手动调用start和done方法进行控制
  ```js
    // 开启进度条
    NProgress.start()
    // 关闭进度条
    NProgress.done()
  ```
  首先，可以看见在main.js里面，引入了permission这个文件
  ```js
    import './permission' // permission control
  ```
  打开这个文件，里面设置了全局路由守卫，注意最后路由访问的时候一定要去调用这个next()回调，不然这个路由是访问不到的。
  permission.js
  ```js
    router.beforeEach(async(to, from, next) => {
    // 开启进度条
        NProgress.start()

        // 根据配置项更改标题（meta里的title属性）
        document.title = getPageTitle(to.meta.title)

        //去cook中获取token
        const hasToken = getToken()

        if (hasToken) {
            if (to.path === '/login') {
                // 如果已登录，请重定向到主页
                next({ path: '/' })
                NProgress.done()
            } else {
                // 确定用户是否已通过getInfo获得其权限角色
                const hasRoles = store.getters.roles && store.getters.roles.length > 0
                if (hasRoles) {
                next()
            } else {
                try {
                    // 获取用户信息
                    //注意：角色必须是对象数组！ 例如：['admin']或，['developer'，'editor']
                    const { roles } = await store.dispatch('user/getInfo')

                    // 根据角色生成可访问的路由表
                    const accessRoutes = await store.dispatch('permission/generateRoutes', roles)

                    // 动态添加可访问的路由
                    // 根据路由表生成菜单栏
                    router.addRoutes(accessRoutes)

                    // hack方法，以确保addRoutes是完整的
                    // 设置replace：true，因此导航将不会留下历史记录，具体表现为从登陆页面进入到主页后，回退是不会退回到登录页面的
                    next({ ...to, replace: true })
                } catch (error) {
                    // 删除token并进入登录页面重新登录
                    await store.dispatch('user/resetToken')
                    // 这里这个resetToken就是清除token的接口
                    Message.error(error || 'Has Error')
                    next(`/login?redirect=${to.path}`)
                    NProgress.done()
                }
            }
        } else {
            /* 没有token*/
            if (whiteList.indexOf(to.path) !== -1) {
                // 看白名单中是否有这个路由，有就直接找重定向访问这个路由
                next()
            } else {
                // 其他无权访问的页面将被重定向到登录页面
                next(`/login?redirect=${to.path}`)
                NProgress.done()
            }
        }
    })
  ```
* ##### 动态路由实现流程
  动态路由生成逻辑如下图
  ![动态路由实现流程](image/动态路由流程图.png)
  ###### 逻辑分析
    首先需要将asyncRoutes和constantRoute读取到，然后获取用户的角色，比如你是admin或者其他什么的。然后做一个判断，你的roles（角色）是否是admin？如果是admin就会直接将asyncRoutes保存到vuex中。并且将2个路由进行合并。这说明如果你是admin权限，将会跳过所有的路由的判断，将所有的菜单进行加载。如果不是admin权限，它将会去遍历所有的routes，取出每个routers进行判断。判断当前这个权限是否具有路由访问的权限。如果没有会遍历下一个，如果有，会进入下一步判断，判断路由当中是否存在子路由。如果存在子路由，会采用迭代的方法进行子路由的一个遍历，子路由当中会进行过滤，过滤完后会对当前这个children进行更新，其实就是将不符合条件的子路由剔除掉。最后当判断完成之后，把路由存入到一个新的数组当中，把这个新的数组替换原来这个asyncRoutes，然后存入vuex。最后将asyncRoutes和constantRoutes进行合并。

    ###### 源码实现
    生成路由是在这里的generateRoutes这个方法里面，它位于store/modules/permission.js里。这里这个方法其实就是上面提到的2个步骤，第一就是判断当中是否包含admin，只要包含admin，比如说你可能有editor权限，admin权限都有，但是只要你包含了admin，那么asyncRoutes就会直接赋给这里的accessedRoutes，accessedRoutes拿到之后会将其调用SET_ROUTES（即给vuex里的routes），保存我们的state当中的addRoutes。然后会把我们state下面的routes和constantRoutes调用concat方法和新的routes进行合并，合并完成之后放到state里的routes下。所以在主页生成sliderbar（左侧侧边栏）的时候，是根据这里的routes进行的，而这个addRoutes是作为一个保存用的。以上是admin权限的一个逻辑。
    如果没有admin权限，将会走到filterAsyncRoutes这个方法里去，他会对routes进行一个遍历，使用const tmp = { ...route }对routes进行一个浅拷贝，拷贝到新的一个tmp里面。然后去做hasPermission的一个校验（校验是否具有权限）。hasPermission源码主要是做2个判断，第一个判断是去找传入的这个路由是否具有meta这个信息，如果没写meta或者meta下面没有roles这个属性，将直接返回true，返回true意味着有权限。这就说明，如果你没有定义权限，那么这个校验就视为你对这个路由有访问权限，因为这里没有配置那些人可以访问，那就默认谁都可以访问。如果说我们定义了这个meta，meta里面有roles，那么他就和我们router目录下的路由情况命中
    ```js
        {
            path: '/book',
            component: Layout,
            redirect: '/book/create',
            meta: { title: '图书管理', icon: 'doucumentation', roles: ['admin'] },
            children: [
            {
                path: '/book/create',
                component: () => import('@/views/book/create'),
                meta: { title: '上传图书', icon: 'edit', roles: ['admin'] }
            }
            ]
        },
    ```
    然后他就会做一个判断，判断hasPermission方法里的routes，其实就是route里的这句话meta: { title: '上传图书', icon: 'edit', roles: ['admin']里的这个数组，进行some判断，some就是命中数组里的一条，即为true，会把下面的routes进行权限检查，回去找我们写好在路由里的数组权限，看这个数组是否包含这里的roles。也就是说只要你这些roles和在路由里定义好的权限数组有一个匹配，就会返回一个true，即认为有权限。
    当判断hasPermission通过的时候，才能进行下一步的操作，他会去找是否有children，就是当前这个路由是否有子路由，因为它需要判断你是否有访问下一级路由的权限，所以他会被子菜单迭代调用自身（filterAsyncRoutes），它会把子路由依次传入，再进行判断，判断的结果会来更新这里的tmp.children，更新完之后把整个tmp赋给res，最后把res返回，返回之后替换掉generateRoutes里的accessedRoutes，赋到‘SET_ROUTES’再将其和constantRoutes结合在一起。如果没有通过，即不能访问的权限，那后面的操作也就不执行了，直接进入下一个迭代。
    ```js

        function hasPermission(roles, route) {
            if (route.meta && route.meta.roles) {
                return roles.some(role => route.meta.roles.includes(role))
            } else {
                return true
            }
        }

        export function filterAsyncRoutes(routes, roles) {
            const res = []

            routes.forEach(route => {
                const tmp = { ...route }
                if (hasPermission(roles, tmp)) {
                if (tmp.children) {
                    tmp.children = filterAsyncRoutes(tmp.children, roles)
                }
                res.push(tmp)
                }
            })

            return res
        }

        const state = {
            routes: [],
            addRoutes: []
            }

            const mutations = {
            SET_ROUTES: (state, routes) => {
                state.addRoutes = routes
                state.routes = constantRoutes.concat(routes)
            }
        }

        const actions = {
            generateRoutes({ commit }, roles) {
                return new Promise(resolve => {
                let accessedRoutes
                if (roles.includes('admin')) {
                    accessedRoutes = asyncRoutes || []
                } else {
                    accessedRoutes = filterAsyncRoutes(asyncRoutes, roles)
                }
                commit('SET_ROUTES', accessedRoutes)
                resolve(accessedRoutes)
                })
            }
        }
    ```

***

* ### 侧边栏

* ### 重定向

* ### 面包屑导航
