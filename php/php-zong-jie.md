# PHP总结

## 基础篇

### PHP如何实现类自动加载？

* 设置自动加载目录 set\_include\_path(String $path)。

```php
set_include_path(dirname(__file__) . '/classes');
```

* 修改php.ini文件include\_path选项
* 写一个\_\_autoload($class)方法。

```php
 function __autoload($class) {
     $path = dirname(__FILE__) . "/autoload/$class.php";
     echo $path;
     if (!file_exists($path)) {
         die ("class $class can not load\n");
     }

     include $path;
 }

 $b = new B();
```

* 写一个加载器，然后用spl\_autoload\_register方法注册。

```php
<?php
DEFINE('ROOT_PATH', dirname(__FILE__));
class Loader {
    public static function load($class) {
        $path = null;
        if (substr($class, 0 - strlen('Controller')) == 'Controller') {
            $path = ROOT_PATH . '/Controllers/';
        } else if (substr($class, 0 - strlen('Model')) == 'Model') {
            $path = ROOT_PATH . '/Models/';
        } else {
            $path = ROOT_PATH . '/Components/';
        }
        $path .= "$class.php";

        if (file_exists($path)) {
            include $path;
            return;
        }

        return false;
    }
}

spl_autoload_register(array('Loader', 'load'), false, false);


$controller = new AController();
```

推荐第四种，原因如下

第一第二种不够灵活

第三种灵活，但是\_\_autoload只能被重写一次

第四种相对灵活，而且在引用第三方差价或者工具的时候有非常明显的优势

### include/include\_once/require/require\_once的区别？

include在文件不存在的时候会打印警告信息然后继续执行，而require会终止程序，相对来说include效率更高

include\_once和require\_once在加载文件之前都会先判断文件是否已经被加载过，相对来说效率低一些

### 双引号和单引号的区别？

双引号会尝试解析替换里面的变量，而单引号不会，相对来说单引号效率高

双引号解释转义字符，而单引号不解释转移字符，比如

### 常见的超全局变量

* $\_GET：获取GET参数
* $\_POST：获取POST参数
* $\_REQUEST：可以接收到get和post两种方式的值
* $\_SESSION：
* $\_COOKIE：
* $\_FILE：上传文件的时候用到
* $\_SERVER：系统环境变量
* $GLOBALS：所有的变量都可以在这里获取

### 全局变量怎么获取？

* 从$\_GLOBALS数组里面取
* global + 变量名

```php
 $a = 1;

 function g1() {
     var_dump($GLOBALS['a']);
 }

 function g2() {
     global $a;
     var_dump($a);
 }

 g1();
 g2();
```

### PHP的几种魔术方法？

* \_\_construct()：构造方法，new一个对象的时候调用
* \_\_destruct()：析构方法，对象销毁的时候调用，通常一个资源的释放放在这里面
* \_\_clone()：定义如何克隆一个对象
* \_\_set($name, $value)：尝试赋值一个不可访问的属性时被调用
* \_\_get($name)：尝试获取一个不存在的属性时被调用
* \_\_isset($name)：在对一个不可访问的属性调用isset()方法时会被调用
* \_\_unset($name)：在调用unset()函数销毁一个不能访问的属性时会被调用
* \_\_sleep()：对象被序列化时调用
* \_\_wakeup()：对象被反序列化时调用
* \_\_toString()：定义对象的字符串表示，echo，print一个对象时自动调用
* \_\_call($method, $arguments)：调用不存在或不可访问的方法时会被调用
* \_\_callStatic($method, $arguments)：调用不存在或不可访问的静态方法时会被调用
* \_\_invoke()：在尝试将对象作为函数使用时会被调用

### 自定义错误日志级别

error\_reporting方法

### 自定义异常

自定义一个class，继承系统系统的Exception

```php
class MyException extends Exception {
 }

throw new MyException('MyException');
```

### 自定义错误处理

通过set\_error\_handler设置自定义的错误处理，通常用来做一些日志记录

```php
<?php
class ErrorHandler {
    public static function error($errno, $errstr, $errfile, $errline) {
        echo "error $errstr [$errno] triggered in $errfile line [$errline];\n";
        die;
    }
}

set_error_handler(array('ErrorHandler', 'error'));

trigger_error('test error');

```

### 自定义异常处理

通过set\_exception\_handler自定一个异常处理器

