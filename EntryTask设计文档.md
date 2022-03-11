# EntryTask设计文档

## 项目来源

### 背景介绍

实现一个用户管理系统，用户可以登录并编辑他们的个人资料。用户可以登录网页。

用户输入用户名和密码后，后端系统将验证其身份。如果登录成功，将显示相关用户信息，否则将显示错误消息。

成功登录后，用户可以编辑以下信息：

* 上传一张照片作为他/她的个人资料照片

* 更改他/她的昵称（支持utf-8编码的unicode字符）

* 用户信息包括：用户名（不能更改）、昵称、个人资料图片。对于测试，初始用户数据可以直接插入数据库。确保测试数据库中至少有1000万个用户帐户。

### 项目要求

#### 时限

* 你需要在5个工作日内完成这项任务。

#### 要求

* 支持1000个并发http请求

1. 有了缓存层，

* 基本要求：

  * 每秒支持6000个登录请求

  * 来自至少200个独特的用户

  * 没有套接字/超时错误

* 挑战： 每秒支持20k个登录请求（可选，如果有时间，您可以尝试达到此目标）

2. 没有缓存层

* 基本要求：
  * 每秒支持3000个登录请求
  * 来自至少200个独特的用户
  * 没有套接字/超时错误

* 挑战： 每秒支持10k个登录请求（可选，如果有时间，您可以尝试达到此目标）

## 系统设计

#### 框架设计

* db
  * mysql connection
    * 使用go原生database/sql
    * 封装DB支持config配置
    * 通过Option模式初始化DB
  * mysql orm
    * sql防注入，语句Prepare后通过stmt执行
    * 通过反射，支持自定义结构体操作db
    * 实现常用函数Insert、Select、Where等方便业务操作mysql

* upload图片系统
  * upload图片上传
    * 统一封装uploadHandler
      * 方便各业务直接接入图片上传
      * 方便后续统一对图片做处理（压缩、裁剪等）
    * 统一图片资源校验
      * 文件格式检查
      * 使用md5作为文件名，减少相同文件多次上传造成的存储资源浪费
  * show图片展示
    * 统一封装showHandler
      * 方便各业务直接接入图片展示
      * 方便后续统一对图片做处理（压缩、裁剪等）
    * 检查请求路径，避免重要文件路径被访问，只允许访问图片存储路径

* rpc框架

  * 接口

    * 类图

       ![image-20220308114139193](/Users/ike.zhang/Library/Application Support/typora-user-images/image-20220308114139193.png)

    * iserver服务端接口

      ```go
      type IServer interface {
        // 服务启动
      	Start()
        // 服务停止
      	Stop()
        // 服务运行
      	Serve()
        // 注册服务handler
      	RegisterService(sd *ServiceDesc, ss interface{})
      }
      ```

    * iconnection服务端连接接口

      ```go
      type IConnection interface {
        // 连接启动
      	Start()
        // 链接停止
      	Stop()
        // 获取远程地址
      	RemoteAddr() net.Addr
      }
      ```

    * iclient客户端接口

      ```go
      type IClient interface {
        // 传输数据
      	Write(data []byte) error
        // 获取数据
      	Read() ([]byte, error)
      }
      ```

  * 架构模型

    ![image-20220308115823270](/Users/ike.zhang/Library/Application Support/typora-user-images/image-20220308115823270.png)

    * Server结合connection、router实现TCP服务端
      * Server
        * Resolve and Listen TCP
        * wait client Accept
      * connection和客户端交互的连接实例
        * 当TCP Srv 收到客户端Accept创建一个connection
        * connection从net.Conn中解包获取数据包
        * 处理客户端保活PING消息
        * 正常接口请求从router中获取handler实例并执行
        * 将handler结果回写客户端
      * package
        * encode
          * 计算数据包长度(int32)，并将长度写入包头
          * 将包头+数据包组合
        * decode
          * 从tcp conn中预读取4字节(int32)，使用Peek不会实际读取Reader中的内容
          * 处理预读内容，获得数据包长度
          * 从Reader中获取数据包长度+4字节的数据
          * 返回剔除包头的数据部分
      * router
        * 维护已注册的service列表(map[服务名]服务信息)
        * 维护已注册的service实例
        * 维护service实例方法及handler的关系
    * Client结合Instance实现TCP客户端
      * instance
        * instance
          * 封装net.Conn的Read、Write
          * 实现healthCheck进行客户端链接保活
          * Close将使用完的instance放归到client中
        * client
          * 进程锁+connCnt控制链接池中链接总数
          * 使用有buffer的channel实现instance池
          * 发起请求前先获取instance如果未从channel中获取到并且连接总量小于配置的最大连接，则创建一个新连接
          * 使用获取到的instance发送消息给Server端
          * 解析Server回复的消息，并返回给Client调用者

