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

MySQL的复制技术是一个非常好用的技术，很容易就能搭建一个集群出来，再配合各种中间件，可以方便的搭建出一个读写分离负载均衡的集群。

MySQL的复制技术也经历了多年的发展，据我自己的工作经验来看，使用异步复制技术和半同步复制技术就能满足大部分系统的实际需求。

下面的图是一个到处都能搜到的图，来自MySQL官方，很好的说明了各类复制技术的基本原理：

![异步复制](../../../../img/in-post/2024-03-26/async-replication-diagram.png)

![半同步复制](../../../../img/in-post/2024-03-26/semisync-replication-diagram.png)

要理解MySQL的复制技术，首先就要去探究binlog技术。binlog是复制的基础，而且是和引擎没有关系的，这是Server层的日志。

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

### 3.3 ROTATE_EVENT事件的解析

先看一个典型的binlog文件（binlog.000050）最后一个事件记录，它一定是这样的一种形式：

| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info|
|--|--|--|--|--|--|
|binlog.000050|708|Rotate|1|752|binlog.000051;pos=4|

这个事件指明了下一个事件的文件名和起始位置。这也是我们能解析到的第一个事件，一段时间以来我一直以为这是binlog记录的第一个事件。

除去common header部分不看，那么这个事件可以整理如下表：

|byte|description|
|--|--|
|8|下一个binlog文件第一个位置，一般都是4，我也没见过别的值|
|n|下一个binlog的文件名，长度不定|

这个事件很简单，没什么多余需要说的。写一段伪代码去解析这个event：

```go
// 截取掉common header部分
event := data[19:]
// 读取8个字节的position
position := littleEndian.Uint64(event[0:])
// index=8之后就是fileName部分
fileName := event[8:]
```

这样就完成了事件的解析。接下来就可以继续解析下面的事件了。

### 3.4 FORMAT_DESCRIPTION_EVENT事件的解析

事件编码：15

这个事件就一定是binlog的第一个事件了：

| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info|
|--|--|--|--|--|--|
| binlog.000050 |   4 | Format_desc    |         1 |         126 | Server ver: 8.0.32, Binlog ver: 4                                   |

这里能看到的信息似乎不多，就是一些基本的信息，比如Server的版本、binlog的版本。

现在解析一下binlog文件来看看这个事件都有哪些信息：

```
# at 4
#240329 19:20:25 server id 1  end_log_pos 126 CRC32 0x6364c980 	Start: binlog v 4, server v 8.0.32 created 240329 19:20:25
# Warning: this binlog is either in use or was not closed properly.
```

这个事件看起来平平无奇，不过是很重要的一个事件，因为这里记录了如何decode后面的binlog事件的相关信息。

在解析这个事件之前，先来看看MySQL都提供了哪些校验方法：

- BINLOG_CHECKSUM_ALG_OFF: 0x00
- BINLOG_CHECKSUM_ALG_CRC32: 0x01
- BINLOG_CHECKSUM_ALG_UNDEF: 0xFF

先用`select @@global.binlog_checksum`命令查询一下，我的MySQL用什么算法去做binlog的checksum，一般结果都是CRC32。

这个事件整理如下表：

|byte|description|
|--|--|
|2|Version，就是binlog里看到的4|
|50|server version，也就是8.0.32，但是后面还有一串编码|
|4|时间戳|
|1|EventHeaderLength，就是19，这里可以做一个校验，如果不是19直接报错即可|
|n|这部分从57开始到len-5结束，不需要关注，也不知道是什么|
|1|校验算法枚举，例如1|
|n|不明|

这一事件的解析决定了以后的事件解析，所以有些校验是必须要做的。

我查阅了go-mysql和canal的代码以后，发现了有几个项目是一定要去校验的：

- commonHeaderLen：这个值的校验两者有些不同，还是看兼容性，在现代版本的MySQL里，这个值一定是19，所以我倾向于直接校验其是否等于19，如果不等就可以结束流程了；
- checksumVersionProduct，这个的校验略显复杂，直接贴代码

```go
checksumVersionSplitMysql   = []int{5, 6, 1}
checksumVersionProductMysql = (checksumVersionSplitMysql[0]*256+checksumVersionSplitMysql[1])*256 + checksumVersionSplitMysql[2]
```

MariaDB略有不同：

