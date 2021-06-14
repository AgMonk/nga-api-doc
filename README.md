# nga-api
NGA网页版实用API

# 参考来源

[官方文档](https://bbs.nga.cn/read.php?tid=6406100)

鉴于官方文档长期未更新，外加上自己的踩坑经验，故有此文。

# 各接口通用参数

## 输出格式

在params中传入 **__output** 参数，可指定返回的数据格式

| 值   | 格式                                                     | Content-Type                 |
| ---- | -------------------------------------------------------- | ---------------------------- |
| 1    | json字符串（有前缀：window.script_muti_get_var_store= ） | text/javascript; charset=GBK |
| 8    | json字符串（无前缀）                                     | text/javascript; charset=GBK |
| 9    | xml格式                                                  | application/xml              |
| 11   | json对象                                                 | text/json;charset=UTF-8      |
| 14   | json对象                                                 | text/json;charset=UTF-8      |

说明：

- 11和14是目前最方便通用的方法，其返回结果只有外层结构略有不同；但是14在一些接口不可用（如read.php)；另外此二均存在个极别用户名或回复正文乱码的情况（如[此贴的2楼回复](https://bbs.nga.cn/read.php?tid=26639977&__output=11),神奇的是[使用pid请求时](https://bbs.nga.cn/read.php?pid=513778921&__output=11)则无乱码），因此实际都不推荐使用。
- 剩下最方便的应该是8，拿到字符串之后自行解析为json对象即可，但是GBK编码对前端程序来说可能有点麻烦，后续我会给一个参考解决方案；另外回复中的 **alterinfo** 字段中可能存在 **\t** 字符 造成JS的JSON.parse()方法解析报错，需要先行删除。

## 输入编码

输入参数中可以在params中传入参数 **__inchst=UTF8** 以使用UTF-8编码，所以一般都是无脑使用这个参数

## JavaScript中GBK解码方案

这里是 **axios** 的解决方案，其他请求工具自行查找了

在axios的配置对象中使用如下字段，这里一并解决了上述的 **\t** 问题

```js
responseType: 'blob',
transformResponse: [function (data) {
        let reader = new FileReader();
        reader.readAsText(data, 'GBK');
        return new Promise(resolve => {
            reader.onload = function () {
                let result = reader.result;
                while (result.includes("\t")) {
                    result = result.replace("\t", "")
                }
                resolve(JSON.parse(result))
            }
        });
    }]
```

