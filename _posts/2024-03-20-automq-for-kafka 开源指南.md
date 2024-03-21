---
layout: mypost
title: automq-for-kafka 开源指南
categories: [开源, AutoMQ, AWS]
---
## 1.awscli + Docker DeskTop + LocalStack 搭建本地 AWS 开发环境

### 1.什么是 LocalStack

Localstack是一个功能齐全的本地云堆栈。它有助于在本地开发和测试云和无服务应用程序。

LocalStack 以 Docker 镜像的形式提供服务，本质是一个 Python 应用程序，以 HTTP 请求的形式运行在某个端口。

### 2.拉起 LocalStack 服务

首先需要安装 Docker，并基于 Docker DeskTop Extension 管理 LocalStack。

LocalStack 会以 Docker 容器形式运行在宿主机上。

### 3.使用 awscli 访问 LocalStack

AWS 命令行界面（AWS CLI）是用于管理 AWS 产品的统一工具。只需要下载和配置一个工具，我们就可以使用命令行控制多个 AWS 产品并利用脚本来自动执行这些服务（类似于 client 之于 kubernetes）。

在使用 awscli 访问 LocalStack 之前，需要执行 aws configure 进行令牌配置（用于获取访问 AWS 云的权限），一般配置如下：

```bash
trueman@truemandeMacBook-Air ~ % aws configure
AWS Access Key ID [****************test]: auto-kafka-test
AWS Secret Access Key [****************test]: auto-kafka-test
Default region name [us-east-1]:
Default output format [None]:
```

之后访问 LocalStack 服务的时候，需要添加 `--endpoint=http://127.0.0.1:4566` 参数。

### 4.常见的 aws cli api

列出所有的 s3 资源

```bash
trueman@truemandeMacBook-Air ~ % aws s3api list-buckets --endpoint=http://127.0.0.1:4566
{
    "Buckets": [
        {
            "Name": "ko3",
            "CreationDate": "2024-03-20T03:17:05+00:00"
        }
    ],
    "Owner": {
        "DisplayName": "webfile",
        "ID": "75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a"
    }
}
```

查看 s3 中某一个 bucket 的全部对象

```bash
trueman@truemandeMacBook-Air ~ % aws s3 ls s3://ko3 --recursive --endpoint=http://127.0.0.1:4566
2024-03-20 14:05:48     142729 00000000/_kafka_OrRRu50xSySpQKN0P1IDbg/0
2024-03-20 14:42:27       6458 10000000/_kafka_OrRRu50xSySpQKN0P1IDbg/1
2024-03-20 14:45:38       4013 20000000/_kafka_OrRRu50xSySpQKN0P1IDbg/2
2024-03-20 15:13:52        291 30000000/_kafka_OrRRu50xSySpQKN0P1IDbg/3
2024-03-20 15:15:29       4502 40000000/_kafka_OrRRu50xSySpQKN0P1IDbg/4
2024-03-20 15:39:33        572 50000000/_kafka_OrRRu50xSySpQKN0P1IDbg/5
2024-03-20 15:41:11       6525 60000000/_kafka_OrRRu50xSySpQKN0P1IDbg/6
2024-03-20 15:47:43       5617 70000000/_kafka_OrRRu50xSySpQKN0P1IDbg/7
```

## 2.AutoMQ Quick Start

### 1.基础信息

仓库：https://github.com/AutoMQ/automq-for-kafka

分支：主分支为 develop

语言：Kafka 基于 java 和 scala 开发，在开始前，确保语言环境ready。语言版本要求：
- java：17 
- scala：2.13

### 2.管理工具

Kafka 使用 gradle 作为工程管理工具。gradle 工程的管理基于 groovy 语法编写的脚本，在 kafka 项目中，项目管理配置主要就是根目录下的 build.gradle，其作用类似于 maven 工程下的根 pom。

Gradle 也支持每个模块各自配置 build.gradle，但 kafka 没有这么做，所有模块都由根目录下的 build.gradle 管理。
安装

不推荐手动安装 gradle。根目录下的 gradlew 脚本会自动帮我们下载 gradle，版本也是gradlew 脚本指定的。

