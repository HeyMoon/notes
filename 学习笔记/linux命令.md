1. ps:查看进程信息
常用组合：ps -efl

2. top:显示进程的实时信息
top默认是根据pid排序，不过输入o之后，可以选择排序的字段：

````
Processes: 438 total, 2 running, 1 stuck, 435 sleeping, 1983 threads   02:36:51
Load Avg: 1.37, 1.32, 1.55  CPU usage: 2.40% user, 4.92% sys, 92.66% idle
SharedLibs: 194M resident, 46M data, 27M linkedit.
MemRegions: 148682 total, 4085M resident, 98M private, 2872M shared.
PhysMem: 16G used (2883M wired), 142M unused.
VM: 5492G vsize, 627M framework vsize, 4224047(0) swapins, 4891071(0) swapouts.
Networks: packets: 68504816/78G in, 48307946/7781M out.
Disks: 4200003/96G read, 5884592/184G written.

PID    COMMAND      %CPU TIME     #TH   #WQ  #PORTS MEM    PURG   CMPRS  PGRP
99440  Google Chrom 0.0  02:39.93 15    2    121    28M    0B     67M    308
99328  KLoader      0.0  00:00.00 2     0    9      24K    0B     332K   12432
99325  java         0.0  04:41.62 27    1    91     1180K  0B     54M    99315
99081  Google Chrom 0.0  02:20.74 15    2    123    7156K  0B     87M    308
99042  Google Chrom 0.0  03:09.87 15    2    120    39M    0B     60M    308
94128  com.apple.au 0.0  00:00.13 2     2    17     252K   0B     908K   94128
92978  com.apple.Pr 0.0  00:00.12 2     2    26     288K   0B     684K   92978
92976  Preview      0.0  03:48.82 6     3    311    286M   1280K  700M
````

比如输入o之后，再输入`-mem`:

````
Processes: 438 total, 2 running, 1 stuck, 435 sleeping, 1992 threads   02:37:33
Load Avg: 1.43, 1.33, 1.54  CPU usage: 2.86% user, 4.89% sys, 92.24% idle
SharedLibs: 194M resident, 46M data, 27M linkedit.
MemRegions: 148682 total, 4085M resident, 98M private, 2875M shared.
PhysMem: 16G used (2884M wired), 139M unused.
VM: 5492G vsize, 627M framework vsize, 4224174(0) swapins, 4891071(0) swapouts.
Networks: packets: 68520971/78G in, 48325031/7782M out.
Disks: 4200303/96G read, 5885948/184G written.
primary key [-pid]: -mem
````

就变成了：

````
PID    COMMAND      %CPU  TIME     #TH    #WQ  #PORTS MEM    PURG   CMPRS  PGRP
0      kernel_task  7.8   09:09:00 142/8  0    2      1499M  0B     0B     0
601    com.docker.h 1.0   05:44:25 18     1    38     983M   0B     613M   595
337    Finder       0.0   15:27.13 10     2    488    862M   0B     253M   337
12843  Google Chrom 0.0   15:03.63 12     1    103    329M   0B     38M    308
92976  Preview      0.0   03:48.82 4      1    303    286M   1280K  700M   92976
308    Google Chrom 0.1   07:22:46 42     3    1855   265M   0B     171M   308
51162  NeteaseMusic 0.1   23:27.60 18     1    50568  239M   36K    275M   51162
219    WindowServer 1.1   06:56:09 6      2    596    223M   34M    346M   219
407    Google Chrom 0.3   04:12:31 8      2    410    122M   8192K  170M   308
47802  Twitter      0.1   00:50.21 22     4    4804   91M    3768K  139M   47802
````

3. kill/killall:终止进程

killall命令非常强大，它支持通过进程名而不是PID来结束进程。killall命令也支持通配符，这在系统因负载过大而变得很慢时很有用。

````
 # killall http*
````   
上例中的命令结束了所有以http开头的进程，比如Apache Web服务器的httpd服务。


4. 监控磁盘空间
+ 挂载存储媒体：mount命令

mount -t type device directory

+ 卸载设备：umount命令

+ df命令
df命令会显示每个有数据的已挂载文件系统.

df命令有一些命令行参数可用，但基本上不会用到。一个常用的参数是-h。它会把输出中 的磁盘空间按照用户易读的形式显示，通常用M来替代兆字节，用G替代吉字节。

+ du命令
默认情况下，du命令会显示当前目录下所有的文件、目录和子目录的磁盘使用情况，它会以 3 磁盘块为单位来表明每个文件或目录占用了多大存储空间。对标准大小的目录来说，这个输出会是一个比较长的列表

下面是能让du命令用起来更方便的几个命令行参数。
 -c:显示所有已列出文件总的大小。
 -h:按用户易读的格式输出大小，即用K替代千字节，用M替代兆字节，用G替代吉字
         节。
 -s:显示每个输出参数的总计。

5. 处理数据文件
+ 排序数据：sort
sort默认按照会话指定的语言的排序规则对文本中的 **数据行** 按照字符串排序，如果想按照数字排序要加`-n`参数。

