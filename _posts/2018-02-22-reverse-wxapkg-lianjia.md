---
layout: post
title: 解析链家小程序 - 获取请求校验方式
key: 20180222
tags: 小程序逆向 请求校验 python爬虫
---

# 解析链家小程序 - 请求验证方式

​        数据需求，对链家小程序进行请求抓包，发现每次合法请求都存在 Authorization 授权码，想要成功模拟小程序发送请求，就必须破解Authorization 授权码的生成方式。

### 目录

- 1、获取链家微信小程序的 .wxapkg 包文件、解开 .wxapkg 程序包
- 2、了解.wxapkg文件结构
- 3、查看程序逻辑，实验生成Authorization
- 4、验证迭代、实现效验方式



### 1、获取链家微信小程序的 .wxapkg 包文件、解开 .wxapkg 程序包

原理：基于 android-sdk\platform-tools\adb工具，通过 root 权限获取小程序安装包 .wxapkg ，通过github 上的[wechat-app-unpack](https://github.com/leo9960/wechat-app-unpack) 项目，逆向解压获取小程序源码。

#### 获取小程序安装包

1. 手机开启root权限，如何开启依据机型系统而定，这里使用网易MuMu模拟器, 默认开启root权限；

2. 手机开启USB调试功能, 安装微信，打开链家小程序；

3. 电脑安装 android-sdk；

4. 获取小程序安装包：

   ```
   > adb shell
   # cd /data/data/com.tencent.mm/MicroMsg/70aa34178251376743797472a68c1c6a/appbrand/pkg
   # ls
   _-1261323258_6.wxapkg
   _1079392110_3.wxapkg
   _1123949441_106.wxapkg
   # cp _-1261323258_6.wxapkg /sdcard/
   # exit
   $ exit
   > adb pull /sdcard/_-1261323258_6.wxapkg .
   ```

注意：

- 目录 /data/data/com.tencent.mm/MicroMsg/70aa34178251376743797472a68c1c6a/appbrand/pkg 中， 70aa34178251376743797472a68c1c6a 文件夹可能因不同的机器有所改变，在/data/data/com.tencent.mm/MicroMsg/ 目录下自行寻找一下，
- _-1261323258_6.wxapkg 在我的机器上为 链家小程序的wxapkg文件，请大家自行寻找自己机器上的名字

由此，即可获取链家小程序安装包；

#### 解开 .wxapkg 程序

这里 使用 python 版的程序解开.wxapkg 包, 将 -1261323258_6.wxapkg放到 unpack-wxapkg.py 同级目录下。

```Shll
# python3 unpack-wxapkg.py _-1261323258_6.wxapkg
```





### 2、了解.wxapkg文件结构

![](https://raw.githubusercontent.com/BuleAnt/RepositoryResources/master/image/RecommendationMovie/pmkz_2018-02-12-15.01.01.png)

- **page-frame.html**:

  ​        存放了我们的页面结构的文件，整个小程序的 WXML 都会处理后放入这个文件，比如你可以看下截图

  ![](https://raw.githubusercontent.com/BuleAnt/RepositoryResources/master/image/RecommendationMovie/pmkz_2018-02-12_15.06.23.png)

- **app-config.json**

  ​        其实是我们小程序根目录的app.json，处理后就变成了app-config.json。格式化后的代码如下

  ![](https://raw.githubusercontent.com/BuleAnt/RepositoryResources/master/image/RecommendationMovie/pmkz_2018-02-12-15.10.49.png)

- **pages**：页面样式存放目录，实际上是将我们的 wxml 处理后，将 wxss 放在这里。

- **app-service.js**：页面逻辑所在位置，使用js格式化工具格式化， 我们等下就是要解析这个文件



### 3、查看程序逻辑，实验生成Authorization

#### 查找可能是 Authorization逻辑的代码段

​	将代码中折叠起来后发现，代码中载入了大量的模块，如下图所示，在这里看到 base64.js 、md5.js 、request.js 等模块，在载入模块语句内就是 这个模块的具体实现代码。

![](https://raw.githubusercontent.com/BuleAnt/RepositoryResources/master/image/RecommendationMovie/pmkz_2018-02-12_15.26.59.png)

​	这里 猜测request.js 是对请求生成授权码的代码部分，展开utils/request.js 下的代码，将代码粘出来到另一个文件中，由上到下进行解析，只看下面注释部分，后面的代码部分 不重要，不想看代码的，可以直接跳到最后总结部分。

```javascript
exports.request = function (i) {
  return new Promise(function (s, u) {
    /* url = 'https://wechat.lianjia.com/ershoufang/search?city_id=310000&condition=&query=&order=&offset=0&limit=10&sign='
    Authorization = bGp3eGFwcDoxYmU3OThjZDg0ZWU4NzNmM2JhMzM0NTFhZTNkNWUwMA==
    */
    // 这里主要进行的是一个相对URL转换为绝对URL的过程，这个不重要。
    i.url = 0 != i.url.indexOf("http") ? "https://wechat.lianjia.com/" + i.url : i.url,

      r().then(function (r) {
      // 在这里 配置请求头信息
      var c = o("lianjia_header", !0);
      c["Lianjia-Session"] = o("session_id"),
        c["Lianjia-Source"] = "ljwxapp",
        c["OS-Version"] = r.platform + "-" + r.system,
        c["Wx-Version"] = r.version,
        c["Wxminiapp-SDK-Version"] = r.SDKVersion,
        c["Lianjia-Wxminiapp-Version"] = .1,
        c["Time-Stamp"] = +new Date,
        i.data = i.data || {};

      // 对请求URL进行 URL + city_id  进行URL格式编码，并从传进来的参数重取出sign, 这里sign 跳过
      var f = encodeURIComponent(i.url) + i.data.city_id || "";
      i.data.sign = o(f);
	  /* 这里定义了一个变量 l, 并空串，对参数key进行排序后组成一个字符串。
	  1、参数部分：
	  city=310000&condition=&query=&order=&offset=0&limit=10&sign=’，
	  2、解析成key-value 键值对：
	  {'city_id': '310000', 'condition': '', 'query': '', 'order': '', 'offset': '0', 'limit': '10', 'sign': ''},
      3、从小到大排序后为：
      {'city_id': '310000', 'condition': '', 'limit': '10', 'offset': '0', 'order': '', 'query': '', 'sign': ''}
      4、用等号连接 key ,value值，并追加到 变量l 中为：
      'city_id=310000condition=limit=10offset=0order=query=sign='
   	  */
      var l = "";
      Object.keys(i.data).sort().forEach(function (e) {
        return l += e + "=" + ("object" == a(i.data[e]) ? JSON.stringify(i.data[e]) : i.data[e])
      }),
        // 这里对l 加上了一个后缀‘6e8566e348447383e16fdd1b233dbb49’，现在l = ‘city_id=310000condition=limit=10offset=0order=query=sign=6e8566e348447383e16fdd1b233dbb49’，
        l += "6e8566e348447383e16fdd1b233dbb49",
        // 获取变量 l 的MD5值：为 '1be798cd84ee873f3ba33451ae3d5e00'
        l = e.default.hexMD5(l),
        // 为 l 的MD5值加上前缀‘ljwxapp:’，现在l 为 ‘ljwxapp:1be798cd84ee873f3ba33451ae3d5e00’
        l = "ljwxapp:" + l,
        // 通过 后面代码中的t 对象中载入了 base64 模块，猜测这里对 l 进行了base64编码
        c.Authorization = t.encode(l),
		// 这里基本完成了 Authorization 的生成过程，后面我们去将生成过程总结一下。

        i.header = Object.assign(i.header || {}, c),
        i.method && "post" == i.method.toLocaleLowerCase() && (i.header["content-type"] = i.header["content-type"] || "application/x-www-form-urlencoded");
      var p = i.success,
          y = i.fail;
      i.success = function (a) {
        a.header["Lianjia-Uuid"] && d("lianjia_header", {
          "Lianjia-Uuid": a.header["Lianjia-Uuid"]
        }),
          a.data && 20007 == a.data.error_code ? i.custom || (n(), console.log("用户未登录")) : a.data && 0 == a.data.error_code ? (a.data.data && 0 === a.data.data.has_new ? a.data.data = o(i.data.sign, !0) : a.data.data && 1 === a.data.data.has_new && a.data.data.sign && (i.data.sign && d(i.data.sign, ""), d(f, a.data.data.sign), d(a.data.data.sign, a.data.data)), s(a.data), p && p(a.data)) : (a.header["Lianjia-Uuid"] && d("lianjia_header", {
          "Lianjia-Uuid": a.header["Lianjia-Uuid"]
        }), u(a.data), y && y(a.data))
      },
        i.fail = function (a) {
        u(a.data),
          y && y(a.data)
      },
        wx.request(i)
    })
  })
};
// 这里是最后的 t 对象，不重要
 var t = function (a) {
        if (a && a.__esModule) return a;
        var e = {};
        if (null != a) for (var t in a) Object.prototype.hasOwnProperty.call(a, t) && (e[t] = a[t]);
        return e.
          default = a,
          e
      }(require("base64.js")),
      n = require("./navToLogin.js"),
      i = void 0,
      r = function () {
        return new Promise(function (a, e) {
          i ? a(i) : wx.getSystemInfo({
            success: function (e) {
              a(i = e)
            },
            fail: function (a) {
              e({})
            }
          })
        })
      },

      o = function (a, e) {
        try {
          var t = wx.getStorageSync(a);
          return e ? JSON.parse(t || "{}") : t || ""
        } catch (a) {
          return e ? {} : ""
        }
      },

      d = function (e, t) {
        wx.setStorage({
          key: e,
          data: "object" == (void 0 === t ? "undefined" : a(t)) ? JSON.stringify(t) : t
        })
      };
```



### 4、验证迭代、实现效验方式

将上面的注释部分提取下来，为以下过程, 这里以获取二手房信息为例

Url = 'https://wechat.lianjia.com/ershoufang/search?city_id=310000&condition=&query=&order=&offset=0&limit=10&sign='

Authorization = 'bGp3eGFwcDoxYmU3OThjZDg0ZWU4NzNmM2JhMzM0NTFhZTNkNWUwMA=='

#### 效验过程

> 1、获取参数部分，这里为 ‘city=310000&condition=&query=&order=&offset=0&limit=10&sign=’
>
> 2、对参数进行从大到小排序、并将key-value用等号连接起来，将所有的元素连接成为一个字符串 S
>
> ​      排序后：
>
> ​           {'city_id': '310000', 'condition': '', 'limit': '10', 'offset': '0', 'order': '', 'query': '', 'sign': ''}
>
> ​      key-value用等号连接起来，并连接所有元素：
>
> ​           S =  'city_id=310000condition=limit=10offset=0order=query=sign='
>
> 3、 添加后缀 "6e8566e348447383e16fdd1b233dbb49"到变量S中，获取S的MD5值。
>
> 4、为变量 S 的MD5值 添加前缀 ‘ljwxapp:’，并作base64 编码，生成为：‘bGp3eGFwcDoxYmU3OThjZDg0ZWU4NzNmM2JhMzM0NTFhZTNkNWUwMA==’的 Authorization



#### Python 实现代码如下

```python
AUTHORIZATION_SIFFOX = "6e8566e348447383e16fdd1b233dbb49"
AUTHORIZATION_PREFIX = 'ljwxapp:'

def get_authorization(data):
	"""
    获取 authorization
    :param data:
    	例子参数
        {'city_id': '310000', 'condition': '', 'query': '', 'order': '',
        'offset': '0', 'limit': '10', 'sign': ''}
    :return:
        例子 authorization 返回值
        b'bGp3eGFwcDoxYmU3OThjZDg0ZWU4NzNmM2JhMzM0NTFhZTNkNWUwMA=='
    """

    global AUTHORIZATION_SIFFOX
    global AUTHORIZATION_PREFIX
    l = ""
    data_sort = dict_sort(data)
    l += ''.join([key + '=' + str(data_sort[key]) for key in data_sort.keys()])
    l += AUTHORIZATION_SIFFOX
    l_md5 = hashlib.md5(l.encode()).hexdigest()
    authorization_source = AUTHORIZATION_PREFIX+l_md5
    authorization = base64.b64encode(authorization_source.encode())

    return authorization.decode()
```

更详细的代码见我的gitHub项目[reverse_lianjia_wxapkg](https://github.com/Ant-Ferry/reverse_lianjia_wxapkg)



***参考文章***：

微信小程序获取与解压：https://github.com/billfeller/billfeller.github.io/issues/200

.wxapkg文件结构：https://cloud.tencent.com/developer/article/1012647



***声明***

此次解析 仅为 研究、学习。