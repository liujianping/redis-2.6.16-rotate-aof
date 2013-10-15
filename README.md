#Redis 2.6.16 rotate.aof 功能说明

## 万恶的 AOF Rewrite

###AOF rewrite 缺陷说明

auto rewrite功能的引入，主要是为了解决aof文件不断增长减少重复键读写操作的日志,
通过定时触发,重新根据实际内存键值情况写入新的aof文件, 再进新旧替换文件,实现aof文件最小化。

然而通过测试,系统在大压力下进行auto rewrite操作【maxmomery 40gb】,导致redis服务会有6-10s的延时。
该延时与设置的 maxmomery 设置值相关。maxmomery 越大延时越严重。

同时auto rewrite的使用设置的 maxmomery 值越大, 还会导致COPY ON WRITE内存的分配。
这就是为什么作者在安装建议中提出，需预留1倍的内存出来。在极限情况下,
一个redis的实例会用去 maxmomery * 2 内存。

###AOF rewrite 缺陷实测数据

版本 2.6.16 中 文件的flush磁盘操作与 close操作 均已经放入 background 线程操作。
但是auto rewrite操作仍然出现较长延时, 对文件操作加入debug日志发现, 以下日志 
设置redis maxmomery 40gb：
    
    [6831] 23 Sep 11:16:26.489 * Starting automatic rewriting of AOF on 100% growth
    [6831] 23 Sep 11:16:26.535 * Background append only file rewriting started by pid 7613
    [6831] 23 Sep 11:16:36.879 * Background AOF buffer size: 80 MB
    [6831] 23 Sep 11:16:38.254 * Background AOF buffer size: 180 MB
    [6831] 23 Sep 11:16:39.910 * Background AOF buffer size: 280 MB
    [6831] 23 Sep 11:16:41.752 * Background AOF buffer size: 380 MB
    [6831] 23 Sep 11:16:43.350 * Background AOF buffer size: 480 MB
    [6831] 23 Sep 11:16:45.067 * Background AOF buffer size: 580 MB
    [6831] 23 Sep 11:16:46.726 * Background AOF buffer size: 680 MB
    [6831] 23 Sep 11:16:48.562 * Background AOF buffer size: 780 MB
    [6831] 23 Sep 11:16:50.342 * Background AOF buffer size: 880 MB
    [6831] 23 Sep 11:16:52.015 # Background AOF buffer size: 980 MB
    [6831] 23 Sep 11:16:53.763 * Background AOF buffer size: 1080 MB
    [6831] 23 Sep 11:16:55.251 * Background AOF buffer size: 1180 MB
    [6831] 23 Sep 11:16:56.928 * Background AOF buffer size: 1280 MB
    [6831] 23 Sep 11:16:58.777 * Background AOF buffer size: 1380 MB
    [6831] 23 Sep 11:17:00.468 * Background AOF buffer size: 1480 MB
    [6831] 23 Sep 11:17:02.127 * Background AOF buffer size: 1580 MB
    [6831] 23 Sep 11:17:03.602 * Background AOF buffer size: 1680 MB
    [6831] 23 Sep 11:17:05.156 * Background AOF buffer size: 1780 MB
    [6831] 23 Sep 11:17:07.040 * Background AOF buffer size: 1880 MB
    [6831] 23 Sep 11:17:08.509 # Background AOF buffer size: 1980 MB
    [6831] 23 Sep 11:17:10.261 * Background AOF buffer size: 2080 MB
    [6831] 23 Sep 11:17:11.837 * Background AOF buffer size: 2180 MB
    [6831] 23 Sep 11:17:13.293 * Background AOF buffer size: 2280 MB
    [6831] 23 Sep 11:17:14.760 * Background AOF buffer size: 2380 MB
    [6831] 23 Sep 11:17:16.432 * Background AOF buffer size: 2480 MB
    [6831] 23 Sep 11:17:18.093 * Background AOF buffer size: 2580 MB
    [6831] 23 Sep 11:17:19.561 * Background AOF buffer size: 2680 MB
    [6831] 23 Sep 11:17:21.206 * Background AOF buffer size: 2780 MB
    [6831] 23 Sep 11:17:22.665 * Background AOF buffer size: 2880 MB
    [6831] 23 Sep 11:17:24.393 # Background AOF buffer size: 2980 MB
    [6831] 23 Sep 11:17:26.582 * Background AOF buffer size: 3080 MB
    [6831] 23 Sep 11:17:29.029 * Background AOF buffer size: 3180 MB
    [6831] 23 Sep 11:17:31.540 * Background AOF buffer size: 3280 MB
    [6831] 23 Sep 11:17:33.321 * Background AOF buffer size: 3380 MB
    [6831] 23 Sep 11:17:34.782 * Background AOF buffer size: 3480 MB
    [6831] 23 Sep 11:17:36.329 * Background AOF buffer size: 3580 MB
    [6831] 23 Sep 11:17:38.017 * Background AOF buffer size: 3680 MB
    [6831] 23 Sep 11:17:39.684 * Background AOF buffer size: 3780 MB
    [6831] 23 Sep 11:17:41.352 * Background AOF buffer size: 3880 MB
    [7613] 23 Sep 11:17:42.899 * SYNC append only file rewrite performed
    [7613] 23 Sep 11:17:42.903 * AOF rewrite: **41858 MB** of memory used by **copy-on-write**
    [6831] 23 Sep 11:17:42.984 # Background AOF buffer size: 3980 MB
    [6831] 23 Sep 11:17:43.070 * Background AOF rewrite terminated with success
    _[6831] 23 Sep 11:17:43.070 # Rewrite Done Parent Proc open file:temp-rewriteaof-bg-7613.aof last:100 usecs_
    [6831] 23 Sep 11:17:45.491 * Parent diff successfully flushed to the rewritten AOF (4178936455 bytes)
    _[6831] 23 Sep 11:17:49.401 # Rewrite Done Parent Proc rename file:temp-rewriteaof-bg-7613.aof last:3909908 usecs_
    [6831] 23 Sep 11:17:49.401 * Background AOF rewrite finished successfully

