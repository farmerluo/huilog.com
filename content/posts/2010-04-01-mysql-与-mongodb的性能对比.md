---
title: mysql 与 mongodb的性能对比
author: 阿辉
date: 2010-04-01T13:16:00+00:00
categories:
- Mongodb
tags:
- Mongodb
keywords:
- Mongodb
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
最近写了个java的程序来测试mysql 与mongodb的性能，代码如下：
```java
package mongotest;
import com.mongodb.Mongo;
import com.mongodb.DB;
import com.mongodb.DBCollection;
import com.mongodb.BasicDBObject;
import com.mongodb.DBObject;
import com.mongodb.MongoException;
import java.net.UnknownHostException;
import java.sql.*;

/**
*
* @author FarmerLuo
*/
public class Main {

    /**
    * @param args the command line arguments
    */
    public static void main(String[] args) {
        // TODO code application logic here
        int rows = 0;
        long start = System.currentTimeMillis();

    //        try { Thread.sleep ( 3000 );
    //        } catch (InterruptedException ie){}

        if ( args.length < 2 ) {
            System.out.print("Lack parameter!!!n");
            System.exit(1);
        }

        if ( !args[0].equals("mysql") && !args[0].equals("mongo") ) {
            System.out.print("First parameter: mysql or mongon");
            System.exit(1);
        }

        if ( !args[1].equals("insert") && !args[1].equals("update") ) {
            System.out.print("Second parameter: insert or updaten");
            System.exit(2);
        }

        if ( args.length > 2 ) {
            rows = Integer.parseInt(args[2]);
        } else {
            rows = 10000;
        }

        if ( args[0].equals("mysql") ) {
            if ( args[1].equals("insert") ) {
                mysql_insert(rows);
            } else {
                mysql_update(rows);
            }
        } else {
            if ( args[1].equals("insert") ) {
                mongo_insert(rows);
            } else {
                mongo_update(rows);
            }
        }

        long stop = System.currentTimeMillis();
        long endtime = (stop - start)/1000;
        if ( endtime == 0 ) endtime = 1;
        long result = rows/endtime;

        System.out.print("Total run time:" + endtime + " secn");
        System.out.print("Total rows:" + rows + "n");
        System.out.print(args[0] + " " + args[1] + " Result:" + result + "row/secn"); 
    }

    public static void mysql_insert(long len) {

        Connection conn = null;
        try {
            conn = DriverManager.getConnection("jdbc:mysql://192.168.20.24/test?user=root&password=1qaz2wsx&characterEncoding=UTF8");
            // Do something with the Connection
        } catch (SQLException ex) {
            // handle any errors
            System.out.println("SQLException: " + ex.getMessage());
            System.out.println("SQLState: " + ex.getSQLState());
            System.out.println("VendorError: " + ex.getErrorCode());
        }
        Statement stmt = null;
        try {
            stmt = conn.createStatement();
        } catch (SQLException ex) {
            System.out.println("SQLException: " + ex.getMessage());
            System.out.println("SQLState: " + ex.getSQLState());
            System.out.println("VendorError: " + ex.getErrorCode());
        }
        String str = "1234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456";

        //System.out.print(sql);
        for( int j = 0; j < len; j++ ){
            String sql = "insert into test (count, test1, test2, test3, test4) values (" + j + ",'" + str + "','" + str + "','" + str + "','" + str + "')";
            try {
                stmt.executeUpdate(sql);
            } catch (SQLException ex) {
                System.out.println("SQLException: " + ex.getMessage());
                System.out.println("SQLState: " + ex.getSQLState());
                System.out.println("VendorError: " + ex.getErrorCode());
            }
        }
    }


    public static void mysql_update(long len) {

        Connection conn = null;
        try {
            conn = DriverManager.getConnection("jdbc:mysql://192.168.20.24/test?user=root&password=1qaz2wsx&characterEncoding=UTF8");
            // Do something with the Connection
        } catch (SQLException ex) {
            // handle any errors
            System.out.println("SQLException: " + ex.getMessage());
            System.out.println("SQLState: " + ex.getSQLState());
            System.out.println("VendorError: " + ex.getErrorCode());
        }
        Statement stmt = null;
        try {
            stmt = conn.createStatement();
        } catch (SQLException ex) {
            System.out.println("SQLException: " + ex.getMessage());
            System.out.println("SQLState: " + ex.getSQLState());
            System.out.println("VendorError: " + ex.getErrorCode());
        }
        String str = "UPDATE7890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456";

        //System.out.print(sql);
        for( int j = 0; j < len; j++ ){
            String sql = "update test set test1 = '" + str + "',test2 = '" + str + "' , test3 = '" + str + "', test4 = '" + str + "' where id = "+ j;
            try {
                stmt.executeUpdate(sql);
            } catch (SQLException ex) {
                System.out.println("SQLException: " + ex.getMessage());
                System.out.println("SQLState: " + ex.getSQLState());
                System.out.println("VendorError: " + ex.getErrorCode());
            }
        }
    }

    public static void mongo_insert(long len){
        Mongo m = null;
        try {
            m = new Mongo("192.168.20.24", 27017);
        } catch (UnknownHostException ex) {
            System.out.println("UnknownHostException:" + ex.getMessage());
        } catch (MongoException ex) {
            System.out.println("Mongo Exception:" + ex.getMessage());
            System.out.println("Mongo error code:" + ex.getCode());
        }

        DB db = m.getDB( "dbname6" );

        DBCollection dbcoll = db.getCollection("test");
        String str = "1234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456";
        for (int j = 0; j < len; j++) {
            DBObject dblist = new BasicDBObject();
            dblist.put("_id", j);
            dblist.put("count", j);
            dblist.put("test1", str);
            dblist.put("test2", str);
            dblist.put("test3", str);
            dblist.put("test4", str);
            try {
                dbcoll.insert(dblist);
            } catch (MongoException ex) {
                System.out.println("Mongo Exception:" + ex.getMessage());
                System.out.println("Mongo error code:" + ex.getCode());
            }
        }
    }

    public static void mongo_update(long len){
        Mongo m = null;
        try {
            m = new Mongo("192.168.20.24", 27017);
        } catch (UnknownHostException ex) {
            System.out.println("UnknownHostException:" + ex.getMessage());
        } catch (MongoException ex) {
            System.out.println("Mongo Exception:" + ex.getMessage());
            System.out.println("Mongo error code:" + ex.getCode());
        }

        DB db = m.getDB( "dbname6" );

        DBCollection dbcoll = db.getCollection("test");
        String str = "UPDATE7890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456";
        for (int j = 0; j < len; j++) {
            DBObject dblist = new BasicDBObject();
            DBObject qlist = new BasicDBObject();
            qlist.put("_id", j);
            dblist.put("test1", str);
            dblist.put("test2", str);
            dblist.put("test3", str);
            dblist.put("test4", str);
            try {
                dbcoll.update(qlist,dblist);
            } catch (MongoException ex) {
                System.out.println("Mongo Exception:" + ex.getMessage());
                System.out.println("Mongo error code:" + ex.getCode());
            }
        }
    }
}
```
<!--more-->
mysql为innodb,数据库表结构如下：
```sql
CREATE TABLE IF NOT EXISTS `test` (
`id` int(11) NOT NULL auto_increment,
`count` int(11) NOT NULL default '0',
`test1` varchar(256) NOT NULL,
`test2` varchar(256) NOT NULL,
`test3` varchar(256) NOT NULL,
`test4` varchar(256) NOT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ;
```
测试是在两台虚拟机之间进行，两台虚拟机在同一台exsi的物理机上，所以中间的网络延时基本为0。每行数据大小为1k。

