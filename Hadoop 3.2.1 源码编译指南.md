## 一、文档概述

本文档针对 **ARM64（aarch64）架构** 环境，提供 Hadoop 3.2.1 源码编译（含 native 库）的完整步骤，包括环境准备、依赖安装、源码编译、问题排查、打包部署全流程。

## 二、环境准备

### 硬件

|  项目  |  说明          |
| -------- | ---------------- |
|  CPU   |  Kunpeng  920  |

### 操作系统

|  项目   |  版本  |
| --------- | -------- |
|  HCE0S  |  2.0   |

### 软件信息

|  项目         |  版本                      |  下载地址                                                         |
| --------------- | ---------------------------- | ------------------------------------------------------------------- |
|  毕昇融合JDK  |  1.8.0_462-b22  (21.0.8)  |  [Link](https://www.hikunpeng.com/zh/developer/devkit/download/jdk)  |
|  Maven        |  3.6.3                     |  参考下面安装说明                                                 |
|  Protoc       |  2.5.0                     |  参考下面安装说明                                                 
|  hadoop       |  3.2.1                     |        参考下面安装说明                                                            |

### 2.3 基础工具安装

先安装系统基础工具（编译依赖、版本控制、网络工具等）：

```bash
sudo yum install -y gcc gcc-c++ make cmake git wget curl openssl-devel zlib-devel which
```

## 三、核心依赖安装（ARM64 专属）

Hadoop 编译依赖 Protobuf、libtirpc、Boost、SASL 等工具，​**必须安装 ARM64 版本**​，避免架构不兼容。

### 3.1 Maven 安装（编译构建工具）

#### 3.1.1 下载并解压

bash

运行

```bash
# 下载 Maven 3.6.3（兼容 Hadoop 3.2.1）
wget https://archive.apache.org/dist/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz

# 解压到 /opt 目录
sudo tar -zxf apache-maven-3.6.3-bin.tar.gz -C /opt/

# 重命名（简化路径）
sudo mv /opt/apache-maven-3.6.3 /opt/maven
```

#### 3.1.2 配置环境变量

```bash
# 编辑环境变量文件
sudo vi /etc/profile

# 添加以下内容（末尾追加）
export M2_HOME=/opt/maven
export PATH=$M2_HOME/bin:$PATH

# 生效环境变量
source /etc/profile

# 验证（输出版本 3.6.3 即成功）
mvn -v
```

### 3.2 Protobuf 2.5.0 安装（序列化依赖）

Hadoop 3.2.1 强制依赖 Protobuf 2.5.0，且 ​**原生不支持 ARM64**​，需手动编译适配。

#### 3.2.1 下载源码

```bash
wget https://archive.apache.org/dist/protobuf/protobuf-2.5.0.tar.gz
tar -zxf protobuf-2.5.0.tar.gz
cd protobuf-2.5.0
```

#### 3.2.2 修改autogen.sh文件的第20-22行：

```bash
vi   autogen.sh

# 替换成如下内容：

curl -L   https://github.com/google/googletest/archive/release-1.5.0.tar.gz | tar zx

mv googletest-release-1.5.0 gtest
```


#### 3.2.3 打ARM补丁

下载protoc.zip并解压得到protoc.patch文件，其中protoc.patch存放的路径可自己指定。

```bash
wget https://mirrors.huaweicloud.com/kunpeng/archive/kunpeng_solution/bigdata/Patch/protoc.zip
unzip protoc.zip
cp   protoc.patch ./src/google/protobuf/stubs/
cd   ./src/google/protobuf/stubs/
patch   -p1 < protoc.patch
cd   -
```

#### 3.2.4 编译并安装到系统默认目录

```bash
./autogen.sh   && ./configure CFLAGS='-fsigned-char' && make && make   install
```

## 五、核心编译步骤

### 5.1 编译前配置（CMake 适配 ARM64）

修改 Hadoop 关键模块的 CMake 配置，确保 ARM64 架构兼容：

```bash
# 1. 修改 hadoop-pipes 模块的 CMakeLists.txt
vi /home/hadoop-3.2.1-src/hadoop-tools/hadoop-pipes/src/main/native/CMakeLists.txt

# （1）添加库搜索路径（确保找到 libtirpc）
在文件开头添加：
link_directories(/usr/lib64)
include_directories(/usr/include)

# （2）修复 target_link_libraries 语法（目标在前，依赖在后）
找到所有 target_link_libraries 配置，修改为：
target_link_libraries(hadoop-pipes
  PRIVATE
    tirpc
    boost_system
    boost_thread
    protobuf
    hadooputils
)
target_link_libraries(wordcount-simple hadooppipes hadooputils tirpc)
target_link_libraries(wordcount hadooppipes hadooputils tirpc)
target_link_libraries(pipes-simple hadooppipes hadooputils tirpc)

# （3）适配 ARM64 架构判断（搜索 x86_64 相关逻辑）
将：
if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
改为：
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|aarch64|arm64")

# 2. 修改 hadoop-hdfs-native-client 模块的 CMakeLists.txt
vi /home/hadoop-3.2.1-src/hadoop-hdfs-project/hadoop-hdfs-native-client/src/main/native/CMakeLists.txt

# （1）添加 Protobuf 库绝对路径（避免链接错误）
在 target_link_libraries(hdfs ...) 中添加：
target_link_libraries(hdfs
  PRIVATE
    /usr/local/protobuf-2.5.0/lib/libprotobuf.so
    ssl
    crypto
    z
    sasl2
)

# （2）适配 ARM64 架构判断（同上述步骤）
将 x86_64 专属判断改为支持 ARM64：
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|aarch64|arm64")
```

### 5.2 执行编译与打包

#### 5.2.1 全量编译（含 native 库，跳过非核心报错模块）

```bash
cd /home/hadoop-3.2.1-src

# 核心编译命令（多线程加速，跳过测试和文档）
mvn clean package -Pnative -DskipTests -Dmaven.javadoc.skip=true \
  -Dprotoc.path=/usr/local/protobuf-2.5.0/bin/protoc \
  -pl '!hadoop-tools/hadoop-pipes' \  # 跳过非核心报错模块（可选）
  -T 4C  # 4 线程编译（根据 CPU 核心数调整，如 8C 改为 -T 8C）
```

#### 5.2.2 关键参数说明

| 参数                                   | 作用                                       |
| ---------------------------------------- | -------------------------------------------- |
| `-Pnative`                         | 启用 native 库编译（C/C++ 模块）           |
| `-DskipTests`                      | 跳过单元测试（加速编译，避免测试报错）     |
| `-Dmaven.javadoc.skip=true`        | 跳过 Javadoc 生成（加速编译）              |
| `-Dprotoc.path=...`                | 指定 Protobuf 编译器路径（避免版本冲突）   |
| `-pl '!hadoop-tools/hadoop-pipes'` | 跳过 hadoop-pipes 模块（非核心，适配复杂） |
| `-T 4C`                            | 多线程编译（4 核心并行，提升效率）         |

#### 5.2.3 编译成功标志

终端输出以下内容，说明全量编译成功：

plaintext

```plaintext
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  XX:XX min (Wall Clock)
[INFO] Finished at: XXXX-XX-XXTXX:XX:XX+08:00
```

### 5.3 手动打包（若自动打包未生成安装包）

若 `hadoop-dist/target/` 无压缩包，手动触发 `hadoop-dist` 模块打包：

```bash
# 进入 hadoop-dist 模块
cd /home/hadoop-3.2.1-src/hadoop-dist

# 手动执行打包插件
mvn assembly:single -DskipTests -Dmaven.javadoc.skip=true \
  -Dprotoc.path=/usr/local/protobuf-2.5.0/bin/protoc
```

## 六、编译产物查找与验证

### 6.1 核心产物路径

编译成功后，核心部署文件位于：

```bash
# 目标目录
cd /home/hadoop-3.2.1-src/hadoop-dist/target/

# 关键产物
ls -l
```

#### 6.1.1 产物说明

| 产物名称                  | 类型     | 用途                                                |
| --------------------------- | ---------- | ----------------------------------------------------- |
| `hadoop-3.2.1.tar.gz` | 压缩包   | 完整可部署安装包（优先使用）                        |
| `hadoop-3.2.1/`       | 解压目录 | 与压缩包内容一致，可直接复制部署                    |
| `lib/native/`         | 目录     | ARM64 架构 native 库（如 libhadoop.so、libhdfs.so） |

### 6.2 产物完整性验证

确保 `hadoop-3.2.1/` 目录包含以下核心子目录（缺失则编译不完整）：

```bash
ls -l /home/hadoop-3.2.1-src/hadoop-dist/target/hadoop-3.2.1/
```

* `bin/`：Hadoop 核心命令（hdfs、yarn、hadoop 等）
* `sbin/`：集群启动 / 停止脚本（start-dfs.sh、start-yarn.sh 等）
* `etc/hadoop/`：默认配置文件目录
* `lib/native/`：ARM64 原生库（验证是否存在 `libhadoop.so.1.0.0`）
* `share/hadoop/`：核心 jar 包（hdfs、yarn、mapreduce 等）

## 七、部署与验证

### 7.1 部署步骤

#### 7.1.1 复制部署文件

```bash
# 方案 1：使用压缩包部署
sudo tar -zxf /home/hadoop-3.2.1-src/hadoop-dist/target/hadoop-3.2.1.tar.gz -C /opt/

# 方案 2：直接使用解压目录部署（无压缩包时）
sudo cp -r /home/hadoop-3.2.1-src/hadoop-dist/target/hadoop-3.2.1 /opt/

# 重命名（简化路径）
sudo mv /opt/hadoop-3.2.1 /opt/hadoop
```

#### 7.1.2 配置环境变量

```bash
# 编辑环境变量文件
sudo vi /etc/profile

# 追加以下内容
export HADOOP_HOME=/opt/hadoop
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"

# 生效环境变量
source /etc/profile
```

#### 7.1.3 核心配置文件修改（单节点模式）

编辑 `$HADOOP_HOME/etc/hadoop/` 下的 3 个核心配置文件：

1. `core-site.xml`：

xml

```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/opt/hadoop/tmp</value>  <!-- 自定义临时目录，需提前创建 -->
  </property>
</configuration>
```

2. `hdfs-site.xml`：

xml

```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>  <!-- 单节点模式，副本数设为 1 -->
  </property>
  <property>
    <name>dfs.permissions.enabled</name>
    <value>false</value>  <!-- 关闭权限检查（测试环境） -->
  </property>
</configuration>
```

3. `yarn-site.xml`：

xml

```xml
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>localhost:8032</value>
  </property>
</configuration>
```

#### 7.1.4 格式化 HDFS

bash

运行

```bash
# 创建临时目录
sudo mkdir -p /opt/hadoop/tmp
sudo chmod 777 /opt/hadoop/tmp

# 格式化 HDFS（仅首次执行）
hdfs namenode -format
```

格式化成功标志：终端输出 `successfully formatted`。

#### 7.1.5 启动集群

bash

运行

```bash
# 启动 HDFS
start-dfs.sh

# 启动 YARN
start-yarn.sh
```

### 7.2 部署验证

#### 7.2.1 进程验证

bash

运行

```bash
jps
```

应输出以下进程（单节点模式）：

* `NameNode`
* `DataNode`
* `ResourceManager`
* `NodeManager`
* `SecondaryNameNode`

#### 7.2.2 命令验证

bash

运行

```bash
# 验证 Hadoop 版本（输出版本 3.2.1 且无 native 库警告）
hadoop version

# 验证 HDFS 文件系统
hdfs dfs -ls /

# 运行 MapReduce 示例（WordCount）
# 1. 创建测试文件
echo "Hello Hadoop ARM64" > test.txt
# 2. 上传到 HDFS
hdfs dfs -put test.txt /
# 3. 运行示例
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.1.jar wordcount /test.txt /output
# 4. 查看结果
hdfs dfs -cat /output/part-r-00000
```

示例运行成功，输出 `Hello 1`、`Hadoop 1`、`ARM64 1` 即验证通过。

## 八、常见问题排查

### 8.1 头文件缺失报错（如 `rpc/xdr.h: No such file or directory`）

* 原因：RPC 头文件路径未适配 ARM64 系统。
* 解决方案：重新执行 ​**4.2 预处理步骤**​，确保 `tirpc` 头文件引用正确。

### 8.2 链接错误（如 `undefined reference to xdrmem_create`）

* 原因：`libtirpc` 库未链接或链接路径错误。
* 解决方案：检查 `CMakeLists.txt` 中 `target_link_libraries` 是否添加 `tirpc`，并确保 `link_directories(/usr/lib64)` 配置正确。

### 8.3 Protobuf 相关报错（如 `Atomic64 does not name a type`）

* 原因：Protobuf 2.5.0 未适配 ARM64 原子操作。
* 解决方案：重新执行 ​**3.2.4 步骤**​，确保 `atomicops_internals_arm64_gcc.h` 文件完整。

### 8.4 打包失败（无 `hadoop-3.2.1.tar.gz`）

* 原因：未执行 `package` 阶段或跳过核心打包模块。
* 解决方案：执行 ​**5.2.3 手动打包步骤**​，或重新运行全量打包命令（不带 `-pl` 限制核心模块）。

### 8.5 Native 库加载警告（`Unable to load native-hadoop library`）

* 原因：环境变量 `HADOOP_OPTS` 未配置 native 库路径。
* 解决方案：重新配置 ​**7.1.2 环境变量**​，确保 `HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"`。