从日志中发现, 文件的rename操作用去了近4s, 相当与redis 4s 没有响应，通过 tcprstat 实时监控，
redis无响应时间在8s左右.如果应用对实时响应要求很高，10s无响应肯定是无法接受的。

##AOF rotate

###目的

将AOF在线实时Rewrite的功能，切换到线下操作。提升了redis性能的同时提升内存的利用率。

###功能说明

rotate其实是参照日志程序的功能，当aof文件达到最大值阀值时，切换aof文件，将老的aof文件重命名添加[.数字]的后缀，后续操作命令写入到新的空的aof文件中。如此循环。

同时，rotate设置最大切换文件的数目，一旦超过该总数，后缀再次重0开始计数。防止硬盘写满出现异常。
运维时必须尽量保证rotate不会自动重0开始计数，因为一旦从0开始循环，相当与早期的操作命令被冲掉。

###功能配置

####redis.conf 增加两项配置：

    #文件rotate最大阀值
    auto-aof-rotate-max-size  20gb

    #磁盘最多存储rotate文件数 保证  
    #【auto-aof-rotate-max-size * auto-aof-rotate-max-total < disk storage】
    auto-aof-rotate-max-total 4

####AOF rotate启用条件

AOF rotate功能不是设置就会生效，必须满足以下两个条件:

    appendonly 功能必须打开： 【appendonly yes】
    auto rewrite功能必须关闭: 【auto-aof-rewrite-percentage 0】

###实测数据

根据写入debug日志，发现每次发生AOF rotate切换文件的速度是 < 45us ，基本上可以忽略不计。redis实例整体不会出现
长时间的延迟。
    
###线下运维

引入AOF rotate模式后，必然导致一个新的问题，这些切分aof文件如何合并。

该版本提供一个线下合并aof文件的redis-offline服务实例工具，系统管理人员必须通过启动线下实例进行aof合并，
因为在 redis-offline 实例模式下提供了 新的 **mergeaof** 的合并命令。

mergeaof 命令语法:

    redis-cli >: mergeaof /path/of/appendonly.aof.x

mergeaof 实现说明:

    mergeaof 是一个同步操作，具体操作时间与待合并aof文件相关。

例如：

redis实例工作目录下,出现以下切分aof文件：

    appendonly.aof
    appendonly.aof.0
    appendonly.aof.1
    appendonly.aof.2
    appendonly.aof.3

一旦实例发生当机或者任何突发事件，需要恢复以上数据，可以通过 **【mergeaof 命令】** 进行恢复

启动一个线下实例，主要用于合并aof重写新的aof

    $: redis-offline  -p 9999 【或者 $: redis-server --offline  】

