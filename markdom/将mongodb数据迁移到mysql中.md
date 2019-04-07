# 将mongodb数据迁移到mysql中

## port导出.csv文件

`mongodb`自带`mongoexport`工具，可便捷导出`csv`、`json`等格式数据：

```
mongoexport -h 127.0.0.1 -u username -p password -d userInfoDB(数据库名称) -c regInfo(集合名称) -f _id,字段1,字段2 --type=csv -o /tmp/mongoStore/userInfo.csv(保存路径)1
```

根据个人需要选择要导出的字段，此处可以无须导出`_id`字段

## 2. 新建数据库和数据表

按照个人需要设计数据表结构。此处注意数据表的字段顺序必须一一对应于csv文件中首行的key，所以对应数据表可以暂时先**不用设置自增id**，否则数据导入后数据表字段对应的值会混乱。

## 3. 新建csv导入mysql的sql脚本

创建load_csv.sql文件

```
load data local infile '/tmp/mongoStore/userInfo.csv'(修改为指定csv文件路径)
into table `userInfo`(修改为mysql中新建数据表名称) character set utf8
fields terminated by ',' optionally enclosed by '"'
lines terminated by '\n'
ignore 1 lines;12345
```

执行以下`mysql load sql`命令

```
mysql -uroot -pmysql -DuserInfoDB --default-character-set=utf8 --local-infile=1 < ~/load_csv.sql1
```

这样数据就从迁移到了mysql

##### 如果mongodb中键比较多，可以通过如下方式获取keys

比较麻烦的点在于导出csv文件的时候要选择字段和新建mysql表的时候也要写全字段。 
如下方式可以比较快的获取字段列表： 
从`mongodb`获取一条数据

```
$ mongo
> use userInfoDB
> db.regInfo.find().limit(1)
{ "_id" : ObjectId("5ac3ac86af5b4e34af40xxxx"), "regAuthority" : "XX市XX区XX局", "entranceName" : 1, "have_data_flag" : 1, "orgNumber" : "091xxxx", "termStart" : "2014-02-12", "businessScope" : "咨询"}1234
```

以上数据复制到python解释器中，使用python命令获取所有key的列表 
（`_id`的值不符合dict格式，此处删掉）

```
>>> import json
>>> s = """
... {"regAuthority" : "XX市XX区XX局", "entranceName" : 1, "have_data_flag" : 1, "orgNumber" : "091xxxx", "termStart" : "2014-02-12", "businessScope" : "咨询"}"""
>>> s_dict = json.loads(s)
>>> s_dict.keys()
['have_data_flag', 'termStart', 'regAuthority', 'orgNumber', 'entranceName', 'businessScope']123456
```





python脚本实现Msql数据迁移到MongoDB



```
#coding: utf-8
import torndb,pymongo,time

# connect to mysql database
mysql = torndb.Connection(host='127.0.0.1', database='database', user='username', password='password')

#connect to mongodb and obtain total lines in mysql
mongo = pymongo.MongoClient('mongodb://ip').database
mongo.authenticate('username',password='password')
countlines = mysql.query('SELECT max(table_field) FROM table_name')
count = countlines[0]['max(table_field)']

#count = 300
print count

i = 0 
j = 100
start_time = time.time()
#select from mysql to insert mongodb by 100 lines.
for i in range(0,count,100):
    #print a,b
    #print i
    #print 'SELECT * FROM quiz_submission where quiz_submission_id > %d and  quiz_submission_id <= %d' %(i,j)
    submission = mysql.query('SELECT * FROM table_name where table_field > %d and  table_field <= %d' %(i,j))
    #print submission 
    if submission:
        #collection_name like mysql table_name
        mongo.collection_name.insert_many(submission)
    else:
        i +=100
        j +=100
        continue
    i +=100
    j +=100
end_time = time.time()
deltatime = end_time - start_time
totalhour = int(deltatime / 3600)
totalminute = int((deltatime - totalhour * 3600) / 60)
totalsecond = int(deltatime - totalhour * 3600 - totalminute * 60)
#print migrate data total time consuming.
print "Data Migrate Finished,Total Time Consuming: %d Hour %d Minute %d Seconds" %(totalhour,totalminute,totalsecond)
```