* web框架

  * router
    * 封装原生http router
    * 接受web handler
  * web handler
    * 实现GetPath()\GetHandlerFunc()两个方法
    * 通过配置的path和handler对应关系，注册到router中
  * middleware
    * 用户态验证
    * 跨域解决
    * 请求时间记录
    * metrics上传

#### 业务架构设计

##### 服务端

* 数据库

  ```sql
  /* 用户账密表 */
  CREATE TABLE `user` (
    `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
    `username` varchar(100) NOT NULL,
    `password` varchar(100) NOT NULL,
    `uid` bigint(11) NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `username` (`username`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  
  /* 用户信息表 */
  CREATE TABLE `user_info` (
    `id` bigint(11) NOT NULL,
    `username` varchar(100) NOT NULL,
    `nickname` varchar(100) NOT NULL,
    `photo` varchar(512) NOT NULL,
    `create_at` int(11) NOT NULL,
    `update_at` int(11) NOT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  ```

* 接口

  * Login：用户登录接口，提供账密验证返回登录态token
  * Valid：验证用户登录态是否合法
  * Info：用户信息获取
  * UpdateInfo：修改用户信息

##### web服务端

* 路由
  * /login: 登录handler，调用tcp服务端Login。并将token写入cookie
  * /upload：图片上传，接入框架的upload handler
  * /uploaded：图片展示，接入框架的show handler
  * /user：用户信息查询，调用tcp服务端Info
  * /updateuser：用户信息修改，调用tcp服务端UpdateInfo
* 中间件
  * 业务执行前
    * before：记录进入接口时间
    * metrics：记录接口请求
    * cros：处理跨域
    * clearHeader：清理x-rpc-token等header
    * valid：验证用户登录态，使用cookie中的token调用tcp服务端Valid
  * 业务执行后
    * after：记录请求历时并上报metric

##### 前端

* 路由
  * /：入口页面
  * /detail：用户详情页
* Home：
  * 用户登录页面，调用web服务端login
* Detail：
  * 用户信息页面，调用web服务端user、updateuser
* Info：
  * 用户信息展示动态加载

## 安装维护

#### tcp服务端

* 配置文件config.yml
* 二进制entrytasksrv

#### web服务端

	* 二进制entrytaskweb

#### 前端

* 使用npm部署

## 性能测试报告

#### 1000VUs情况下，限制3000qps，持续压测5分钟。

![image-20220308143318698](/Users/ike.zhang/Library/Application Support/typora-user-images/image-20220308143318698.png)

![image-20220308143437066](/Users/ike.zhang/Library/Application Support/typora-user-images/image-20220308143437066.png)

## 总结报告

### 收获

* tcp链路流程
* service和proto结合注册并完成handler实例框架
* orm开发，及其中用到的反射等
* react从0-1的项目开发

### 不足及展望

* token过于简单使用了aes加密。未来可以替换为JWT
* protobuf服务代码生成未自动化，未来可以增加增加grpc代码生成插件
* 参数未完全配置化，未来全config.yml管理配置
* 缺失服务注册发现，client需要配置server的ip和端口
* 用户态token未持久化，未来可引入redis存储token(通过uuid等实现)
* TCP服务端中间件缺失，需要在接口内调用登录态验证函数
* ……

