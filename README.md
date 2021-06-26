# nga-api
NGA网页版实用API

# 参考来源

[官方文档](https://bbs.nga.cn/read.php?tid=6406100)

鉴于官方文档长期未更新，外加上自己的踩坑经验，故有此文。



但接口的响应对象各字段含义依然可以按照官方文档。

# 通用

## 请求方法

部分接口要求为POST，部分可以用GET，建议统一用POST

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

输入参数中可以在params中传入参数 `__inchst=UTF8` 以在传参时使用UTF-8编码，所以一般都是无脑使用这个参数。**注意 ：此参数不影响输出编码，GBK该解码的依然要解码**

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

正则前面加一个字符是为了避开value中可能出现的时间格式（如：10:00）

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

## 使用Form-Data传参

目前看来所有参数都可以在Params或From-Data中传入，鉴于Params的长度限制问题（特别是回复正文），推荐所有参数通过Form-Data传递，为此需要把`Headers`的`Content-Type`设置为`application/x-www-form-urlencoded`

Axios的参考方案，在配置对象中使用如下字段

```js
headers:{'Content-Type': 'application/x-www-form-urlencoded'},
transformRequest:[
    function (data) {
        let ret = ''
        for (let it in data) {
            ret += encodeURIComponent(it) + '=' + encodeURIComponent(data[it]) + '&'
        }
        ret = ret.substring(0, ret.lastIndexOf('&'));
        return ret
    }
],
```

## 时间戳

NGA所有涉及到时间戳的地方，均使用的为UNIX秒，在转换为实际时间时应当使用 `new Date(timestamp*1000)`

另外还可以为Date对象添加一个原型方法：

```js
Date.prototype.format = function (fmt) {
  let o = {
    "M+": this.getMonth() + 1,                 //月份
    "d+": this.getDate(),                    //日
    "h+": this.getHours(),                   //小时
    "m+": this.getMinutes(),                 //分
    "s+": this.getSeconds(),                 //秒
    "q+": Math.floor((this.getMonth() + 3) / 3), //季度
    "S": this.getMilliseconds()             //毫秒
  };

  if (/(y+)/.test(fmt)) {
    fmt = fmt.replace(RegExp.$1, (this.getFullYear() + "").substr(4 - RegExp.$1.length));
  }

  for (let k in o) {
    if (new RegExp("(" + k + ")").test(fmt)) {
      fmt = fmt.replace(
        RegExp.$1, (RegExp.$1.length === 1) ? (o[k]) : (("00" + o[k]).substr(("" + o[k]).length)));
    }
  }
  return fmt;
}
```

即可使用 `new Date(timestamp*1000).format("yyyy-MM-dd hh:mm:ss")`输出为常见的时间格式

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
| page       | 页码                                                | 不传默认为1 |
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

- 评论贴条的正文内容保存在被评论回复的 `comment` 字段中，其所在楼层无正文内容。

- 热评内容保存在主楼的`hotreply`字段中，其所在楼层也有相同内容。

- `__U`字段中为该页出现的用户信息，其中`__REPUTATIONS`字段为这些用户的本版声望

- `__F`字段中的`custom_level`字段为本版面声望数值对应的声望等级。其值为以json字符串格式书写的一个数组。使用方法为：反向遍历该数组，当声望数值大于等于当前对象的 `r` 字段时，其 `n` 字段即为对应的声望等级。**注意这里的字段名是不严格的，解析前需要补足引号。**

- `__R`字段有时为 对象有时为数组，即便是只有一个`page`参数不同也可能不一样，故如果需要前端遍历它时最好使用`Object.keys(res.__R).forEach(key=>{let item = res.__R[key]})` ~~虽然vue的v-for不受影响~~。

- 匿名用户在回复中的 `authorid`为负数，在`__U`中下有对应的负数字段中 `username`字段为该匿名用户在该主题中的唯一标识（**注意 该负数不是唯一标识，每一页都可能不一样**），该唯一标识转换为常见的乱码中文的方法未明。

- `alterinfo`字段中包括了回复的编辑记录（E开头）、版主的处罚记录（L开头）、处罚撤销记录（U开头）

- 

- 正文部分有部分转义字符在显示和编辑时需要替换

  - 
    
    ```js
    reply.content = reply.content.toString()
    	.replace(/&quot;/g, "\"")
    	.replace(/&amp;/g, "&")
    	.replace(/&lt;/g, "<")
    	.replace(/&gt;/g, ">")
    	.replace(/&#39;/g, "'")
    ```

### 获取回复列表

Params：

| 字段     | 值      | 是否必须                             |
| -------- | ------- | ------------------------------------ |
| tid      | 主题id  | tid 和 pid 二选一 当传入pid时tid无效 |
| pid      | 回复id  | tid 和 pid 二选一 当传入pid时tid无效 |
| page     | 页码    | 不传默认为1，传 e 时返回最后一页     |
| authorid | 用户uid | 如果传入表示`只看TA`功能             |

## 综合操作

接口： /nuke.php

### 获取指定用户信息

Params：

| 字段     | 值           | 是否必须                           |
| -------- | ------------ | ---------------------------------- |
| __lib    | 固定为 "ucp" | ✓                                  |
| __act    | 固定为 "get" | ✓                                  |
| uid      | 用户id       | uid和username二选一，uid优先级更高 |
| username | 用户名       | uid和username二选一，uid优先级更高 |

这里的返回信息并不能拿到实际用户名（除了自己，其他的都是 UIDxxxxx），只能从`thread` `read`等接口中间接获取 uid 和 用户名的对应关系

### 收藏版面操作

Params：

| 字段   | 值                         | 是否必须       |
| ------ | -------------------------- | -------------- |
| __lib  | 固定为 "forum_favor2"      | ✓              |
| __act  | 固定为 "forum_favor"       | ✓              |
| action | 有效值为 "get" "add" "del" | ✓              |
| fid    | 版面id                     | add和del时必须 |

### 子版面操作

Params：

| 字段      | 值                         | 是否必须                                |
| --------- | -------------------------- | --------------------------------------- |
| __lib     | 固定为 "user_option"       | ✓                                       |
| __act     | 固定为 "set"               | ✓                                       |
| raw       | 固定为 3                   | ✓                                       |
| fid       | 主板面id                   | ✓                                       |
| type      | 固定为 1                   | ✓                                       |
| info      | 固定为 "add_to_block_tids" | ✓                                       |
| del / add | id 含义后详                | 字段名选择 del 代表关注，选择 add为取关 |

Parmas中的 id 数据可以来自两个地方：

- 主题列表中会存在已关注的子版面入口，其数据格式类似一个主题，其 `tid`字段 即为本id
- `__F`字段中的`sub_forums`字段保存着本版面下所有子版面信息，每个信息为一个数组，其中第`4`个成员即为本id

### 赞踩

Params：

| 字段  | 值                       | 是否必须 |
| ----- | ------------------------ | -------- |
| __lib | 固定为 "topic_recommend" | ✓        |
| __act | 固定为 "add"             | ✓        |
| tid   | 主题id                   | ✓        |
| pid   | 赞踩的回复id             | ✓        |
| raw   | 固定为 3                 | ✓        |
| value | 1 = 赞 ， - 1  = 踩      | ✓        |

返回对象中会告诉你本次操作造成的赞踩和的变化量，例如你原本是赞，在取消赞之前直接点踩，返回的变化量为-2

### 提醒消息

#### 获取提醒消息

Params：

| 字段       | 值               | 是否必须 |
| ---------- | ---------------- | -------- |
| __lib      | 固定为 "noti"    | ✓        |
| raw        | 固定为 3         | ✓        |
| __act      | 固定为 "get_all" | ✓        |
| time_limit | 固定为 1         | ✓        |

- 返回对象是不严格的json字符串，需按前文先行处理再解析

- res.data["0"] 下是具体数据 各字段含义为：字段 `0` =  回复提醒 ，字段`1` = 短消息提醒 ， 字段`2` = 赞踩数提醒

- 一个参考的解析方法

- ```js
  // 回复提醒
  let replies = res.data["0"]["0"];
  replies = !replies ? undefined : replies.map(reply => {
      return {
          authorId: reply["1"],
          authorName: reply["2"],
          repliedId: reply["3"],
          repliedName: reply["4"],
          threadSubject: reply["5"],
          tid: reply["6"],
          replyPid: reply["7"],
          repliedPid: reply["8"],
          timestamp: reply["9"],
          page: reply["10"],
          timeString: new Date(reply["9"] * 1000).format("yyyy-MM-dd hh:mm:ss")
      }
  }).reverse();
  
  // 短消息提醒
  let pm = res.data["0"]["1"];
  pm = !pm ? undefined : pm.map(r => {
      return {
          authorId: r["1"],
          authorName: r["2"],
          mid: r["6"],
          timestamp: r["9"],
          timeString: new Date(r["9"] * 1000).format("yyyy-MM-dd hh:mm:ss")
      }
  }).reverse();
  // 赞踩变化
  let approbation = res.data["0"]["2"];
  approbation = !approbation ? undefined : approbation.map(r => {
      return {
          uid: r["3"],
          threadSubject: r["5"],
          tid: r["6"],
          pid: r["7"],
          timestamp: r["9"],
          timeString: new Date(r["9"] * 1000).format("yyyy-MM-dd hh:mm:ss")
      }
  }).reverse();
  
  return {replies, approbation, pm}
  ```

#### 不再提示

Params：

| 字段    | 值                | 是否必须 |
| ------- | ----------------- | -------- |
| func    | 固定为 "noti_tag" | ✓        |
| no_hint | 固定为 1          | ✓        |
| tid     | 主题id            | ✓        |
| pid     | 回复id            | ✓        |
| raw     | 固定为 3          | ✓        |

注：这是赞踩消息的“不再提醒”功能，我没有尝试回复消息的该功能是否也是相同的格式。

#### 清空消息

清空回复、私信、赞踩消息列表

Params：

| 字段  | 值            | 是否必须 |
| ----- | ------------- | -------- |
| __lib | 固定为 "noti" | ✓        |
| raw   | 固定为 3      | ✓        |
| __act | 固定为 "del"  | ✓        |

## 搜索版面

接口：/forum.php

Params：

| 字段 | 值     | 是否必须 |
| ---- | ------ | -------- |
| key  | 关键字 | ✓        |

## 发帖、回复

接口：/post.php

直接看 [官方文档](https://bbs.nga.cn/read.php?pid=118598798) 比较全面，仅补充一下

- 不传正文字段时为“回复准备”功能，会返回一些预设代码（即点击“引用”和“回复”时看到输入框中的代码）；但发表回复并不必须先做“准备”，除非需要上传图片时（需要获取图片的上传地址）