测试结果：

1) mysql 插入50万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mysql insert 500000
Total run time:545 sec
Total rows:500000
mysql insert Result:917row/sec
```
资源占用：
```
Cpu(s): 7.4%us, 4.7%sy, 0.0%ni, 11.7%id, 72.6%wa, 0.7%hi, 3.0%si, 0.0%st   load average为1以下
```

2）mysql更新50万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mysql update 500000
Total run time:838 sec
Total rows:500000
mysql update Result:596row/sec
```
资源占用：
```
Cpu(s): 7.4%us, 3.3%sy, 0.0%ni, 6.7%id, 80.6%wa, 0.7%hi, 1.3%si, 0.0%st     load average为1以下
```

3) mongo插入50万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mongo insert 500000    
Total run time:52 sec
Total rows:500000
mongo insert Result:9615row/sec
```
把数据清空，第二次跑测试程序，速度要慢一些。
```bash
[root@web dist]# java -jar "mongotest.jar" mongo insert 500000
Total run time:60 sec
Total rows:500000
mongo insert Result:8333row/sec
```
资源占用：
```bash
Cpu(s): 6.0%us, 3.0%sy, 0.0%ni, 0.0%id, 88.6%wa, 0.3%hi, 2.0%si, 0.0%st load average为1.5~2.0
```

4) mongo更新50万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mongo update 500000
Total run time:32 sec
Total rows:500000
mongo update Result:15625row/sec
```
资源占用：
```
Cpu(s): 90.0%us, 7.4%sy, 0.0%ni, 0.0%id, 0.0%wa, 0.0%hi, 2.7%si, 0.0%st     load average为1以下
```