合并切分文件    

    $: redis-cli -p 9999 mergeaof /path_of_backup/appendonly.aof.0
    $: redis-cli -p 9999 mergeaof /path_of_backup/appendonly.aof.1
    $: redis-cli -p 9999 mergeaof /path_of_backup/appendonly.aof.2
    $: redis-cli -p 9999 mergeaof /path_of_backup/appendonly.aof.3
    $: redis-cli -p 9999 mergeaof /path_of_backup/appendonly.aof

线下合并后执行bgrewrite操作  

    $: redis-cli -p 9999 bgrewriteaof

实例目录下，生成新的 appendonly.aof 则为以上切分文件合并后的aof数据。


#Redis 2.6.16 README
Where to find complete Redis documentation?
-------------------------------------------

This README is just a fast "quick start" document. You can find more detailed
documentation at http://redis.io

Building Redis
--------------

Redis can be compiled and used on Linux, OSX, OpenBSD, NetBSD, FreeBSD.
We support big endian and little endian architectures.

It may compile on Solaris derived systems (for instance SmartOS) but our
support for this platform is "best effort" and Redis is not guaranteed to
work as well as in Linux, OSX, and *BSD there.

It is as simple as:

    % make

You can run a 32 bit Redis binary using:

    % make 32bit

After building Redis is a good idea to test it, using:

    % make test

Fixing problems building 32 bit binaries
---------

If after building Redis with a 32 bit target you need to rebuild it
with a 64 bit target, or the other way around, you need to perform a
"make distclean" in the root directory of the Redis distribution.

In case of build errors when trying to build a 32 bit binary of Redis, try
the following steps:

* Install the packages libc6-dev-i386 (also try g++-multilib).
* Try using the following command line instead of "make 32bit":

    make CFLAGS="-m32 -march=native" LDFLAGS="-m32"

Allocator
---------

Selecting a non-default memory allocator when building Redis is done by setting
the `MALLOC` environment variable. Redis is compiled and linked against libc
malloc by default, with the exception of jemalloc being the default on Linux
systems. This default was picked because jemalloc has proven to have fewer
fragmentation problems than libc malloc.

To force compiling against libc malloc, use:

    % make MALLOC=libc

To compile against jemalloc on Mac OS X systems, use:

    % make MALLOC=jemalloc

Verbose build
-------------

Redis will build with a user friendly colorized output by default.
If you want to see a more verbose output use the following:

    % make V=1

Running Redis
-------------

To run Redis with the default configuration just type:

    % cd src
    % ./redis-server
    
If you want to provide your redis.conf, you have to run it using an additional
parameter (the path of the configuration file):

    % cd src
    % ./redis-server /path/to/redis.conf

It is possible to alter the Redis configuration passing parameters directly
as options using the command line. Examples:

    % ./redis-server --port 9999 --slaveof 127.0.0.1 6379
    % ./redis-server /etc/redis/6379.conf --loglevel debug

All the options in redis.conf are also supported as options using the command
line, with exactly the same name.

Playing with Redis
------------------

You can use redis-cli to play with Redis. Start a redis-server instance,
then in another terminal try the following:

    % cd src
    % ./redis-cli
    redis> ping
    PONG
    redis> set foo bar
    OK
    redis> get foo
    "bar"
    redis> incr mycounter
    (integer) 1
    redis> incr mycounter
    (integer) 2
    redis> 

You can find the list of all the available commands here:

    http://redis.io/commands

Installing Redis
-----------------

In order to install Redis binaries into /usr/local/bin just use:

    % make install

You can use "make PREFIX=/some/other/directory install" if you wish to use a
different destination.

Make install will just install binaries in your system, but will not configure
init scripts and configuration files in the appropriate place. This is not
needed if you want just to play a bit with Redis, but if you are installing
it the proper way for a production system, we have a script doing this
for Ubuntu and Debian systems:

    % cd utils
    % ./install_server

The script will ask you a few questions and will setup everything you need
to run Redis properly as a background daemon that will start again on
system reboots.

You'll be able to stop and start Redis using the script named
/etc/init.d/redis_<portnumber>, for instance /etc/init.d/redis_6379.

Code contributions
---

Note: by contributing code to the Redis project in any form, including sending
a pull request via Github, a code fragment or patch via private email or
public discussion groups, you agree to release your code under the terms
of the BSD license that you can find in the COPYING file included in the Redis
source distribution.

Please see the CONTRIBUTING file in this source distribution for more
information.

Enjoy!
