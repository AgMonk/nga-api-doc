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

## User-Agent

将header中的`User-Agent`字段设置为`NGA_WP_JW`，可以拿到更多信息，如用户信息中的部分字段，非该值时是拿不到数据的。

## 输出格式

传入`__output` 参数，可指定返回的数据格式，原文介绍了多种参数值，常用的仅为：

- `8`：`GBK`编码返回`json`数据
- `11`：`UTF-8`编码返回`json`数据

11在以前会有概率出现乱码，但是似乎已经修复，如果测试没有问题建议直接使用11

注意，如果使用8，解码时建议使用`GB18030`字符集，否则部分繁体字可能无法正确解码。