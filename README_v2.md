# 参考来源

[官方文档](https://bbs.nga.cn/read.php?tid=6406100&authorid=58)

# 通用

## 域名

NGA有三个域名，其功能相同，如果有一个故障可以尝试其他的

- https://ngabbs.com/ (app复制的链接会给这个)
- https://bbs.nga.cn/ (主站)
- https://nga.178.com/ (备用)

## API

NGA的接口API地址均为在域名后加`*.php`，除上传以外有如下API：

- `thread.php`：查询主题
- `read.php` ：查询主题内容
- `forum.php`：版面操作
- `post.php`：发帖操作
- `nuke.php`：其他操作

## 请求方法和传参方式

部分接口要求为POST，部分可以用GET，建议统一用POST。

所有参数均可以使用`QueryString`和`Forum-Data`方式传递，鉴于`QueryString`有长度限制，建议把正文之类的字段放到`Form-Data`中传递，可以二者都传，会合并识别。



`axios`的参考方案，在配置对象中使用如下字段，此时`data`字段的内容即被作为`Form-Data`传递

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



传入参数`__inchst=UTF8` 可以在传参时使用UTF-8编码，建议无脑使用；注意：它不影响响应编码，GBK依然需要解码。

## User-Agent

将header中的`User-Agent`字段设置为`NGA_WP_JW`，可以拿到更多信息，如用户信息中的部分字段，非该值时是拿不到数据的。

## 输出格式

传入`__output` 参数，可指定返回的数据格式，原文介绍了多种参数值，常用的仅为：

- `8`：`GBK`编码返回`json`数据
- `11`：`UTF-8`编码返回`json`数据

注意：

- `11`在以前会有概率出现乱码，但是似乎已经修复，如果测试没有问题建议直接使用`11`
- `8`在解码时建议使用`GB18030`字符集，否则部分繁体字可能无法正确解码。

`axios`的解码方案参考：

```js
responseType: 'blob',
transformResponse: [function (data) {
        let reader = new FileReader();
        reader.readAsText(data, 'GBK');
        return new Promise(resolve => {
            reader.onload = function () {
                let result = reader.result;           
                resolve(JSON.parse(result))
            }
        });
    }]
```

## JSON格式不严谨的情况

1. 回复对象的`alterinfo`字段中会出现`\t`字符，会导致解析报错；同时私信的参与用户列表字段也将它作为分隔符使用；因此不能简单地删除所有`\t`字符，建议的处理方法是先替换为其他字符，解析为JSON之后再分别处理。
2. 个别字段以数字而不是"数字"作为字段名，可能导致解析报错；版面数据的`custom_level`字段以JSON字符串保存数据，字段名也没有带引号；可以使用这个正则表达式`([,{ ])(\\w+?):`（注意空格不能删），将其替换为`$1\"$2\":`来添加引号。
3. 个别字段可能出现有时为数组，有时为对象(以数字为字段名)的情况，因此遍历时需要采取一些通用方式。

## 时间戳

所有使用时间戳的地方，均使用的为`UNIX时间戳`，单位为“秒”。在JS中可以使用`new Date(timestamp*1000)`转换为`Date`对象

## 匿名唯一标识转换为对应中文乱码的方法

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


// #anony_8905635a3dc3a79511fc6217423df746 --> 壬柯顾己蓝应
```

## 官方表情包

可以通过请求[这个JS文件](https://img4.nga.178.com/common_res/js_bbscode_core.js)获得，其中的`ubbcode.smiles`对象即为表情数据，第一层级的key为表情分组名称，分组内的key为表情名称，value为对应的图片名称；分组内的key`_______name`为该分组的中文名。

正文中引用表情的格式为：`[s:组名称:表情名称]`， 对应的图片绝对路径为在图片名称前添加前缀：`https://img4.nga.178.com/ngabbs/post/smile/`

## 标题字体数据的解析方法

原始数据来自主题数据的`topic_misc`字段，参考官方文档`2.5.2`章节

`JS`的参考实现：

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

其中删除线可对应 CSS：` text-decoration:line-through`

# API

## thread

## read

## forum

## post

## nuke