```php
<?php
class ExceptionHandler {
    public static function handler($exception) {
        echo get_class($exception), ": ", $exception->getMessage(), "\n";
        debug_print_backtrace();
        die;
    }
}

set_exception_handler(array('ExceptionHandler', 'handler'));

class MyException extends Exception {
}

throw new MyException("test exception");
```

### 什么是魔术引号

magic\_quotes\_gpc，自动转义引号，不建议开启，在业务需要的时候去做即可；

可以通过set\_magic\_quotes\_runtime设置，get\_magic\_quotes\_runtime获取配置。

### 方法前面加@和不加的区别

方法前面加@会抑制错的打印，性能会有影响，不建议使用，应该在开发阶段尽量的把错误暴露出来。

### 传值和传引用

一般变量作为参数时通常都是传值，对象作为参数时是传引用；传引用函数修改变量时，外部变量会同步修改，传引用本质上是传递的内存地址

### PHP内存回收机制

php内存回收是通过引用计数来操作的，变量被初始化时会有一个引用次数，变量没被使用一次，引用计数加1，当引用计数为0时就可以被回收

## 协议篇

### http常见的响应码

200：成功

301：永久重定向

302：临时重定向

401：未授权，通常是身份验证失败

403：资源禁止访问，通常是用户验证通过了，但是没有对应权限

404：资源不存在

500：内部错误，通常是php执行出错，一般通过日志文件可以找到原因

502：网关错误，通常是php执行时间过长

503：服务器目前无法使用（由于超载或停机维护）。通常，这只是暂时状态。（服务不可用）

504：网关超时，通常是反向代理后面的服务挂了，nginx连不上内部服务

### GET方法和POST方法区别

* 语义上来说，GET表示获取一个资源，POST通常表示创建一个资源
* URL上来说，GET请求参数都在URL上，POST请求参数多在请求体(body)里，相对来说POST请求更安全
* 通常来说，GET请求和通过刷新重复访问，而POST请求不可以（chrome可以支持刷新再次访问，对于开发者来说也是坑，无意中多写了一条数据到数据库）
* 长度限制，各家浏览器对于URL的长度都是有限制，通常1024字节；GET请求的参数室友限制的，理论上来说POST请求的参数是没有限制的

### REST/RESTFUL接口是什么意思？

REST是参照了HTTP协议的初衷来设计的方式，即把所有的数据看做一个个的资源，把对数据的CRUD看做对资源的查看/创建/修改/删除，对应HTTP的GET/POST/PUT/DELETE方法；从语义上来说更加清晰，适用于比较简单的业务或者逻辑紧凑，高内聚的业务

比如以新闻为例来设计接口

/api/news：GET方法，表示获取新闻列表，返回数据格式是一个list

/api/news/1：GET方法，表示获取信id=1的新闻详情，返回数据格式是一个dict/object

/api/news/：POST方法，表示创建一条新闻，返回数据格式是一个dict/object，包含新闻id

/api/news/1：PUT方法，表示修改id=1的新闻信息，返回格式是一个dict/object

/api/news/1：DELETE方法：表示删除id=1的新闻信息，可以不返回body部分

### session和cookie的区别和联系？

* 存储上说session存在服务端，cookie存在客户端；时间上来说cookie的有效期比session长；安全性上说session相对cookie安全。
* session是会话相关的，会话没了（比如浏览器被关了），session也就不存在了。
* session依靠cookie，因为session的识别是通过cookie传过来的（通常key是PHPSESSIONID），浏览器禁用了cookie，session也就没法识别了

建议重要信息和敏感信息都放在session，不重要的放到cookie

### cookie的格式，有效期，访问权限控制

```
key1=value1; key2=value2; 过期时间; 路径; 域名
```

过期时间，路劲，和域都不是必须的；默认是根路径（/）；默认域名是当前域名

子域名（pan.baidu.com）可以访问父域名(.baidu.com)的下cookie，反过来不行

通过set\_cookie或者header方法可以设置cookie。

### 跨域限制如何解决？

* 异步处理可以使用ajax + jsonp搞定
* p3p协议授权

### TCP的三次握手和4次挥手协议

http://blog.csdn.net/whuslei/article/details/6667471/

#### 三次握手

客户端发送SYN

服务端收到SYN：此时服务端知道客户端可以正常发送数据包，同时也知道服务端可以正常接收数据包

服务端发送SYN + ACK：

客户端收到SYN + ACK: 此时客户端知道自己可正常发送和接收数据包，同时知道服务端也可以正常发送和接收数据包（为什么？因为服务端收到了SYN才会给客户端发送SYN+ACK）