```go
checksumVersionSplitMariaDB   = []int{5, 3, 0}
checksumVersionProductMariaDB = (checksumVersionSplitMariaDB[0]*256+checksumVersionSplitMariaDB[1])*256 + checksumVersionSplitMariaDB[2]
```

也就是说，如果binlog里记录的版本是大于5.6.1的，就可以通过校验，否则就会失败。对于MariaDB而言，版本号需要大于5.3.0。从做项目的角度来说，应该在产品手册里明确写上不支持5.6.1以下版本的MySQL。

在得到了校验算法类型之后，后面的时间在parse的时候，就需要首先完成verify。

这里有一个有趣的issue：[10.1.22-MariaDB版本数据库 journalName乱码](https://github.com/alibaba/canal/issues/1081)

我们在生产环境中也遇到过，就是在解析binlog name的时候，莫名奇妙的出现了一些乱码，导致解析失败。这就是由于开源工程的复杂性导致的。

MariaDB和MySQL不同，在ROTATE_EVENT里就采用了checksum逻辑，所以文件名之后那一堆乱码就是checksum。

### 3.5 PREVIOUS_GTIDS_LOG_EVENT事件的解析

这是binlog的第二个事件。这个事件更是平平无奇，平平无奇到canal里连解析都没有：

```java
public class PreviousGtidsLogEvent extends LogEvent {

    public PreviousGtidsLogEvent(LogHeader header, LogBuffer buffer, FormatDescriptionLogEvent descriptionEvent){
        super(header);
        // do nothing , just for mysql gtid search function
    }
}
```

不过go-mysql还是给了解析的逻辑，一遍让我来一探究竟。为了简单期间，我假定的gtid是这样的，也就是只有一组UUID：

```uuid
3d687a2c-b38e-11ed-91ef-3f952e283fc3:1-278
```

|byte|description|
|--|--|
|8|uuid count，也就是有几组UUID|
|16|uuid|
|8|sliceCount，也就是有几组范围，比如1-3:11:47-49就有3组，这里假定一组|
|8|startPos，比如上面的1|
|8|endPos，比如上面的3|

在实际的生产过程中，我们也没有使用过这个事件来做什么，从canal的代码来看，这个事件的作用也确实不大。

到这里，我就分析完了所有binlog的头三板斧。

### 3.6 QUERY_EVENT事件的解析

我个人认为这个Event的名字没有很好的做到见名知意。从其名称来看，似乎是各类`select`之类的查询事件，但实际上却是一类DDL的集合，比如`CreateTableStatement`、`AlterTableStatemt`。

按照惯例，要分析这个Event就要知道这个Event的结构：

|byte|description|
|--|--|
|4|sessionId，分配给这个statement的id，uint32|
|4|execTime，该查询消耗的时间，uint32|
|1|schemaLength，uint8|
|2|errorCodem，uint16|
|2|status variables的长度，v1，v3没有这个参数，uint16|
|n|status variables|
|n|default schema name|
|n|SQL 语句|

如果是MariaDB，在处理SQL语句的时候，需要注意`compress`参数，如果开启了就需要特殊处理。这个可以参考canal的一个issue：[支持下mysql 8.0/maraidb 10.x下的binlog压缩解析能力](https://github.com/alibaba/canal/issues/4388)。

这里没有一个地方标识查询的种类的，比如`CreateTableStatment`，所以，在完成Event的解析后，SQL就显得非常重要。需要引入一个解析器来，将SQL解析成对应的类型。

读代码到这里就会意识到，如果要做事件的解析和后续的工作，就需要自己实现一套模型，这套模型里需要一些满足工作需要的参数。比如QUERY_EVENT，就需要引入一个`Type`参数，标记具体的Query类型。

这个以后再详细描述。

### 3.7 GTID_LOG_EVENT事件的解析

这里需要区分一下一些长相相似的孪生兄弟事件：

- GTID_EVENT：编号162，事实上从160开始的就是MariaDB的事件了，这里先按下不表，有机会研究
- GTID_LOG_EVENT：这才是在`show binlog events`命令里看到的Gtid事件，编号33

这个事件的组成如下表：

|byte|description|
|--|--|
|1|commit flag, 布尔型，保存为uint8|
|16|sid，就是一个uuid|
|8|gno，long64型|
|1|logic timestamp type code，保存为uint8。以下属于可选类型|
|8|lastCommitted，保存为long64|
|8|sequenceNumber，保存为long64|
|7|immediate commit timestamp，定长|
|7|original commit timestamp，定长|
|n|transaction length|
|4|immediate server version|

实际上，canal只分析解析到了sequence number，后面的内容就都忽略掉了。可能这部分的内容对数据的获取没有什么影响。

作为学习，还是需要查查这些内容究竟是什么。

MySQL有一种延迟复制机制，这种能力对DBA来说很实用，比如可以设置一个延迟复制的Slave，作为一个快照，在Master误删数据之后可以补救。

所以，如果设置了延迟复制，那么就会有两个可度量的参数，这两个参数在MySQL8以后才提供：

- immediate_commit_timestamp：将事务写入（提交）到从库的二进制日志的时间戳
- original_commit_timestamp：将事务写入（提交）到主库二进制日志之后的时间戳

来看一个实例：

```text
# at 1297
#240329 23:53:51 server id 1  end_log_pos 1376 CRC32 0x65b589b5 	GTID	last_committed=4	sequence_number=5	rbr_only=yes	original_committed_timestamp=1711727631039014	immediate_commit_timestamp=1711727631039014	transaction_length=316
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1711727631039014 (2024-03-29 23:53:51.039014 CST)
# immediate_commit_timestamp=1711727631039014 (2024-03-29 23:53:51.039014 CST)
/*!80001 SET @@session.original_commit_timestamp=1711727631039014*//*!*/;
/*!80014 SET @@session.original_server_version=80032*//*!*/;
/*!80014 SET @@session.immediate_server_version=80032*//*!*/;
SET @@SESSION.GTID_NEXT= '3d687a2c-b38e-11ed-91ef-3f952e283fc3:283'/*!*/;
```

我的例子里，这两个时间戳就没有区别，这是因为我的库是个主库，主库里这个数据永远相同。所以canal不去处理这两个参数就可以理解了，拿到的数据是主库的，也就没有解析的必要了。

可以查阅参考文献1。

后面的事务大小，我个人觉得也没有太多的解析的必要，作为一个拉取binlog并去做数据同步的组件，这些信息就足够了。

这就回到了事件本身能带给我们的意义了，GTID_LOG_EVENT主要能提供给我们的信息就是GTID本身。

### 3.8 TABLE_MAP_EVENT事件的解析

这是一个TABLE_MAP_EVENT的binlog记录：

```text
# at 981
#240402 19:29:29 server id 1  end_log_pos 1064 CRC32 0xa0606671 	Table_map: `sbtest`.`xa_record5` mapped to number 111
# has_generated_invisible_primary_key=0
# Columns(`id` INT NOT NULL,
#         `uname` VARCHAR(10) CHARSET utf8mb4 COLLATE utf8mb4_0900_ai_ci)
# Primary Key(id)
# at 1064
```

就这么一点点内容，但是极端重要。为了说明其重要，我们需要去看看TABLE_MAP_EVENT之后紧紧跟随的Write_Rows事件：

```text
# at 925
#240329 23:52:40 server id 1  end_log_pos 969 CRC32 0x5f836017 	Delete_rows: table id 111 flags: STMT_END_F
### DELETE FROM `sbtest`.`xa_record5`
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='bar' /* VARSTRING(40) meta=40 nullable=1 is_null=0 */
```

对于执行回放任务的SQL线程来说，这里的"@1"和"@2"就谜语一样无法下手。

所以，在row格式的binlog里，每一个有关数据的DML操作，都一定会有一个TABLE_MAP_EVENT去将这个表定义映射成一个数字。

这个数字就是之前看到的111。以我现在的理解，这个111是MySQL内部给表的编号，可以利用这个编号查询到表的DDL。

也能推出一个结论：

> row格式下的binlog，dml是由TABLE_MAP_EVENT和ROW_LOG_EVENT两个事件组成。

这个table_id是一个很重要的数字，但是却无从查找，因为这个数字是不定的，而且保存在内存里。

现在来分析分析这个事件的构成。先看Post-header部分：

|byte|description|
|--|--|
|6|table_id,6 bytes unsined integer|
|2|flags, 2bytes，现阶段永远是0|

接下来就可以开始解析event里最丰富的信息了，换一种表格：

|name|format|description|
|--|--|--|
|database_name_length|1byte|数据库名称长度，有时候叫schema name|
|database_name|n|数据库名称|
|termination|1|终止符，固定的，解析的时候一定注意跳过|
|table_name_length|1byte|表名的长度|
|table_name|n|表名|
|termination|1|终止符，固定的，解析的时候一定注意跳过|
|column count|packet length|column数量，注意，这里是一个封装起来的长度，如何解析可以参考资料2|
|column type|n|长度取决于column count|
|meta data|n|列的元数据|
|null bits|n|是否为null标记|
|optional|n|这部分比较复杂，下面会仔细解析|

从之前的分析可以得知，想要将ROW_LOG_EVENT里记录的数据变更变成可执行的SQL，那至少需要知道表结构。

基本的table_name和database_name可以用下面的伪代码来解析：

```go
pos = 0
tableId = buffer.uint48()
pos += 6
// 或者直接给0，并将pos+2
flags = buffer.uint16()
pos += 2
databaseNameLength = buffer.uint8()
pos++
// golang的切片这里是两个index，所以end index加上了pos
databaseName = buffer[pos:pos+databaseNameLength]
pos += databaseNameLength

// 跳过终止符
pos++

tableNameLength = buffer.uint8()
pos++

tableName = buffer[pos:pos+tableNameLength]
pos += tableNameLength

// 跳过终止符
pos++
```

到这里，简单的信息我都解析出来了，其中的tableId现在来说没什么用。

更复杂的就是Column的解析。假定我们已经解析出来columnCnt，接下来就可以解析columnType：

```go
columnType = data[pos, pos+columnCnt]
pos += columnCnt
```

到这里，go-mysql和canal就有了设计上的区别了。go-mysql只是简单的将columnType以byte数组的形式保存到了struct里，而canal则直接开始解析byte数组，并将解析结果保存到ColumnInfo类中。作为对事件的理解需要，摘录这段解析代码：

```java
columnInfo = new ColumnInfo[columnCnt];
for (int i = 0; i < columnCnt; i++) {
    ColumnInfo info = new ColumnInfo();
    info.type = buffer.getUint8();
    columnInfo[i] = info;
}
```

虽然columnType本身作为byte数组的长度是不定的，但是内部每一个column的type确实固定的1个byte，用`uint8()`就能取出。

此时，复杂性逐渐就出来了。这个type解析出来仅仅是一个int数据而已，但是它决定了后面的meta应该如何读取。来看看官方对meta的描述，能略知一二：

> For each column from left to right, a chunk of data who's length and semantics depends on the type of the column. The length and semantics for the metadata for each column are listed in the table below.

明确的说了，meta是depends on column type的。

不管是canal还是go-mysql，在解析完type之后，就可以开始解析meta了，比较复杂，所以官方给了一个表格，参见文献3。

举个例子可以加深理解，比如常见的decimal(m,n)，这个类型的identifier是246，名称是MYSQL_TYPE_NEWDECIMAL，注意不要和MYSQL_TYPE_DECIMAL混淆。

这个类型的meta长度是2byte：

```java
int x = buffer.getUint8() << 8; // precision
x += buffer.getUint8(); // decimals
```

完成meta的解析以后，我们可以得到的信息就相当完整了，有database-table的全部信息和column的大部分信息。

nullable的属性也和columnCnt有关，解析代码如下：

```go
nullBitmapSize := bitmapByteSize(int(e.ColumnCount))
if len(data[pos:]) < nullBitmapSize {
    return io.EOF
}

e.NullBitmap = data[pos : pos+nullBitmapSize]

pos += nullBitmapSize
```

`bitmapByteSize`是`int((column_count + 7) / 8)`，可以参考文献3的说明。

由于我的目的不是将MySQL的数据转换为MySQL，所以我对后面的charset等都不是很感兴趣，所以option的信息就先不分析如何解析了。

所以这个Event完成分析以后，我们就能得到一个完整的数据表DDL信息了。接下来只要知道数据，就可以将这些信息拼成一个可用的SQL了。

### 3.9 WRITE_ROWS_EVENTv2事件的解析

### 3.10 XID_EVENT事件的解析

### 3.11 DELETE_ROWS_EVENTv2事件的解析

### 3.12 UPDATE_ROWS_EVENTv2事件的解析

## 参考文献

[1. replication-delayed](https://dev.mysql.com/doc/refman/8.3/en/replication-delayed.html)
[2. getPackedLong](https://github.com/alibaba/canal/blob/master/dbsync/src/main/java/com/taobao/tddl/dbsync/binlog/LogBuffer.java#L1088)
[3. Table__map__even](https://dev.mysql.com/doc/dev/mysql-server/8.0.34/classbinary__log_1_1Table__map__event.html)