# nga-api
NGA网页版实用API

# 参考来源

[官方文档](https://bbs.nga.cn/read.php?tid=6406100)

鉴于官方文档长期未更新，外加上自己的踩坑经验，故有此文。



但接口的响应对象各字段含义依然可以按照官方文档。

# 各接口通用参数

## 输出格式

在params中传入 `__output` 参数，可指定返回的数据格式

| 值   | 格式                                                     | Content-Type                 |
| ---- | -------------------------------------------------------- | ---------------------------- |
| 1    | json字符串（有前缀：window.script_muti_get_var_store= ） | text/javascript; charset=GBK |
| 8    | json字符串（无前缀）                                     | text/javascript; charset=GBK |
| 9    | xml格式                                                  | application/xml              |
| 11   | json对象                                                 | text/json;charset=UTF-8      |
| 14   | json对象                                                 | text/json;charset=UTF-8      |

说明：

- 11和14是目前最方便通用的方法，其返回结果只有外层结构略有不同；但是14在一些接口不可用（如read.php)；另外此二均存在个极别用户名或回复正文乱码的情况（如[此贴的2楼回复](https://bbs.nga.cn/read.php?tid=26639977&__output=11),神奇的是[使用pid请求时](https://bbs.nga.cn/read.php?pid=513778921&__output=11)则无乱码），因此实际都不推荐使用。
- 剩下最方便的应该是8，拿到字符串之后自行解析为json对象即可，但是GBK编码对前端程序来说可能有点麻烦，后续我会给一个参考解决方案；另外回复中的 `alterinfo` 字段中可能存在 `\t` 字符 造成JS的JSON.parse()方法解析报错，需要先行删除。

## 输入编码

输入参数中可以在params中传入参数 `__inchst=UTF8` 以使用UTF-8编码，所以一般都是无脑使用这个参数

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

## 一些Json字符串不严格的情况

例如：获取提醒信息（回复提醒，赞踩提醒）的接口中返回的Json字符串结构不严格，对象的字段名使用了数字(1)而不是字符串("1")，会导致JSON.parse报错 我的解决方案是在上方 `transformResponse` 字段的方法中添加：

```js
let r1 = /\s\d{1,2}:/g;
let r2 = /,\d{1,2}:/g;
let res
while (res = r1.exec(result)){
	let startIndex = res.index
	let endIndex = startIndex + res[0].indexOf(":")
	result = result.substring(0,startIndex)
		+`"`+result.substring(startIndex,endIndex).trim()
		+`"`+result.substring(endIndex)
}
while (res = r2.exec(result)){
	let startIndex = res.index
	let endIndex = startIndex + res[0].indexOf(":")
	startIndex++
	result = result.substring(0,startIndex)
		+`"`+result.substring(startIndex,endIndex).trim()
		+`"`+result.substring(endIndex)
}
```



# API

## 获取主题列表

接口： /thread.php

### 响应对象的一些说明

- 主题标题的字体为 `titlefont` 或 `topic_misc`字段

  - 部分颜色对照

  - 官方文档有说明实际的解析方法，但是个人认为实际情况没那么复杂，可以直接用map

  - 倒数第二位为`C`的即为加粗

  - ```json
     [
      {value: "AQAAAAA", key: "灰色普通"},
      {value: "AQAAACA", key: "灰色加粗"},
      {value: "AQAAAAE", key: "红色普通"},
      {value: "AQAAACE", key: "红色加粗"},
      {value: "AQAAAAQ", key: "绿色普通"},
      {value: "AQAAACQ", key: "绿色加粗"},
      {value: "AQAAAAI", key: "蓝色普通"},
      {value: "AQAAACI", key: "蓝色加粗"},
      {value: "AQAAAAg", key: "棕色普通"},
      {value: "AQAAACg", key: "棕色加粗"},
    ]
    ```

  - 根据以上表格的参考CSS生成方法

  - ```js
    export const titleStyle = (titleFont)=> {
      let s = "";
      if (titleFont) {
        s += titleFont[5] === 'C' ? "font-weight: bold;" : "";
      } else {
        // console.log(titleFont)
        return s;
      }
      switch (titleFont[6]) {
        case "A":
          s += "color: gray;";
          break;
        case "E":
          s += "color: red;";
          break;
        case "Q":
          s += "color: green;";
          break;
        case "I":
          s += "color: blue;";
          break;
        case "g":
          s += "color: #A06700;";
          break;
      }
      return s;
    }
    ```

- dfsd

### 获取本账号的收藏主题

Params：

| 字段  | 值     |
| ----- | ------ |
| favor | 任意值 |

如： https://bbs.nga.cn/thread.php?favor=1&__output=11

### 搜索主题

Params：

| 字段    | 值                                        | 是否必须    |
| ------- | ----------------------------------------- | ----------- |
| key     | 关键字                                    | ✓           |
| fid     | 版面id                                    | ✓           |
| page    | 页码                                      | 不传默认为1 |
| content | 0 - 在标题搜索关键字 1 - 也在主楼正文搜索 | 不传默认为0 |

### 获取版面主题

Params：

| 字段       | 值                                                  | 是否必须    |
| ---------- | --------------------------------------------------- | ----------- |
| fid        | 版面id                                              | ✓           |
| page       | 页码 不传默认为1                                    | 不传默认为1 |
| authorid   | 用户uid，如果传入表示搜索该用户在该版面的主题或回复 |             |
| searchpost | 任意值，如果传入表示搜索用户回复                    |             |

### 获取合集内主题

Params：

| 字段 | 值               | 是否必须    |
| ---- | ---------------- | ----------- |
| stid | 合集tid          | ✓           |
| page | 页码 不传默认为1 | 不传默认为1 |

## 获取主题内容/回复

接口： /read.php

### 响应对象的一些说明

- 评论贴条的正文内容保存在被评论回复的 `comment` 字段中
- 热评内容保存在主楼的`hotreply`字段中