5) mysql 插入100万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mysql insert 1000000
Total run time:1085 sec
Total rows:1000000
mysql insert Result:921row/sec
```
资源占用和50万差不多。

6) mysql 更新100万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mysql update 1000000
Total run time:1646 sec
Total rows:1000000
mysql update Result:607row/sec
```
资源占用和50万差不多。

7) mongo插入100万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mongo insert 1000000   
Total run time:120 sec
Total rows:1000000
mongo insert Result:8333row/sec
```
资源占用明显增高了很多，系统基本不能操作了。
```bash
top - 14:52:10 up 2 days, 5:00, 1 user, load average: 3.84, 1.41, 0.98
Tasks: 78 total,   2 running, 76 sleeping,   0 stopped,   0 zombie
Cpu(s): 6.4%us, 26.2%sy, 0.0%ni, 0.0%id, 63.6%wa, 0.0%hi, 3.7%si, 0.0%st
Mem:   2059620k total, 2049616k used,    10004k free,     1692k buffers
Swap: 1048568k total,   988112k used,    60456k free, 1380512k cached

PID USER      PR NI VIRT RES SHR S %CPU %MEM    TIME+ COMMAND                                                                
2817 root      16   0 2328m 614m 612m S 12.5 30.5   8:01.53 mongod   
```

8) mongo更新100万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mongo update 1000000
Total run time:66 sec
Total rows:1000000
mongo update Result:15151row/sec
```

资源占用:
```bash
top - 14:58:14 up 2 days, 5:06, 1 user, load average: 0.53, 0.99, 0.97
Tasks: 75 total,   2 running, 73 sleeping,   0 stopped,   0 zombie
Cpu(s): 89.7%us, 6.7%sy, 0.0%ni, 0.0%id, 0.0%wa, 0.3%hi, 3.3%si, 0.0%st
Mem:   2059620k total, 2048632k used,    10988k free,     2468k buffers
Swap: 1048568k total, 1046996k used,     1572k free, 1578768k cached

PID USER      PR NI VIRT RES SHR S %CPU %MEM    TIME+ COMMAND                                                                
2817 root      16   0 3352m 1.2g 1.2g S 98.6 62.1   9:46.79 mongod   
```

9) mongo插入1000万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mongo insert 10000000
Total run time:774 sec
Total rows:10000000
mongo insert Result:12919row/sec
```
资源占用:
```bash
top - 15:14:17 up 2 days, 5:23, 1 user, load average: 4.56, 4.04, 2.64
Tasks: 75 total,   2 running, 73 sleeping,   0 stopped,   0 zombie
Cpu(s): 28.5%us, 9.3%sy, 0.0%ni, 0.0%id, 57.0%wa, 1.0%hi, 4.3%si, 0.0%st
Mem:   2059620k total, 2049132k used,    10488k free,     1580k buffers
Swap: 1048568k total, 1047520k used,     1048k free, 1556816k cached

PID USER      PR NI VIRT RES SHR S %CPU %MEM    TIME+ COMMAND                                                                
2817 root      16   0 13.3g 1.5g 1.5g S 35.3 75.3 14:31.29 mongod
```

10) mongo更新1000万行数据
```bash
[root@web dist]# java -jar "mongotest.jar" mongo update 10000000
Total run time:830 sec
Total rows:10000000
mongo update Result:12048row/sec
```
资源占用:
```bash
top - 15:22:45 up 2 days, 5:31, 1 user, load average: 1.31, 1.69, 1.99
Tasks: 75 total,   2 running, 73 sleeping,   0 stopped,   0 zombie
Cpu(s): 48.7%us, 5.0%sy, 0.0%ni, 0.0%id, 42.3%wa, 0.7%hi, 3.4%si, 0.0%st
Mem:   2059620k total, 2049772k used,     9848k free,     1336k buffers
Swap: 1048568k total, 1044068k used,     4500k free, 1559972k cached

PID USER      PR NI VIRT RES SHR S %CPU %MEM    TIME+ COMMAND                                                                
2817 root      16   0 13.3g 1.5g 1.5g S 54.3 75.6 16:52.04 mongod
```