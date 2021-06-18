#### 添加用户

```
#添加用户 
#指定用户登入后所使用的shell。默认值为/bin/bash。
useradd es -s /bin/bash
```

#### 解压文件

```
#解压文件
tar -zxvf elasticsearch-6.2.4.tar.gz

-z 支持gzip解压文件
-x 从压缩的文件中提取文件
-v 显示操作过程
-f 指定文件名，这个参数后面必须跟一个文件

-c 建立新的压缩文件
```

#### 快速复制一行

```
yy+p
```

#### 查看进程、杀死进程

```
ps aux|grep elastic
kill -9 pid

ps:查找与进程相关的PID
ps a 显示现行终端机下的所有程序，包括其他用户的程序。
pa u 以用户为主的格式来显示程序状况
ps x 显示所有程序，不以终端机来区分
```

