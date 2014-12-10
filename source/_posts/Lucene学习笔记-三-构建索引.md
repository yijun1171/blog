title: Lucene学习笔记(三)构建索引
date: 2014-12-07 17:07:25
tags: search
categories: java
---

# 构建索引
一个好的索引构建策略是搜索程序成功的基石
<!--more-->
## 对搜索内容进行建模

### 文档和域
Document是索引和搜索的原子单位,Field包含真正的搜索的内容.域中的值会被分析成粒度更小的语汇单元,对这样的语汇单元进行索引.
  例如,存在这样的域:`"content":"hello world! i'm new here"`,根据不同的分析策略,域值将会被分割成不同的语汇单元的集合,并分别建立索引.假如按照独立的单词进行分别索引,搜索域中任意一个单词,例如`new`,该文档都会被搜索出来.

### 反向规格化
  真实的文档之间可能存在嵌套,指向,主次之类的关系,而Lucene的Document都是单一的,为了解决这一矛盾需要进行**反向规格化**操作

---

## 索引过程

1. 将原始文档转换成文本
2. 分析文本
3. 将分析好的文本保存至索引

### 提取和转换文本
  Lucene建立索引需要文本格式的数据内容,所以第一项任务就是从各种各样的数据来源中提取出文本内容.(提取文本信息的细节可以借助**Tika**框架)

### 分析文档
  处理文本内容,例如转换大小写,去除无意义的冠词(针对英语),总之是利用一些过滤和转换手段将文本转换成利于有效索引的语汇单元.分析器可以自己定制也可以使用现成的.

### 向索引添加文档
  Lucene使用反向索引,即一个*语汇单元*指向所有包含它的*文档*.

## 基本索引操作

### 添加文档
`indexWriter.addDocument(Document)` 使用默认分析器
`indexWriter.addDocument(Document,Analyzer)` 使用指定分析器

### 删除文档
``deleteDocument(Term)`` 删除包含项的所有文档
`deleteDocument(Term[])` 删除包含任意项的所有文档
`deleteDocument(Query)` 删除满足查询条件的所有文档
`deleteDocument(Query[])` 删除满足任一查询条件的所有文档

如何实现删除单一指定文档?
使用类似数据库主键的**Field**来标识每个文档:`new Field("id","**")`,id不会重复.

#### commit与optimize
`commit()`操作只会标记那些被执行删除操作的文档,当周期性刷新文档目录时才真正删除该文档.

`optimize()`操作强制删除文档后合并索引

#### maxDoc与numDoc
`maxDoc()`返回被删除和未被删除的文档总数
`numDoc()`返回未被删除的文档总数

### 更新文档
`updateDocument(Term, Document)` 首先删除包含Term变量的所有文档,然后使用writer的默认分析器添加新文档

---

## 域选项
4.10版本通过 `FieldType`对象和它的`set()`方法来控制域选项

### 域索引选项
常用选项:
`ANALYZED` 将域值分解成独立的语汇单元,使得每个单元都能被搜索
`NOT_ANALYZED` 域值整体作为语汇单元,用于精确匹配
`NO` 该域不能被搜索
4.10中
通过`FieldType.setTokenized()`设置是否将域值分解
通过`FieldType.setIndex()`设置该域值是否建立索引

### 域存储选项
`setStored()` 如果设置为不保存域值,则该值不会在索引中保存,即搜索完毕后,不可从Document对象中访问.

### 域排序选项
NumericType

### 多值域
以相同的域名添加不同域值
```
Document doc = new Document();
FieldType type = new FieldType();
type.setIndex(true);
type.setStored(true);
for(String string : authors){
  doc.add(new Field("author",string,type));      
  }

```

---

## 加权
Lucene中使用的评分算法是**TF-IDF**(词频-逆文档频率),一个关键字的评分跟它在本文档中出现的频率成正比,跟它在所有文档中出现的频率成反比

4.x的改动中只能对域进行加权

对**Field**加权
```
Field field = new Field("title","process",type);
field.setBoost(10.0f);
```

要想对文档加权,则必须对每个域进行加权
更加具体的评分机制和实现 [Lucene全文检索之评分](http://www.iteye.com/job/topic/1133219)

### 加权基准(Norms)

---
## 索引数字.日期和时间

### 数字
1. 数字内嵌在文本中,如
> "Be sure to include Form 1099 in your tax return"
如果想保留数字`1099`的搜索,可以使用`WhitespaceAnalyzer`和`StandardAnalyzer`,他们将`1099`提取出来并写入索引.

2. 某个域只包含数字,而希望将其进行精确匹配,使用`NumericddDocValuesField`类.
其子类有: *ByteDocValuesField*, *DoubleDocValuesField*, *FloatDocValuesField*, *IntDocValuesField*, *LongDocValuesField*, *PackedLongDocValuesField*, *ShortDocValuesField*
```
dco.add(new DoubleDocValueField("price",199.9));
```

### 日期和时间
将日期转换成`long`型,按数值类型保存
```
doc.add(new NumericDocValueField("timestamp",new Date(){}.getTime()));
```
或者从`Calendar`实例中获取日期
```
Calendar cal = Calendar.getInstance();
cal.setTime(new Date());
document.add(new NumericDocValuesField("dayOfMonth",cal.get(Calendar.DAY_OF_MONTH)));
```

---

## 域截取
Lucene4.x中 限制域长度的设置移到了`analysis`子模块中,参考`LimitTokenCountAnalyzer`

## 索引优化
indexWriter.forceMerge(int maxNumSegments)
将索引合并为最多max个段
优化期间会占用大量的空间,会产生大量开销
了解了索引存储细节后再回头来思考合并优化的原理