客户端发送ACK：

服务端接收ACK：此时服务端知道客户端可以正常接收数据包（为什么？客户端收到了SYN+ACK之后才会给服务端发送ACK）

_**注意**_：三次握手不保证传输可靠性（不丢包），只是确保了双方都可以正常接收和发送数据包；至于不丢包是通过超时重传保证的

#### 超时重传

发送方每次发送数据包的时候，都会带上一个随机数，接收方收到数据之后会回应一个响应包（ack），然后加上一个数字（之前的随机数+1），发送方收到ack之后就知道接接收方已经收到数据包了，如果超过一定时间没有收到ack，发送方就认为数据包丢了，会再发一次数据包；所以接收方在理论上可能接到统一份数据包多次，需要做去重处理（系统已经做了）

#### 四次挥手

* 客户端发送关闭请求（FIN）：我该发的数据发完了，我准备关闭了
* 服务端回应ACK：行，我知道了，但是我数据还没传完，你等等再关
* 服务端发送关闭请求（FIN）：我数据传完了，你可以关闭了
* 客户端发送ACK：好的，我过会儿就自动关了哈
* 服务端收到ACK：

需要四次挥手的原因主要是因为需要确认双方的数据都发送完了才能关闭；所以双方在发送关闭请求之后会有两次确认，一次确认收到了对方的关闭请求，一次确认数据发送完了。

#### 客户端状态变化

CLOSED：关闭状态（初始状态）

SYN\_SEND：刚刚发送SYN，准备建立链接

ESTABLISHED：收到了SYN+ACK，此时客户端发送完ACK就可以发送数据包了

FIN-WAIT1：发送完FIN，准备关闭链接

FIN-WAIT2：收到服务端的ACK，服务端已经收到了关闭请求，但是数据还没有传输完

TIME-WAIT：收到了服务端的FIN，表示服务端数据传完了，发送完ACK等待一段时间就可以关闭了

#### 服务端状态变化

LISTEN：监听状态（初始状态）

SYN\_RCVD：收到了客户端SYN包，需要准备建立连接了

ESTABLISHED：收到了客户端ACK包，准备收发数据了

CLOSE-WAIT：收到了客户端FIN包，客户端数据传完了

LAST-ACK：服务端数据传输完了，发送ACK通知客户端

CLOSE：客户端收到了FIN，并且ACK了，服务端关闭连接

如何查看服务器上的tcp链接

netstat命令：http://man.linuxde.net/netstat

### HTTPS和HTTP的区别

http://www.ruanyifeng.com/blog/2011/02/seven\_myths\_about\_https.html

http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html

HTTPS=HTTP + SSL/TLS

本质上就是HTTP协议，只不过在开始之前会有一次握手协商加密密钥

服务器配置的证书只是为了加密传输最后一个随机数，因为是从客户端往服务端发的，且服务端持有的是私钥，所以被破解的概率很小；因为最后一个随机数被破解的概率小，所以也就保证了整个会话过程密钥被破解的概率很小；同时每次会话都会生成不同的密钥，所以暴力破解的希望也不大。

## 数据库篇

### MyISAM和InnoDB的区别

MyISAM：不支持事务；表级锁；索引和数据存放在一起；数据恢复相对困难；查询相对快。

InnoDB：支持事务；行级锁，索引和数据分开存放；数据恢复相对简单；查询相对慢。

一般来说现在说的都是InnoDB存储引擎

### 索引结构

B+Tree：最常见的索引结构，默认说的索引就是B+数索引

Hash索引：从名字就可以看出来，索引算法是hash算法，只适用于等值查询(a = 'xxxxxx')，对于范围查询不适用

### 索引创建的一些原则

* 区分度越来越好；根据索引查询时扫描的行数少
* 索引长度越短越好；单页存储的索引数据多，IO次数少
* 索引越简单越好；数字索引比字符串索引好
* 查询使用多的列加索引
* 排序用到的列加索引
* 索引不是越多越好；索引也占据磁盘空间，写入的时候构建索引也耗时

### 如何查看sql语句性能

使用explain

```sql
explain select c from t where id = 5;
```

```
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: NULL
   partitions: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: no matching row in const table
1 row in set, 1 warning (0.00 sec)
```

关注的列

* type
* key：使用到的索引
* key\_len：索引长度
* rows：预计扫描行数
* extra：查看是否使用文件排序，临时表之类的