如果不想按照行排序而想自己指定分隔符可以使用`-t`参数,然后用`-k`参数指定排序开始的位置：

| 单破折线 | 双破折线     |描述|
| :------------- | :------------- |:------------- |
|   -k     |  --key=POS1[,POS2]     |排序从POS1位置开始;如果指定了POS2的话，到POS2位置结束|
|     -t   |   --field-separator=SEP    |指定一个用来区分键位置的字符|

比如：`sort -t ':' -k 3 -n /etc/passwd`

+ 搜索数据:grep
grep命令的命令行格式如下。

grep [options] pattern [file]

如果要进行反向搜索(输出不匹配该模式的行)，可加-v参数。
    $ grep -v t file1
    one
    four
    five
$ 如果要显示匹配模式的行所在的行号，可加-n参数。
    $ grep -n t file1
    2:two
    3:three
    $
如果只要知道有多少行含有匹配的模式，可用-c参数。
    $ grep -c t file1
    2
    $
如果要指定多个匹配模式，可用-e参数来指定每个模式。
    $ grep -e t -e f file1
    two
    three
    four
    five
    $
这个例子输出了含有字符t或字符f的所有行。

+ 压缩数据

| 工具 | 文件扩展名     |描述|
| :------------- | :------------- |:------------- |
|   bzip2    | .bz2       |采用Burrows-Wheeler块排序文本压缩算法和霍夫曼编码|
|    compress   |     .Z   |最初的Unix文件压缩工具，已经快没人用了|
|     gzip  |   .gz     ||
|     zip  |     .zip   ||

gzip是Linux上最流行的压缩工具。这个软件包含有下面的工具。

 gzip:用来压缩文件。
 gzcat:用来查看压缩过的文本文件的内容。
 gunzip:用来解压文件。

gzip命令会压缩你在命令行指定的文件。也可以在命令行指定多个文件名甚至用通配符来一次性批量压缩文件。gzip命令会压缩该目录中匹配通配符的每个文件。

6. 归档数据:tar
tar命令最开始是用来将文件写到磁带设备上归档的，然而它也能把输出写到文件里，这种 用法在Linux上已经普遍用来归档数据了。

下面是tar命令的格式:
tar function [options] object1 object2 ...

function参数定义了tar命令应该做什么，如表4-8所示。

表4-8 tar命令的功能

| 功能     | 长名称    |描述|
| :------------- | :------------- |:------------- |
|    -A    |    --concatenate     |将一个已有tar归档文件追加到另一个已有tar归档文件 |
|    -c    |    --create    |创建一个新的tar归档文件|
|     -d   |    --diff   |检查归档文件和文件系统的不同之处|
|        |   --delete     |从已有tar归档文件中删除|
|    -r    |    --append   |追加文件到已有tar归档文件末尾|
|    -t    |    --list   |列出已有tar归档文件的内容|
|     -u   |      --update |将比tar归档文件中已有的同名文件新的文件追加到该tar归档文件中|
|     -x   |   --extract    |从已有tar归档文件中提取文件|
|        |       ||
|        |       ||

每个功能可用选项来针对tar归档文件定义一个特定行为。表4-9列出了这些选项中能和tar 命令一起使用的常见选项。

表4-9 tar命令选项

| 选项     | 描述    |
| :------------- | :------------- |
|    -C dir   |   切换到指定目录     |
|     -f file  |   输出结果到文件或设备file     |
|   -j    |    将输出重定向给bzip2命令来压缩内容    |
|    -p   |    保留所有文件权限    |
|     -v  |     在处理文件时显示文件   |
|     -z  |    将输出重定向给gzip命令来压缩内容    |

这些选项经常合并到一起使用。首先，你可以用下列命令来创建一个归档文件:
tar -cvf test.tar test/ test2/

上面的命令创建了名为test.tar的归档文件，含有test和test2目录内容。接着，

用下列命令: tar -tf test.tar
列出tar文件test.tar的内容(但并不提取文件)。

最后，用命令: tar -xvf test.tar

通过这一命令从tar文件test.tar中提取内容。如果tar文件是从一个目录结构创建的，那整个目 录结构都会在当前目录下重新创建。

## 5. 理解Shell

## 6.环境变量
### 6.1 全局变量
可以使用env 或 printenv 命令查看全局变量。

要显示个 环境变量的 ，可以使用printenv命令，但是不要用env命令。

````
$ printenv HOME /home/Christine $
$ env HOME
    env: HOME: No such file or directory   
````

也可以使用echo显示变量的值。在这种下引用某个环境变量的时候，必须在变量前面加上一个`$`。

````
$ echo $HOME /home/Christine $
````

### 设置全局变量
在设定全局环境变量的进程所创建的子进程中， 变量都是可见的。创建全局环境变量的方法是 创建一个局部环境变量，然后在把它导出到全局环境中。
这个过程通过 export 命令来完成，变量名前面不需要加$。

````
$ my_variable="I am Global now" $
$ export my_variable
````

### 删除全局变量
unset 指令
