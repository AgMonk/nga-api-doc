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

目前看来所有参数都可以在Params或Form-Data中传入，鉴于Params的长度限制问题（特别是回复正文），推荐所有参数通过Form-Data传递，为此需要把`Headers`的`Content-Type`设置为`application/x-www-form-urlencoded`

Axios的参考方案，在配置对象中使用如下字段，此时`data`字段的内容即被作为`Form-Data`传递

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

注意：附件上传部分的请求并不使用本方法；我本人是使用`Element-plus ` 提供的上传组件，并没使用`axios`执行过上传操作。

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

### 标题字体的数据解析方法

- 参照官方文档章节： 2.5 主题其他数据(topic_misc)

- 主题标题的字体为 `titlefont` 或 `topic_misc`字段

JS实现:

```js

export const parseTitleFont = (data) => {
    //将字串使用base64解码，并切割为单字节数组
    const s = window.atob(data).split("");
    //将各字节转换为8位二进制数组（补齐位数）
    const array = s
        .map(i => bin2UInt(i).toString(2))
        .map(i => ('00000000' + i).slice(-8));
    const res = {}
    //以5为步长循环该数组（如果为合集主题array长度为10，否则为5）
    for (let i = 0; i < array.length - 1; i += 5) {
        //首字节表示数据类型
        const type = parseInt(array[i], 2) === 2 ? "stid" : 'bit'
        //将后续4个字节数据拼接并转换为十进制数
        let bit = parseInt(array.slice(i + 1, i + 5).join(''), 2)
        if (type === 'stid') {
            //如果数据类型为1 ， 表示bit为集合id
            res.stid = bit;
        }
        if (type === 'bit') {
            //如果是字体数据，把数据转换为2进制字符串，并反向方便后续处理
            res.titleFont = bit.toString(2).split("").reverse().join('')
        }
    }
    return res
}

//二进制字符串转为多字节整数(big-endian)
export const bin2UInt = (x) => {
    let z = 0, y = 0;
    for (let i = 0; i < x.length; i++) {
        y = x.charCodeAt(i)
        //如果输入字符串中有utf16字符则一次移动两字节
        z = (z << (y > 255 ? 16 : 8)) + y
    }
    return z
}

```

最终返回的 `res.titleFont` 为由01组成的字符串，根据每一位的值为 0（否） ，1（是），决定主题字体的格式，位数不存在等效于0

| 位数 | 含义       | 位数 | 含义   |
| ---- | ---------- | ---- | ------ |
| 1    | 红色       | 6    | 加粗   |
| 2    | 蓝色       | 7    | 斜体   |
| 3    | 绿色       | 8    | 删除线 |
| 4    | 橙（棕）色 |      |        |
| 5    | 银（灰）色 |      |        |

其中删除线可对应 CSS： text-decoration:line-through

### 获取本账号的收藏主题

Params：

| 字段  | 值                            |
| ----- | ----------------------------- |
| favor | 收藏夹id ， 填1则为默认收藏夹 |

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

| 字段 | 值      | 是否必须    |
| ---- | ------- | ----------- |
| stid | 合集tid | ✓           |
| page | 页码    | 不传默认为1 |

## 获取主题内容/回复

接口： /read.php

Params：

| 字段     | 值      | 是否必须                             |
| -------- | ------- | ------------------------------------ |
| tid      | 主题id  | tid 和 pid 二选一 当传入pid时tid无效 |
| pid      | 回复id  | tid 和 pid 二选一 当传入pid时tid无效 |
| page     | 页码    | 不传默认为1，传 e 时返回最后一页     |
| authorid | 用户uid | 如果传入表示`只看TA`功能             |

### 响应对象的一些说明

- 评论贴条的正文内容保存在被评论回复的 `comment` 字段中，其所在楼层无正文内容。

- 热评内容保存在主楼的`hotreply`字段中，其所在楼层也有相同内容。

- `__U`字段中为该页出现的用户信息，其中`__REPUTATIONS`字段为这些用户的本版声望

- `__F`字段中的`custom_level`字段为本版面声望数值对应的声望等级。其值为以json字符串格式书写的一个数组。使用方法为：反向遍历该数组，当声望数值大于等于当前对象的 `r` 字段时，其 `n` 字段即为对应的声望等级。**注意这里的字段名是不严格的，解析前需要补足引号。**

- `__R`字段有时为 对象有时为数组，即便是只有一个`page`参数不同也可能不一样，故如果需要前端遍历它时最好使用`Object.keys(res.__R).forEach(key=>{let item = res.__R[key]})` ~~虽然vue的v-for不受影响~~。

- 匿名用户在回复中的 `authorid`为负数，在`__U`中下有对应的负数字段中 `username`字段为该匿名用户在该主题中的唯一标识（**注意 该负数不是唯一标识，每一页都可能不一样**），。

- `alterinfo`字段中包括了回复的编辑记录（E开头）、版主的处罚记录（L开头）、处罚撤销记录（U开头）

- 正文部分有部分转义字符(& quot; 等)在显示和编辑时需要反转义

  ```js
  export const unEscape = (text) => {
      let temp = document.createElement("div");
      temp.innerHTML = text.replace(/<br\/>/g, "\n");
      let output = temp.innerText || temp.textContent;
      temp = null;
      return output;
  }
  ```

### 匿名唯一标识转换为对应中文乱码的方法

从官方文件 [js_commonui.js](https://img4.nga.178.com/common_res/js_commonui.js) 中获知 ，可搜索 “甲乙丙丁”

略微修改如下:

```js
const t1 = "甲乙丙丁戊己庚辛壬癸子丑寅卯辰巳午未申酉戌亥"
const t2 = '王李张刘陈杨黄吴赵周徐孙马朱胡林郭何高罗郑梁谢宋唐许邓冯韩曹曾彭萧蔡潘田董袁于余叶蒋杜苏魏程吕丁沈任姚卢傅钟姜崔谭廖范汪陆金石戴贾韦夏邱方侯邹熊孟秦白江阎薛尹段雷黎史龙陶贺顾毛郝龚邵万钱严赖覃洪武莫孔汤向常温康施文牛樊葛邢安齐易乔伍庞颜倪庄聂章鲁岳翟殷詹申欧耿关兰焦俞左柳甘祝包宁尚符舒阮柯纪梅童凌毕单季裴霍涂成苗谷盛曲翁冉骆蓝路游辛靳管柴蒙鲍华喻祁蒲房滕屈饶解牟艾尤阳时穆农司卓古吉缪简车项连芦麦褚娄窦戚岑景党宫费卜冷晏席卫米柏宗瞿桂全佟应臧闵苟邬边卞姬师和仇栾隋商刁沙荣巫寇桑郎甄丛仲虞敖巩明佘池查麻苑迟邝'

export const getAnonyName = (name) => {
    let i = 6;

    let s = ""
    for (let j = 0; j < 6; j++) {
        if (j === 0 || j === 3)
            s += t1.charAt(('0x0' + name.substr(i + 1, 1)) - 0)
        else if (j < 6)
            s += t2.charAt(('0x' + name.substr(i, 2)) - 0)
        i += 2
    }
    return s
}


// #anony_8905635a3dc3a79511fc6217423df746
```

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

### 获取本账号的收藏夹列表

Params：

| 字段  | 值                      | 是否必须 |
| ----- | ----------------------- | -------- |
| __lib | 固定为 "topic_favor_v2" | ✓        |
| __act | 固定为 "list_folder"    | ✓        |

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