### 如何找到查询比较慢的sql

* 通过mysql慢查询日志查看
* 写完sql explain看下性能

### 什么情况下索引无效

* 通过索引扫描行数大于总数据20%时不会走索引；走索引会大量随机IO，性能可能不如全表扫描走顺序IO效率高
*   最左前缀匹配原则

    ```sql
    create index `idx_xxx` (`a`, `b`, `c`);
    ```

    ```sql
    select * from t where a = 1 and b = 2 and c = 3;
    ```

    会用到索引的所有部分

    ```sql
    select * from t where a = 1 and c = 3;
    ```

    会用到索引的a部分

    ```sql
    select * from t where b = 2 and c = 1
    ```

    不会用到索引

### 数据库隔离级别什么意思？

mysql做到了RR级别 [微信上一篇讲隔离级别比较好的文章](https://mp.weixin.qq.com/s?__biz=MzAxOTc0NzExNg==\&mid=2665513435\&idx=1\&sn=41ffb114be4a5b53de87831b9c8427bf\&chksm=80d67998b7a1f08e1b1187915609ecc43715151b015ac156ce5a9645e654910cdda1d48d5883\&mpshare=1\&scene=1\&srcid=06023qmtatDjl2iUrO9YjeUw\&key=7dffde877c8eec6a2f4bcfbae75c619b4add5dfa71357aa804b426737c57081f21daff3d1e925dc6eebd3424cdd0e92f1e314c867cd1e2987aa6f2292875952e582abf0ab35de6e98869a7c8a2e82009\&ascene=0\&uin=OTkyNDg2MDgx\&devicetype=iMac+MacBookPro11%2C2+OSX+OSX+10.12.4+build\(16E195\)\&version=12020510\&nettype=WIFI\&fontScale=100\&pass_ticket=Vp76quG8JQIgm40HO1JQshMZuSEvKCAkEKOsSnlTetkRzpT2f3QD%2FkM0eHVx8DKT)

## 缓存篇

### memcache

* 纯内存数据，无法备灾
* 不支持主从结构
* 淘汰算法：LRU
* 简单kv结构：
* slab chunk机制：memcache会把内存划分成不同的slab，每个slab由大小相同的多个chunk组成，不同的slab chunk大小不一致；比如128B的一堆chunk组成一个slab，256B的chunk组成另一chunk，chunk的增长因子默认是2，可以通过参数调整；比如一个大小为98B的数据过来，memcache会选择一个最小的能容下数据的chunk(128b)来存储，当一个大小为130的过来，memcache会选择256的chunk来存储，所以memcache会有很多的碎片：一个slab可能有剩余的空间不足一个chunk；一个chunk内因为数据太小不够一个chunk也有碎片；所以需要根据实际需要划分好chunk和增长因子

### redis

* 非纯内存结构，直接持久化到磁盘：两种形式，RDB(快照)，或者AOF(写请求日志)，数据恢复时优先选择AOF，类似mysql binlog
*   支持主从结构：全量同步/增量同步，全量同步步骤

    * 1、从连主，发送sync命令
    * 2、主bgsave生成快照
    * 3、发送快照
    * 4、从load快照
    * 5、主发送增加写命令

    增量同步只发送写命令即可
* 数据结构丰富：kv、list、map、set...

## 队列篇

### beanstalkd

* producer、consumer、tube、job
* 纯内存，不支持集群，无备灾

### kafka

* producer、consumer、consumer group、broker、partition、offset
* partition 分段存储，多个segment file，只有flush 到磁盘的数据才能被消费
* 高可用
  * 每个partition有自己的replication
  * 多个partition需要选取出lead partition，lead
  * partition负责读写，并由zookeeper负责fail over

## 分布式

### CAP

* C：一致性
* A: 可用性
* P：容错性 CAP只能同时满足其中的两个，不能同时满足三个，系统设计都是在CAP中权衡；通常性能一致性，通过补偿机制来保证最终一致性

### 分布式算法

#### 两阶段提交和三阶段提交

http://blog.jobbole.com/95632/

**两阶段提交**

* 1、协调者向所有参与者发送提交请求，并接收响应
* 2、有一个失败然后就发送回滚请求

**三阶段提交**

* 1、canCommit
* 2、preCommit
* 3、doCommmit
* 4、超时机制

**paxos & raft**

http://www.cnblogs.com/cchust/p/5617989.html paxos是比较经典的分布式一致性算法；业界已经有现成的实现 zookeeper raft是paxos的精简版本

## 高并发

### 常见的方法

* 系统尽量做到无状态，可以通过扩容来提升服务能力
* 使用缓存来减轻db压力，提升响应速度
* 使用队列异步化一些比较耗时的操作
* 数据库层面还可以做读写分离
* 数据量特别大的时候可以对数据做，分库分表处理，也可以做分区处理（通常用在实时性要求，不高或者写入多读取少的场景）
* 有可能设计到分布式锁，可以使用redis来实现setNx http://blog.csdn.net/lihao21/article/details/49104695
* 限流/过载保护

### 说到无状态，分布式session怎么做？

* 复制：每台机器都有全部的session数据，机器之前互相拷贝；容量有瓶颈，同时浪费资源
* hash：相同ip的请求始终落到一台后端机器上，每个机器只存自己的那一份session数据就好了；宕机了一部分session就丢了，hash策略换了也比较麻烦
* 集中管理：做一个独立的session服务器，常见的用db或者缓存来做（redis）

### SOA和微服务的区别？

https://www.zhihu.com/question/37808426

### 分表的一些做法

* 横切：每张表的数据结构一致，只是数据不同，通常根据用户的一些属性来分表（比如user\_id）；常见的分表策略有两种
  * 根据分片键取模：比如 user\_id % 100 = 1 的在一张表，=2的在另一张表；缺陷就是取模可能导致没张表的数据不均衡，同时再次扩容的时候数据迁移也是麻烦
  * 根据分片键范围：比如 user\_id < 10000的在一张表，10000 - 20000 的在另一张表，缺陷就是最后一张表数据可能比较少，不均衡，同时如果想改变分片策略也很麻烦，但是扩容比较容易
* 纵切：把一张大小拆分成几张小表，没张表的数据一样多，但是数据结构不一样，比如把用户表拆分核心字段表(用户名，密码，手机）和扩展字段表（积分，等级，简介等等）

### 限流

限流，顾名思义，即限制系统的访问流量，常见的限流算法：计数器，漏通，令牌桶

http://www.cnblogs.com/clds/p/5850070.html

## 网络安全篇

* XSS：跨站脚本攻击，用户输入带有一些html或者js标签，展示的时候如果不处理会有一些意想不到的效果；转义处理就好
* CSRF: 跨站请求伪造，尽量不要用GET方法操作数据；数据处理的时候做权限校验；敏感操作加验证码；表单隐藏提交一个token做校验
* SQL注入：比较常见，一般都是手动拼装sql导致的，通过prepare预处理的方式即可规避
* DDOS：分布式拒绝服务攻击，通常没有特别好的办法，一般就是加机器
* CC：重放攻击，一直访问一个比较耗时的请求，可以击垮数据库，或者占满服务器连接
* 文件上传漏洞：通过上传一个可执行文件，然后通过url访问执行在服务器上做破坏操作；上传文件的时候校验文件格式，另外上传的文件不要放到可执行目录(可以被解析执行的目录)

## 设计模式篇

* 单例模式：数据库连接
* 工厂模式：PDO创建数据库连接
* 代理模式：比如对缓存的封装，提供统一的接口，可以随时用redis代替memcache，对上层应用无感知
* 策略模式：主要是减少if-else
* 装饰器模式：对数据的增强
* 模板模式：框架里面用的比较多，比如Yii model提供的beforeSave方法
* 观察者模式：队列里就有
* 责任链模式：流程处理方面用的比较多
* 门面模式（外观模式）: 总结，基本上就是面向对象的5大原则
* 单一职责原则（SRP）：一个类只做一件时间
* 开放封闭原则（OCP）：对扩展开放，对修改关闭
* 里氏替换原则（LSP）：子类可以当做一个父类
* 依赖倒置原则（DIP）：高层模块不要依赖底层模块；面向接口编程，不依赖具体实现
* 接口隔离原则（ISP）：使用多个专门的接口比使用单个接口要好的多

## 其他

### PHP变量的实现以及内存回收

变量实现：http://www.cunmou.com/phpbook/2.md 内存管理：http://www.cunmou.com/phpbook/3.md

### PHP数组的底层实现

http://www.cunmou.com/phpbook/8.md

### PHP扩展可以做连接池吗？

可以，参考模块的加载流程，在MINIT方法里申请资源，MSHUTDOWN里面销毁资源 http://www.cunmou.com/phpbook/1.3.md
