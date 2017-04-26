---
title: "InfluxDB 写数据语法"
date: 2017-04-19 16:09
---


本篇是官方文档的一个蹩脚翻译，原文：[Write Syntax](https://docs.influxdata.com/influxdb/v0.13/write_protocols/write_syntax/)

语法总是容易忘记，这里是一个参考手册

[TOC]

# 行协议 ( Line Protocol ) 

行协议的语法是：

    measurement[,tag_key1=tag_value1...] field_key=field_value[,field_key2=field_value2] [timestamp]

例如：

    measurement,tkey1=tval1,tkey2=tval2 fkey=fval,fkey2=fval2 1234567890000000000

## 空格

measurement 和 filed(s) 之间 必需用空格分隔，如果有 tag(s) ， tag 和 field 间必需空格分隔。measurement 和 tag，tag 和 tag 之间必行用一个逗号分隔，并且之间不能有空格。如果有 timestamp，field 和 timestamp 之间也必需用空格分隔。

合法的例子  ( value 和 otherval 是 field, foo 和 bat 是 tag )

    measurement value=12
    measurement value=12 1439587925
    measurement,foo=bar value=12
    measurement,foo=bar value=12 1439587925
    measurement,foo=bar,bat=baz value=12,otherval=21 1439587925

不合法的

    measurement,value=12
    measurement value=12,1439587925
    measurement foo=bar value=12
    measurement,foo=bar,value=12 1439587925
    measurement,foo=bar
    measurement,foo=bar 1439587925

## Timestamp （时间戳）

Timestamp 不是必需。 如果没有写 timestamp，服务器在插入数据点时会写入当时的系统本地时间。如果写 timestamp，timestamp 必需和 field 之间以空格分隔。Timestamp 默认是纳秒为单位的 Unix 时间。也可以指定其他精度，参加 HTTP 语法的细节。我们建议尽可能使用最小的精度，因为这样可以显著地提高压缩率。

## 键值（key-value）分隔符

Tag 或 field 的键、值之间必需用 “=” 分隔

## 转义字符

如果一个 tag 的键或值，或者一个 field 的键包含空格，逗号，等号，必需使用反斜线转义。反斜线不必转义。在 measurement 中， 逗号和空格也需要转义，而等号不需要。

## 注释

以 # 为首的行是合法的注释行。

## 数据类型



Measurements, tag key, tag value 以及 field key 在数据库中总是以 string 保存。string 值得最大限制是 64 KB。所有的 Unicode 字符必须合法，并且逗号和空格需要转义。反斜线不需要转义，但是不能直接用在逗号和空格之前 （注意 string 类型的 field value 有另一套不同于 measurement，tag 和 field key 的应用转义规则）。字段（Field） location="us-west" 存储了一个 string 值。 Field 值可以是 float64，int64，boolean 或 string。所有的后续 field 值必须和第一次写入measurement 的数据点的 field 值类型相同。float64 是默认的数字类型。1 是 一个 float 类型数，1i 是一个 integer 类型数。int64 类型数必行跟着一个 i。Field bikes_present=15i 存储一个 integer， field bikes_present=15 存储一个 float。布尔值是 t，T，true，True，TRUE，f，false，False，FALSE 字符串，赋值给 field value 必须被双引号包围。双引号包围的字符串必须被转义，其他部分的字符可以不转义。

## 例子

### 最简单的合法写入点  (measurement + field)

    disk_free value=442221834240i

### 带 timestamp 的

    disk_free value=442221834240i 1435362189575692182

### 带 tag 的

    disk_free,hostname=server01,disk_type=SSD value=442221834240i

### 带 tag 时间戳

    disk_free,hostname=server01,disk_type=SSD value=442221834240i 1435362189575692182

### 多个 field

    disk_free free_space=442221834240i,disk_type="SSD" 1435362189575692182

### 转义逗号和空格

    total\ disk\ free,volumes=/net\,/home\,/ value=442221834240i 1435362189575692182

### 转义等号

    disk_free,a\=b=y\=z value=442221834240i

### tag value 中有斜线

    disk_free,path=C:\Windows value=442221834240i

斜线用在字符串中不需要转义，除了后面跟着逗号，空格或等号

### 转义 field key

    disk_free value=442221834240i,working\ directories="C:\My Documents\Stuff for examples,C:\My Documents"

### 一个例子展现所有转义和引用规则

    "measurement\ with\ quotes",tag\ key\ with\ spaces=tag\,value\,with"commas" field_key\\\\="string field value, only \" need be quoted"

上面的例子， measurement 是“带引用的 measurement”，tag key 是带空格的 tag key，tag value 是带逗号的 tag value，field key 是 filed_key\\\\ ，filed value 是字符串 string field value，only " need be quoted 。

## 警告

如果你批量写入数据点，并且所有的写入点没有显示的 timestamp，那么它们将被以相同的 timestamp 写入。因为一个数据点被 measurement，tag 集，timestamp 定义，所以这会导致数据点重复。当 InfluxDB 遇到重复数据点，field 会变成新旧 filed 集的 union，并且绑定到新的 fied 集。最佳实践是为所有写入点提供显示的 timestamp。 measurement，tag key，tag values 不用引用，空格和逗号必须转义。string 型的 field value 必须被双引号包围，只有索引号需要被转义。查询包好双引号的 measurement，tag 后比较困难，因为双引号在语法上也是一个标识符。在表达式遇到这些限制时，正确处理并不容易。避免使用标识符语法的关键词。InfluxDB 所有的值都是大小写敏感的： MyDB != mydb != MYDB。关键词例外，它们是大小写不敏感的。

# CLI

使用命令行接口写入数据点，使用 insert 命令

## 使用 CLI 写入一个数据点

    > insert disk_free,hostname=server01 value=442221834240i 1435362189575692182

成功时命令行不返回任何信息，如果数据点不能写入后给出一些解析器错误信息。使用 CLI 从数据文件导入数据的方法参见 InfluxDB CLI/Shell 。


# HTTP

写入数据点使用 HTTP，POST 到 8086 端口的 /write 端点。在查询字串中你必须使用 db=<target_database> 指定目标数据库。通过查询字串，你可以提供可选的 retention policy，指定任何你提供的任何 timestamp 的精度，通过任何必需的验证。成功的写入会返回 204 HTTP Stauts Code。语法错误会返回 400。

## 写入操作的查询字串参数

使用 HTTP API 时，下面的字符串参数作为 GET 字串的一部分时也能通过

- db=&lt;database&gt; 必需 —— 设置写入的目标数据库
- rp=&lt;retention_policy&gt; —— 设置写入的目标 retension policy （如果不是用目前默认的 retention policy 的话）
- u=&lt;username&gt;, p=&lt;password&gt; —— 如果数据库打开了鉴权，必需验证用户的写权限
- precision=[n,u,ms,s,m,h] —— 设置写入Unix 时间的精度，如果不设置默认使用纳秒

## 使用 curl 写入一个数据点

    curl -X POST 'http://localhost:8086/write?db=mydb' --data-binary 'disk_free,hostname=server01 value=442221834240i 1435362189575692182'

## 使用非默认 Retention Policy 写入一个数据点    

    curl -X POST 'http://localhost:8086/write?db=mydb&rp=six_month_rollup' --data-binary 'disk_free,hostname=server01 value=442221834240i 1435362189575692182'

## 写入一个需要鉴权的数据点

    curl -X POST 'http://localhost:8086/write?db=mydb&u=root&p=123456' --data-binary 'disk_free,hostname=server01 value=442221834240i 1435362189575692182'

## 指定非纳秒级时间戳

- n: 纳秒
- u: 微秒
- ms: 毫秒
- s: 秒
- m: 分
- h: 小时
 
Example 

    curl -X POST 'http://localhost:8086/write?db=mydb&precision=ms' --data-binary 'disk_free value=442221834240i 1435362189575'

## 使用 curl 批量写入多个数据点

你可以传入一个数据文件，只需要文件名前加 @ 符号。文件里可以包含多条数据，每条一行。数据必须以换行符 '\n' 分隔。我们建议一次批量的写入 5000 ~ 10000 条数据。少量的数据产生过多的 HTTP request，并不能达到最高效率。

    curl -X POST 'http://localhost:8086/write?db=<database>' --data-binary @<filename>

## 警告

使用 curl 的 --data-binary 编码所有行协议（line protocal） 格式的 write 数据。使用 --data-binary 以外的方法写数据点可能会有一些问题。 -d， --data-urlencode，--data-ascii 可能都会截掉换行符或者用其他格式替换。
