title: Lucene学习笔记(一)
date: 2014-12-06 20:03:51
tags: search
categories: java
---
Lucene学习笔记(一)

文件系统实例

参考原文来自:[实战Lucene](https://www.ibm.com/developerworks/cn/java/j-lo-lucene1/)
<!--more-->
## Lucene是什么

Lucene是一个基于java的全文信息检索**工具包**,他以jar包的形式为开发者提供**索引**和**搜索**的功能.

## 索引

### 建立索引
五个基础类: **Document**,**Field**,**IndexWriter**,**Analyzer**,**Directory**

* **Document**
用来描述文档,文档的形式可以是*HTML*页面,电子邮件或者文本文件.一个**Document**对象由多个**Field**对象组成.

* **Field**
用来描述文档的某个属性,例如一封电子邮件的标题和内容

* **Analyzer**
用于对文档内容进行 *分词* 处理.

* **IndexWriter**
将**Document**对象加入到索引中

* **Directory**
该类代表索引的存储位置,两个实现类分别是**FSDirectory**(文件系统中的索引的位置),**RAMDirectory**(内存中的索引的位置)


实例1.对文本文件建立索引

    Directory indexDir = FSDirectory.open(
                    new File("/home/yj/code/java/lucene/indexDir"));

    File dataDir = new File("/home/yj/code/java/lucene/dataDir");

    Analyzer luceneAnalyzer = new StandardAnalyzer();

    File[] dataFiles = dataDir.listFiles();

    IndexWriterConfig config = new IndexWriterConfig(Version.LUCENE_4_10_2,luceneAnalyzer);

    IndexWriter indexWriter = new IndexWriter(indexDir,config);

    for(int i = 0; i < dataFiles.length; i++){
      if(dataFiles[i].isFile() && dataFiles[i].getName().endsWith(".txt")){
        System.out.println("indexing file:" + dataFiles[i].getCanonicalPath());
        Document document = new Document();
        Reader textReader = new FileReader(dataFiles[i]);
        document.add(new TextField("path",dataFiles[i].getCanonicalPath(), Field.Store.YES));
        document.add(new TextField("content",textReader));
        document.add(new TextField("name",dataFiles[i].getName(), Field.Store.YES));
        indexWriter.addDocument(document);
      }
    }
    indexWriter.commit();

本实例基于Lucene4.10 版本升级,api改动很大,参考[最新的api文档](http://lucene.apache.org/core/4_10_2/core/overview-summary.html#overview_description)

总体过程是:读取要建立索引的文件目录下的文件,将有用的属性保存在**Field**中,生成**Document**对象,并用**IndexWriter**去记录每个**Document**对象中的信息,在指定的索引存储目录*indexDir*下保存索引.

### 搜索文档
基础类:**Term**,**Query**,**TermQuery**,**Hits**

**Query**
抽象类,实现类包括:**TermQuery**,**BooleanQuery**,**PrefixQuery**.将输入的查询字符串封装成Lucene能识别的Query.

**Term**
搜索的基本单位.

    Term term = new Term("fieldName","queryWord");
        
两个参数分别是:在文档的哪个*Field*上查找和要查找的关键字

**TermQuery**
Lucene支持的最基本的查询类

    TermQuery termQuery = new TermQuery(term);
                
接受一个 `Term` 对象作为参数

**IndexSearcher**
用来在建立好的索引上进行搜索

**Hits**
用来保存搜索结果

**TopDocs**
Represents hits returned by ``IndexSearcher.search(Query,Filter,int)`` and ``IndexSearcher.search(Query,int)``.

实例2.查询

    File indexDir = new File("/home/yj/code/java/lucene/indexDir");
      Query query = new TermQuery(new Term("content","foo"));
        try {
          FSDirectory fsDirectory = FSDirectory.open(indexDir);
          DirectoryReader dr = DirectoryReader.open(fsDirectory);
          IndexSearcher indexSearcher = new IndexSearcher(dr);
          TopDocs td = indexSearcher.search(query,100);
          ScoreDoc[] sds = td.scoreDocs;
          for(ScoreDoc sd : sds){
          Document document = indexSearcher.doc(sd.doc);
          System.out.println(document.get("path"));
         }
       } catch (IOException e) {
         e.printStackTrace();
       }

**TermQuery**对象由**Term**构造,搜索结果由**TopDoc**对象保持.
以上实例基于文件系统的文件索引和搜索.