### 3.国内代理

Gradle 仓库国内需要配置代理，在当前用户目录的.gradle 目录下，新增 init.gradle 文件：

```gradle
allprojects {
    repositories {
        mavenLocal()
        maven { url 'https://maven.aliyun.com/repository/central'}
        maven { url 'https://maven.aliyun.com/repository/public' }
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/spring' }
        maven { url 'https://maven.aliyun.com/repository/grails-core' }
        maven { url 'https://maven.aliyun.com/repository/apache-snapshots' }
        mavenCentral()
    }
}
```

配置实际上是依赖的搜索路径，意思是：
- 如果依赖包在本地 maven 缓存目录下有，则使用 maven 目录下的 jar 包
- Maven {xxx}: 相应 maven 仓库使用国内代理
- 如果上述还没有命中，则使用 maven 中心仓库查找

另外，s3 stream 放在了https://s01.oss.sonatype.org/content/repositories/snapshots/，国内没有mirror，需要自行代理。

### 4.编译

```bash
./gradlew jar -x test
```

我们可以把 gradlew 等效为 gradle 程序使用。-x test 用于跳过 kafka 的测试。


### 5.idea本地启动

依赖：需要有个 s3 服务，可以使用 localstack 模拟。记得提前创好一个bucket，以 localstack 为例 ：

```bash
aws s3api create-bucket --bucket ko3 --endpoint=http://127.0.0.1:4566
```

Config 配置：修改 config/kraft/server.properties文件。至少需要修改以下配置：

```properties
s3.endpoint=https://s3.amazonaws.com

s3.region=us-east-1

s3.bucket=ko3
```

如果使用localstack，s3.endpoint 务必填写 http://127.0.0.1:4566，不要写 localhost。区域填 us-east-1。bucket 与上面创建的 bucket 一致。

### 6.Format 路径
生成 Cluster UUID：
```bash
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
```

Format 元数据目录：
```bash
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
```

### 7.启动配置
启动入口：core/src/main/scala/kafka/Kafka.scala

具体的 idea 启动配置：

```md
# class path
-cp automq-for-kafka.core.main

# jvm 参数
-Xmx1G
-Xms1G
-server
-XX:+UseZGC
-XX:MaxDirectMemorySize=2G
-Dkafka.logs.dir=logs/
-Dlog4j.configuration=file:config/log4j.properties
-Dio.netty.leakDetection.level=paranoid

# 程序输入参数
"config/kraft/server.properties"

# ENV，需要提供 AK 和 SK 信息以供连接 S3 服务。
KAFKA_S3_ACCESS_KEY=test;KAFKA_S3_SECRET_KEY=test
```

### 7.启动
idea 中启动

### 8.关闭
如果不是以 debug 模式启动，可以通过 idea 界面优雅关闭 kafka 进程。

也可以通过命令优雅关闭 kafka 进程：

```bash
jcmd | grep -e kafka.Kafka | awk '{print $1}' | xargs kill
```

## 3.Feat(core) 日志功能

关键，需要保证 log4j 的日志生成路径和环境变量 LOG_DIR 是一致的。

## 4.Checkstyle

Checkstyle是一个开源的代码规范检查工具，主要用于Java代码。它可以帮助开发者确保他们的代码遵循一定的编码规范和标准。Checkstyle可以检查许多方面的代码规范，例如：

* 缩进和格式化
* 命名规范
* Javadoc注释
* 代码复杂度
* 代码的长度（行数或字符数）
* 导入的控制
* 空白字符的使用
* 代码的结构（例如，是否有过多的方法或类）

Checkstyle是一个非常灵活的工具，可以通过XML配置文件来定制检查的规则。这使得它可以适应各种不同的编码规范和标准。

AutoMQ-for-Kafka 在审查 PR 时会基于 Checkstyle 检查 Java 代码规范。

## 5.SpotBugs

SpotBugs，一个用于检查 Java 代码中可能的 bug 的静态分析工具。可以使用命令 `gradle spotbugsMain` 来执行 SpotBugs 命令，执行结果会被保存在 HTML 文件中。

> gradle 可以集成 SpotBugs。
