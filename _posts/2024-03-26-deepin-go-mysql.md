---
layout:     post
title:      深入探究MySQL的复制原理
subtitle:   由canal源码学习展开
date:       2024-03-26
author:     Lyonzhi
header-img: img/in-post/2024-03-26/deepin-go-mysql-bg.jpg
catalog: true
tags:
    - golang
    - go-mysql
    - mysql
    - mysql-replication
---


## 0. 前言

由于工作的关系，我需要大量的使用[go-mysql](https://github.com/go-mysql-org/go-mysql)的代码中关于复制的一部分。

作为一个曾经的DBA，也刚好对这部分内容感兴趣，所以就深入的研究了一下这部分的代码，现在将学习的笔记整理在此。

如果可能的话，顺手就连[alibaba canal](https://github.com/alibaba/canal)一起学了。

## 1. Binlog各类报文

首先考虑这样一个问题：我们要如何得到MySQL的Binlog，像Slave一样？

回答这个问题之前，先回忆一下如何将一个slave加入到集群里：

- 启动一个MySQL实例，执行`change master`语句，指定master的基本信息和binlog的基本信息；
- 执行`start slave`命令，开始复制。

这些操作其实就是一系列的网络请求，只需要了解其背后的报文格式，就可以实现。

先说说`change master`做了什么：发送了一个`COM_REGISTER_SLAVE`报文给master机器，这个报文的编码是`0x15`，也就是10进制的21。

这样就完成了注册操作，但是此时还不能开始dump binlog。接下来会给master发送一个`COM_BINLOG_DUMP`报文，这个报文的编码是`0x12`，也就是10进制的18。

大部分情况下，我们只需要关注这两种报文就可以启动一个进程来源源不断的获取binlog了。

### 1.1 COM\_REGISTER\_SLAVE

先来看看一个典型的`change master`语句：

```sql
change master to master_host='localhost', MASTER_PORT=3306, master_user='repl', master_password='repl', master_log_file='mysql-bin.000001', master_log_pos=0;
```

也就是说，在发送`COM_REGISTER_SLAVE`报文的时候，还需要携带一些关键信息上去。

看一下这个报文的结构。

| 字节数 | 说明 |
|:----|:----|
| 1 | 0x15 |
| 4 | serverId |
| 1 | hostname长度 |
| n | hostname |
| 1 | user name长度 |
| n | user name |
| 1 | password长度 |
| n | password |
| 2 | port | ​ |
| 4 | replication rank，忽略 |
| 4 | master id，给0 |


但是，这里需要注意，go-mysql里这个报文不是从位置0开始的，而是从4开始：

```go
pos := 4

data[pos] = COM_REGISTER_SLAVE
pos++
```

对于这一点，我是比较疑惑的，后来翻看了一下alibaba canal的相同代码，没有发现前4位有什么特殊的地方：

![请在此添加图片描述](../../../../img/in-post/2024-03-26/img-1.png)

不过后来我发现MariaDB里有这样一个文档：

[https://mariadb.com/kb/en/com\_register\_slave/](https://mariadb.com/kb/en/com_register_slave/)

![请在此添加图片描述](../../../../img/in-post/2024-03-26/img-2.png)

也就是说，前4位可能是一个类似于魔法数一样的东西。

这个问题留待以后再去探求。

### 1.2 COM\_BINLOG\_DUMP

完成注册以后，就可以开始DUMP binlog数据了。

这个报文有官方文档可以看，还是比较友好的：

![请在此添加图片描述](../../../../img/in-post/2024-03-26/img-3.png)

这个报文基本都是定长数据，只有filename这一个变长数据。

但是，go-mysql里的cmd也是从位置4开始写的。还是从MariaDB处得到答案：

![请在此添加图片描述](../../../../img/in-post/2024-03-26/img-4.png)

可见，前面4位确实是一类类似魔数的东西。

这个答案，在调试代码的还是发现了，其实这4位就是预留了Header的位置。

那么这个Header有什么呢？还是在alibaba的canal里找到了答案：

![请在此添加图片描述](../../../../img/in-post/2024-03-26/img-5.png)

也就是前三位保存body的长度（小端序），第4位保存一个序列号，这个序列号在拼body的数据时候就已经置为0了，在alibaba canal里也是一样的处理方式：

![请在此添加图片描述](../../../../img/in-post/2024-03-26/img-6.png)

### 1.3 COM\_BINLOG\_DUMP\_GTID

这个报文和之前的`COM_BINLOG_DUMP`有类似之处，都是启动binlog的dump操作，不同的是，这个报文是专门为了gtid而设计的。

比如这个命令：

```sql
change master to master_host='192.110.103.41',master_port=3106,master_auto_position=1; 
```

结构说明如下：

| 字节数 | 说明 |
|:----|:----|
| 1 | 0xef |
| 2 | 0x04 |
| 4 | server-id |
| 4 | binlog filename len，可以写成0，后面的file name就直接忽略掉，跳过不传即可 |
| n | file name |
| 8 | Pos，可以写死成4 |
| 4 | gtid data len |
| n | gtid data |


- go-mysql和alibaba canal对file name的处理是不同的，alibaba直接忽略了filename，并给file name length设置成了0。
- go-mysql和alibaba canal对pos的处理也不同，alibaba给了4。

实际上，从节约的角度来看问题，alibaba的处理更为优秀，在autoposition的设置下，fileName和Pos是无关紧要的东西。

## 2. 各类常见事件

go-mysql的注释写的过于简略，并不利于理解代码。Golang本身对程序内文档的支持不如Java那样，所以有的时候还是需要参考alibaba canal的代码。

事件类型枚举如下，这里还有许多MariaDB的事件类型，全部摘录在此以备查找：

```java
    public static final int    UNKNOWN_EVENT                            = 0;
    public static final int    START_EVENT_V3                           = 1;
    public static final int    QUERY_EVENT                              = 2;
    public static final int    STOP_EVENT                               = 3;
    public static final int    ROTATE_EVENT                             = 4;
    public static final int    INTVAR_EVENT                             = 5;
    public static final int    LOAD_EVENT                               = 6;
    public static final int    SLAVE_EVENT                              = 7;
    public static final int    CREATE_FILE_EVENT                        = 8;
    public static final int    APPEND_BLOCK_EVENT                       = 9;
    public static final int    EXEC_LOAD_EVENT                          = 10;
    public static final int    DELETE_FILE_EVENT                        = 11;

    /**
     * NEW_LOAD_EVENT is like LOAD_EVENT except that it has a longer sql_ex,
     * allowing multibyte TERMINATED BY etc; both types share the same class
     * (Load_log_event)
     */
    public static final int    NEW_LOAD_EVENT                           = 12;
    public static final int    RAND_EVENT                               = 13;
    public static final int    USER_VAR_EVENT                           = 14;
    public static final int    FORMAT_DESCRIPTION_EVENT                 = 15;
    public static final int    XID_EVENT                                = 16;
    public static final int    BEGIN_LOAD_QUERY_EVENT                   = 17;
    public static final int    EXECUTE_LOAD_QUERY_EVENT                 = 18;

    public static final int    TABLE_MAP_EVENT                          = 19;

    /**
     * These event numbers were used for 5.1.0 to 5.1.15 and are therefore
     * obsolete.
     */
    public static final int    PRE_GA_WRITE_ROWS_EVENT                  = 20;
    public static final int    PRE_GA_UPDATE_ROWS_EVENT                 = 21;
    public static final int    PRE_GA_DELETE_ROWS_EVENT                 = 22;

    /**
     * These event numbers are used from 5.1.16 and forward
     */
    public static final int    WRITE_ROWS_EVENT_V1                      = 23;
    public static final int    UPDATE_ROWS_EVENT_V1                     = 24;
    public static final int    DELETE_ROWS_EVENT_V1                     = 25;

    /**
     * Something out of the ordinary happened on the master
     */
    public static final int    INCIDENT_EVENT                           = 26;

    /**
     * Heartbeat event to be send by master at its idle time to ensure master's
     * online status to slave
     */
    public static final int    HEARTBEAT_LOG_EVENT                      = 27;

    /**
     * In some situations, it is necessary to send over ignorable data to the
     * slave: data that a slave can handle in case there is code for handling
     * it, but which can be ignored if it is not recognized.
     */
    public static final int    IGNORABLE_LOG_EVENT                      = 28;
    public static final int    ROWS_QUERY_LOG_EVENT                     = 29;

    /** Version 2 of the Row events */
    public static final int    WRITE_ROWS_EVENT                         = 30;
    public static final int    UPDATE_ROWS_EVENT                        = 31;
    public static final int    DELETE_ROWS_EVENT                        = 32;

    public static final int    GTID_LOG_EVENT                           = 33;
    public static final int    ANONYMOUS_GTID_LOG_EVENT                 = 34;

    public static final int    PREVIOUS_GTIDS_LOG_EVENT                 = 35;

    /* MySQL 5.7 events */
    public static final int    TRANSACTION_CONTEXT_EVENT                = 36;

    public static final int    VIEW_CHANGE_EVENT                        = 37;

    /* Prepared XA transaction terminal event similar to Xid */
    public static final int    XA_PREPARE_LOG_EVENT                     = 38;

    /**
     * Extension of UPDATE_ROWS_EVENT, allowing partial values according to
     * binlog_row_value_options.
     */
    public static final int    PARTIAL_UPDATE_ROWS_EVENT                = 39;

    /* mysql 8.0.20 */
    public static final int    TRANSACTION_PAYLOAD_EVENT                = 40;

    /* mysql 8.0.26 */
    public static final int    HEARTBEAT_LOG_EVENT_V2                   = 41;

    public static final int    MYSQL_ENUM_END_EVENT                     = 42;

    // mariaDb 5.5.34
    /* New MySQL/Sun events are to be added right above this comment */
    public static final int    MYSQL_EVENTS_END                         = 49;

    public static final int    MARIA_EVENTS_BEGIN                       = 160;
    /* New Maria event numbers start from here */
    public static final int    ANNOTATE_ROWS_EVENT                      = 160;
    /*
     * Binlog checkpoint event. Used for XA crash recovery on the master, not
     * used in replication. A binlog checkpoint event specifies a binlog file
     * such that XA crash recovery can start from that file - and it is
     * guaranteed to find all XIDs that are prepared in storage engines but not
     * yet committed.
     */
    public static final int    BINLOG_CHECKPOINT_EVENT                  = 161;
    /*
     * Gtid event. For global transaction ID, used to start a new event group,
     * instead of the old BEGIN query event, and also to mark stand-alone
     * events.
     */
    public static final int    GTID_EVENT                               = 162;
    /*
     * Gtid list event. Logged at the start of every binlog, to record the
     * current replication state. This consists of the last GTID seen for each
     * replication domain.
     */
    public static final int    GTID_LIST_EVENT                          = 163;

    public static final int    START_ENCRYPTION_EVENT                   = 164;

    // mariadb 10.10.1
    /*
     * Compressed binlog event. Note that the order between WRITE/UPDATE/DELETE
     * events is significant; this is so that we can convert from the compressed to
     * the uncompressed event type with (type-WRITE_ROWS_COMPRESSED_EVENT +
     * WRITE_ROWS_EVENT) and similar for _V1.
     */
    public static final int    QUERY_COMPRESSED_EVENT                   = 165;
    public static final int    WRITE_ROWS_COMPRESSED_EVENT_V1           = 166;
    public static final int    UPDATE_ROWS_COMPRESSED_EVENT_V1          = 167;
    public static final int    DELETE_ROWS_COMPRESSED_EVENT_V1          = 168;
    public static final int    WRITE_ROWS_COMPRESSED_EVENT              = 169;
    public static final int    UPDATE_ROWS_COMPRESSED_EVENT             = 170;
    public static final int    DELETE_ROWS_COMPRESSED_EVENT             = 171;

    /** end marker */
    public static final int    ENUM_END_EVENT                           = 171;
```

## 3. 事件的读取和解析

### 3.1 Common Header

每一个事件都是由Header和Event Data组成。其中Header是固定的19个字节。

|name|format|description|
|--|--|--|
|timestamp|4 bytes unsigned integer|query发生的时间戳|
|type|1 byte enumeration|事件类型|
|server_id|4 bytes unsigned integer|产生event的server id|
|total_size|4 bytes unsigned integer|是event size + common-header size + post-header size|
|master_position|4 byte unsigned integer|下一个event的位置|
|flags|2 byte bitfield||

可以看到common header中记录了许多重要的信息。对于event的解析提供了很多关键的参数。在理解如何解析事件之前，先来研究一下网络包是如何解析的。

### 3.2 网络包的解析

我们抛开所有连接建立的过程直接分析包的读取。

在写代码的时候我就有几个疑问：

- 程序是如何知道event的开始和结束的？
- 如果一个event由多个包组成如何去识别？

带着这些疑问，我翻阅了go-mysql和alibaba canal的代码，其代码位置列在下面：

- [go-mysql](https://github.com/go-mysql-org/go-mysql/blob/master/packet/conn.go#L195)
- [canal](https://github.com/alibaba/canal/blob/master/dbsync/src/main/java/com/taobao/tddl/dbsync/binlog/DirectLogFetcher.java#L279)

两者比较可以发现packet的前4位似乎很有意思。看一看比较好理解的go代码片段：

```go
if _, err := io.ReadFull(r, c.header[:4]); err != nil {
    return errors.Wrapf(ErrBadConn, "io.ReadFull(header) failed. err %v", err)
}

length := int(uint32(c.header[0]) | uint32(c.header[1])<<8 | uint32(c.header[2])<<16)
sequence := c.header[3]

if sequence != c.Sequence {
    return errors.Errorf("invalid sequence %d != %d", sequence, c.Sequence)
}
```

可以看出这些端倪：

- 前3位是一种长度记录，就是event的总大小+1，比如第一个Rotate事件，大小就是41
- 第4位是序列号，至于这个序列号是做什么的，现在还不清楚，留一个问题
- 我们可以把这4位叫做包头部分

现在回头来看看上面记录的事件长度41，这个长度应该包括了：1字节的包类型标记、40字节的事件长度。

这1字节的包类型标记枚举如下：

|packet|编码|
|--|--|
|OK_HEADER|0x00|
|ERR_HEADER|0xFF|
|EOF_HEADER|0xFE|

可以参考MySQL自己的[开发文档](https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_ok_packet.html)

这里提及了自从5.7.5之后，EOF_PACKET就标记为过时了。

知道了上述细节以后，现在看一个简单的buf片段，来分析一下：

```
41,0,0,1, 0, 0,0,0,0,4
```

- 前4位能计算出length=41和seq=1；
- 第5位表示这是一个OK_PACKET
- 0,0,0,0,4就是event最开始的5位，表示timestamp=0和type=4，也就是ROTATE_EVENT

了解了如何去分析一个binlog event的网络包之后，就可以开始分析binlog event了。