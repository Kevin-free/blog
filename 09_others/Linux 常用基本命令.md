## Linux 常用基本命令

### 命令格式

​		命令  [-选项]  [参数]  

例		ls	  - la		/etc	

说明：

1. 个别命令使用不遵循此规则
2. 当有多哥选项时，可以写在一起
3. 简化选项与完整选项 -a 等于 --all
4. [] 表示可省



### 重要的几个热键

【tab】-- 具有【命令补全】与【档案补全】的功能

【ctrl + c】-- 让当前的程序【停掉】

【ctrl + d】 --  通常代表着：『键盘输入结束(End Of File, EOF 戒 End OfInput)』的意思；另外，他也可以用来取代exit 

【ctrl + l】-- 等价于命令`clear`，用于清屏



### 查看日志

#### **第一种:查看实时变化的日志(比较吃内存)**

**最常用的:**

**tail -f filename (默认最后10行,相当于增加参数 -n 10)**

**Ctrl+c 是退出tail命令**



其他情况:

tail -n 20 filename (显示filename最后20行)

tail -n +5 filename (从第5行开始显示文件)



#### **第二种:搜索关键字附近的日志**

**最常用的:**

**cat -n filename |grep "keyword"**



其他情况:

cat filename | grep -C 5 '关键字' (显示日志里匹配字串那行以及前后5行)

cat filename | grep -B 5 '关键字' (显示匹配字串及前5行)

cat filename | grep -A 5 '关键字' (显示匹配字串及后5行)



#### **第三种:进入编辑查找:vi(vim)(大文件不适合)** 

**1、进入vim编辑模式:vim filename**

**2、输入“/关键字”,按enter键查找**

**3、查找下一个,按“n”即可**

**退出:按ESC键后,接着再输入:号时,vi会在屏幕的最下方等待我们输入命令**

**wq! 保存退出;**

**q! 不保存退出;**



 **其他情况:**

**/关键字**  注:正向查找,按n键把光标移动到下一个符合条件的地方
**?关键字**  注:反向查找,按shift+n 键,把光标移动到下一个符合条件的**



### 查看端口占用情况

#### netstat

**netstat -tunlp** 用于显示 tcp，udp 的端口和进程等相关情况。

netstat 查看端口占用语法格式：

```sh
netstat -tunlp | grep 端口号
```

- -t (tcp) 仅显示tcp相关选项
- -u (udp)仅显示udp相关选项
- -n 拒绝显示别名，能显示数字的全部转化为数字
- -l 仅列出在Listen(监听)的服务状态
- -p 显示建立相关链接的程序名

例如查看 8000 端口的情况，使用以下命令：

```sh
# netstat -tunlp | grep 8000
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      26993/nodejs   
```

更多命令：

```sh
netstat -ntlp   //查看当前所有tcp端口
netstat -ntulp | grep 80   //查看所有80端口使用情况
netstat -ntulp | grep 3306   //查看所有3306端口使用情况
```

------

#### kill

在查到端口占用的进程后，如果你要杀掉对应的进程可以使用 kill 命令：

```sh
kill -9 PID
```

如上实例，我们看到 8000 端口对应的 PID 为 26993，使用以下命令杀死进程：

```sh
kill -9 26993
```



### ssh

#### 说明

加密的网络协议，提供客户-服务模式。

#### 格式

ssh username@ip 

> ssh root@47.101.38.56

![image-20200525203847304](/Users/zonst/Library/Application Support/typora-user-images/image-20200525203847304.png)

#### 参数说明

ssh ip #不提供用户名默认使用当前用户的用户名

-C：数据在传输过程中被压缩

-p + 端口号：指定端口号（默认端口号22）

-v：调试模式

-F+文件路径：使用指定的配置文件



### scp

和cp一样用来实现文件的复制功能，但是主要是用在不用的Linux系统之间。有security的文件复制，基于ssh登录

- cp [options] file_source file_target

  -r：原有目录，递归复制

  -f：force，覆盖同名目录

- scp [options] file_source file_target

  -v：显示状态

  -C：使用压缩

  -P：选择端口

> 从本地到远端 scp /file/from.txt username@ip:/file/to.txt
>
> 从远端到本地 scp username@ip:/file/from.txt /file/to.txt



### grep

#### 说明

