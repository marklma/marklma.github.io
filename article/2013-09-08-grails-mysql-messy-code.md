
---
layout: post
title: grails和mysql乱码问题
date: '2013-09-08 22:50:02'
tags: ['Grails','Mysql','code']
category: groovy
description:
---
前几天试用grails+mysql想快速做一个页面，没想到最令人恶心的乱码又出现了，而且这次的乱码问题让我处理了半天都没处理出来，结果当我换回h2 database，发现中文正常才让我发现原来是mysql的编码问题导致的乱码。
简单的domain:
book.groovy
<!--more-->
```groovy
package tes
class Book {
    static constraints = {}
    String title
    String writer
}
```

然后用grails generate-all tes.Book 生成对应controller、View等内容。
修改conf/DataSource.groovy如下：
```groovy
dataSource {
    pooled = true
    driverClassName = "com.mysql.jdbc.Driver"
    username = "mark"
    password = "123"
}
hibernate {
    cache.use_second_level_cache = true
    cache.use_query_cache = true
    cache.region.factory_class = 'net.sf.ehcache.hibernate.EhCacheRegionFactory'
}
// environment specific settings
environments {
    development {
        dataSource {
            dbCreate = "update" // one of 'create', 'create-drop', 'update', 'validate', ''
            url = "jdbc:mysql://localhost:3306/bookdb?useUnicode=true&characterEncoding=UTF-8"
            }
        }
    }
    test {
        dataSource {
            dbCreate = "update"
            url = "jdbc:mysql://localhost:3306/bookdb?useUnicode=true&characterEncoding=UTF-8"
        }
    }
    production {
        dataSource {
            dbCreate = "update"
            //url = "jdbc:h2:prodDb;MVCC=TRUE;LOCK_TIMEOUT=10000"
        url = "jdbc:mysql://localhost:3309/bookdb?useUnicode=true&characterEncoding=UTF-8"
            }
        }
    }
}
```

然后进入mysql建立database bookdb并授权给mark@loacalhost，脚本如下：
```
create database bookdb;
```

运行grails run-app，进入http://localhost:8080/tes/book/save输入[title:"123",writer"321"]这类不含中文字符的内容没有任何异常。当开发快完毕后我想起测试一下中文愕然发现出错了！
```
Error 500: Internal Server Error

URI
    /tes/book/save
Class
    java.sql.SQLException
Message
    Incorrect string value: '\xE4\xB8\xAD\xE6\x96\x87' for column 'title' at row 1

Around line 24 of grails-app\controllers\tes\BookController.groovy

21:22:    def save() {23:        def bookInstance = new Book(params)24:        if (!bookInstance.save(flush: true)) {25:            render(view: "create", model: [bookInstance: bookInstance])26:            return27:        }
```

由于在my.ini我设置的是gbk，配置如下：
```
//……
[mysqld]
default-character-set=gbk
//……
[mysql]
default-character-set=gbk
//……
```

我将其改为utf8也仍旧不行，在BookController.groovy的save方法加入pringln查看错误信息，结果发现是乱码（这就是误导我一直以为是grails传参问题导致），于是在网上搜索方法，将request.setCharacterEncoding("utf-8");  response.setContentType("text/html;charset=utf-8");这类代码都写了个遍，结果发现还是未能解决，于是只能用h2 database暂时解决问题。当使用h2 database发现不存在中文问题我才确信原来是mysql乱码问题（之前也不敢相信java.sql.SQLException这个错误信息导致走了弯路）。
经过测试，果然就是乱码问题导致，我的mysql默认建立数据库是latin1的编码，所以无法接受中文字符。正确建立数据库应该是：
```sql
CREATE DATABASE bookdb DEFAULT CHARACTER SET utf8;
```

由于我在conf/DataSource.groovy中数据库的配置我也加上了characterEncoding=UTF-8，默认建立的表格也就默认是utf8编码的了。而且实际和my.ini中配置无关。

### 后记：
弯路也不白绕，接触了一下H2 DATABASE，发现确实挺好用，jdbc url配置如下：
```
jdbc:h2:file:d:/devDb2;MVCC=TRUE;LOCK_TIMEOUT=10000;FILE_LOCK=NO
```

其中FILE_LOCK=NO可以不独占数据库，让dbeaver这类数据库查看软件可以实时查看数据库中信息。