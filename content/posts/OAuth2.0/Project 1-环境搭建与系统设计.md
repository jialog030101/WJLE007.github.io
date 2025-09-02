---
title: Project 1-环境搭建与系统设计
date: 2025-07-08
categories:
  - Java
tags:
  - OAuth
---
# 实验目标

本节主要是搭建开发环境并完成系统整体设计，为后续实验打下基础。

# 详细任务

## 环境搭建

### 1.安装JDK17

**步骤：​**​
1. 访问[Java Archive Downloads - Java SE 17.0.12 and earlier](https://www.oracle.com/java/technologies/javase/jdk17-archive-downloads.html) 或 OpenJDK 17(自行寻找网站下载)
2. 下载对应操作系统的安装包（Windows选.exe，macOS选.dmg，Linux选.tar.gz）
3. ​**Windows/macOS**​：运行安装向导，按提示完成安装  
    ​**Linux**​：
```bash
tar -xzf jdk-17_linux-x64_bin.tar.gz
sudo mv jdk-17 /opt/
 ```
4. 配置环境变量：
    - ​**Windows**​：
        - 新增系统变量 `JAVA_HOME` = `C:\Program Files\Java\jdk-17`
        - 编辑 `Path` 添加 `%JAVA_HOME%\bin`
    - ​**macOS/Linux**​：
```bash
echo 'export JAVA_HOME=/opt/jdk-17' >> ~/.bashrc  # 或 ~/.zshrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
 ```
        
5. 验证：
```bash
 java -version  # 应显示 "java version "17.x.x"
```

### 2.安装Maven3.8+

**步骤：​**​

1. 访问 [Download Apache Maven – Maven](https://maven.apache.org/download.cgi)
2. 下载 Binary zip 包（如 `apache-maven-3.8.6-bin.zip`）
3. 解压到本地目录（如 `C:\Program Files\apache-maven-3.8.6` 或 `/opt/maven`）
4. 配置环境变量
- Windows
	- 新增 `MAVEN_HOME` = `C:\Program Files\apache-maven-3.8.6`
	- 编辑 `Path` 添加 `%MAVEN_HOME%\bin`
- macOS/Linux
```shell
echo 'export MAVEN_HOME=/opt/maven' >> ~/.bashrc
echo 'export PATH=$MAVEN_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```
5. 验证
```shell
mvn -v  # 应显示 Apache Maven 3.8.x
```

### 3.安装 MySQL 8.0
**步骤**：

1. 访问 [MySQL :: Download MySQL Community Server](https://dev.mysql.com/downloads/mysql/)
2. 下载对应操作系统的安装包
3. ​**Windows**​：
    - 运行安装向导，选择 "Developer Default"
    - 设置 root 密码（务必牢记）
    - 端口保持默认 `3306`

### 4.安装Redis

**步骤：​**​

1. ​**Windows**​：
    - 下载Redis [Title Unavailable \| Site Unreachable](https://redis.io/docs/latest/operate/oss_and_stack/install/archive/install-redis/)
    - 运行安装程序，默认端口 `6379`
2. macOS
```shell
brew install redis
brew services start redis
```

3.Linux
 
```shell
sudo apt install redis-server
sudo systemctl enable redis
```
4.验证
```shell
redis-cli ping  # 返回 "PONG" 表示成功
```

### 5.配置IDEA

**IntelliJ IDEA 推荐步骤：​**​

1. 下载 IntelliJ IDEA Community（免费）或 Ultimate 版
2. 安装后启动，配置 JDK：
    - `File > Project Structure > SDKs > + > JDK` 选择 JDK 17 安装路径
3. 配置 Maven：
    - `Settings > Build, Execution, Deployment > Maven`
    - 设置 `Maven home path` 为你的 Maven 安装目录
    - 勾选 `Override` 指定 `settings.xml`（如有需要）
4. 安装插件（可选）：
    - Database Tools（连接MySQL）
    - Redis（连接Redis）

IDEA的无限制使用的方法请大家通过自己的信息检索能力获取，不过免费版的应该也是够用的。

### 6.测试环境可用

```shell
https://github.com/jialog030101/OAuth2.0.git
```

使用git将项目下载到本地之后checkout到project1分支，运行demo

访问 http://localhost:8080/api/hello 看到如下页面
![PixPin_2025-07-08_23-23-35.png|650](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20250708232342262.png)

访问 http://localhost:8080/api/env 之后看到如下页面
![PixPin_2025-07-08_23-24-42.png|650](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20250708232444549.png)

到此为止环境的搭建已经结束，现在为止已经具备了完成实验的基础了

## OAuth2.0概念学习

- 学习OAuth2.0基本概念和流程
- 理解授权码模式的工作原理
- 分析Token的生成、验证和刷新机制
- 了解单点登录和单点退出的实现方式

这一段概念比较抽象，同学们可能很多都没有接触过，所以我将通过一段视频带大家了解一下OAuth2.0的主要内容。剩下的细节部分需要同学们自己找学习资料进行学习。

[理解OAuth 2.0 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