grep (global search regular expression(RE) and print out the line, 全面搜索正则表达式并把行打印出来) 是一种强大的文本搜索工具，它能使用正则表达式搜索文本，并把匹配的行打印出来。

#### 格式

grep [options] str_to_search filename

#### 参数说明

-a：将二进制文件以text文件的方式查找

-c：输出匹配行的行数

-i：忽略大小写

-n：在输出时输出行号

-v：反向选择，输出没有匹配的行

str_to_search 可以使用正则表达式

​		[ ^abc ]：反向选择，不能是a，b，c

​		[abc]：abc中的一个

​		[a-z1-9]：a-z和1-9中的一个

​		^：行首

​		$：行尾

​		.：任意一个字符

​		*：任意多个字符，包括零个

​		{}：限定重复次数，'{', '}'两个符号在shell中特别的含义，所以要用’\'使其失去特殊含义

​		o \ { 2 \ }：o重复两次

​		o \ { 2,5 \ }：o重复两次到五次

​		o \ { 2, \ }：o重复两次以上



### find

#### 说明：

在指定目录下根据指定规则查找文件

#### 格式：

find [dir dir ...] [options] [exec commad {} \;]

#### 参数说明：

-name <文件名>：要查找到文件名，可以使用glob规则

​		glob：’*‘统配任意字符

​					’?'统配任意一个字符

​					‘[abc]'统配括号中的任意一个字符

​					‘[a-c1-9]'统配括号内字符范围中的任意一个字符

-atime n|-n|+n：最后访问时间等于n｜小于n｜大于n

-mtime n|-n|+n：最后修改内容时间

-ctime n|-n|+n：最后修改属性时间（也就是i节点信息修改时间）

-type：根据文件类型查找

​		f：普通文件

​		d：目录

​		l：链接

​		b：块设备文件

​		c：字符设备

​		p：管道

​		s：socket文件

-size 2M|+2M|-2M：根据大小找文件（大小为2M｜大于2M｜小于2M）

### cat

#### 说明

用于连接文件并打印到标准输出设备上

#### 格式

cat [-AbeEnstTuv] [--help] [--version] filename

#### 参数说明

-n 或 --number：由1开始对所有输出到行数编号

-b 或 --number-nonblank：和 -n 相似，只不过对空白行不编号

-s 或 --squeeze-blank：当遇到有连续两行以上的空白行，就代换为一行的空白行

-v 或 --show-nonprinting：使用 ^ 和 M- 符号，除了 LFD 和 TAB 之外

-E 或 --show-ends：在每行结束处显示$

-T 或 --show-tabs：将 TAB 字符显示为^l

-A 或 --show-all：等价于 -vET

-e：等价于 -vE

-t：等价于 -vT

#### 实例

把 textfile1 的文档内容加上行号后输入 textfile2 这个文档里

> cat -n textfile1 > textfile2

把 textfile1 和 textfile2 的文档内容加上行号（空白行不加）之后将内容附加到 textfile3 文档里

> cat -b textfile1 textfile2 >> textfile3

清空 /etc/test.txt 内容

> cat /dev/null > /etc/test.txt

cat 也可用来制作镜像文件。例如要制作软盘的镜像文件，将软盘放好输入

> cat /dev/fd0 > OUTFILE

相反的，如果想要把image file写到软盘，输入

> cat IMG_FILE > /dev/fd0

注：

- OUTFILE 指输出的镜像文件名
- IMG_FILE 值镜像文件
- 若从镜像文件写回device时，device容量需与相当
- 通常制作开机磁片

### vi/vim

Vim是从 vi 发展出来的一个文本编辑器。

vim键盘图

![img](https://www.runoob.com/wp-content/uploads/2015/10/vi-vim-cheat-sheet-sch.gif)





## Sed



用sed命令可以批量替换多个文件中的 字符串。 

````
sed -i "s/原字符串/新字符串/g" `grep 原字符串 -rl 所在目录`
````

例如：我要把mahuinan替换 为huinanma，执行命令： 

```
sed -i "s/mahuinan/huinanma/g" `grep mahuinan -rl /www`
```



替换带有特殊字符 "/" 的字符串，在 / 前加 \ 

```
sed -i 's/data\/nfs\/config\/1/data\/nfs\/config\/3/g' `grep "data/nfs/config/1" -rl ./3`
```









































