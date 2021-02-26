### 概述
JAVA的IO大概可以分为以下几类：
* 磁盘操作：File
* 字节操作：InputStream 和 OutputStream
* 字符操作：Reader 和 Writer
* 对象操作：Serializable
* 网络操作：Socket
* 新的输入输出：NIO
***  
### 磁盘操作
File 类可以用来表示文件和目录信息，但是他不表示文件的内容
从 JAVA 7 开始，可以使用 Paths 和 Files 代替 File

***
### 字节操作
#### 装饰者模式
JAVA IO 使用了装饰者模式来实现，以 InputStream 为例：
* InputStream 是抽象组件
* FileInputStream 是 InputStream 的子类，