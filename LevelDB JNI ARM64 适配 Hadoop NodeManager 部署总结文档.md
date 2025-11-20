## 一、背景与目标

### 1. 核心需求

* 编译适用于 **Linux aarch64（ARM64）架构** 的 `leveldbjni-all-1.8.jar`（LevelDB 的 Java 原生接口封装包）。
* 该 JAR 包用于 Hadoop NodeManager，作为 LevelDB 状态存储的 Java 适配层，支持 NodeManager 加载原生库实现元数据存储。
* ### 2. 依赖关系说明
* ​**Snappy-1.1.5**​：LevelDB 依赖的高性能压缩库，LevelDB JNI 编译需链接其原生库（`libsnappy.so`）。
* ​**LevelDB-1.20**​：核心键值存储库（C++ 实现），LevelDB JNI 通过 JNI 封装其接口，需先编译为 ARM64 架构的 `libleveldb.so`。
* ​**LevelDB JNI**​：基于上述两个原生库，生成 Java 可调用的 `libleveldbjni.so`，最终打包为 `leveldbjni-all-1.8.jar`。

### 2. 最终目标

* 生成包含 ARM64 原生库（`libleveldbjni.so`）的标准 JAR 包（非 OSGi bundle）。
* 解决 Hadoop NodeManager 启动依赖缺失、原生库加载失败等问题，确保服务正常运行。

## 二、编译与部署（含依赖）

### 模块 1：编译依赖库（Snappy-1.1.5 + LevelDB-1.20）

#### 1.1 编译 Snappy-1.1.5（压缩库）

##### 前置条件

安装编译工具：

```bash
yum install -y gcc g++ make autoconf libtool  # CentOS/RHEL
```

##### 编译步骤

```bash
# 下载源码
wget https://github.com/google/snappy/archive/refs/tags/1.1.5.tar.gz -O snappy-1.1.5.tar.gz
# 解压
tar -zxf snappy-1.1.5.tar.gz && cd snappy-1.1.5

# 生成配置文件
autoreconf -i
# 配置（指定安装路径为 /usr/local，便于后续引用）
./configure --prefix=/usr/local --host=aarch64-linux-gnu
# 编译并安装（-j4 启用4线程加速）
make -j4 && make install
```

##### 验证安装

```bash
# 检查 ARM64 架构的 libsnappy.so 是否生成
file /usr/local/lib/libsnappy.so
# 预期输出：ELF 64-bit LSB shared object, ARM aarch64...
```

#### 1.2 编译 LevelDB-1.20（核心存储库）

##### 编译步骤

```bash
# 下载源码
wget https://github.com/google/leveldb/archive/refs/tags/1.20.tar.gz -O leveldb-1.20.tar.gz
# 解压
tar -zxf leveldb-1.20.tar.gz && cd leveldb-1.20

# LevelDB 用 cmake 构建，需先安装 cmake
yum install -y cmake  # 或 apt-get install -y cmake

# 创建构建目录
mkdir -p build && cd build
# 配置（指定 Snappy 路径，启用 Snappy 压缩）
cmake -DCMAKE_INSTALL_PREFIX=/usr/local \
      -DCMAKE_BUILD_TYPE=Release \
      -DLEVELDB_BUILD_SHARED=ON \
      -DSNAPPY_INCLUDE_DIR=/usr/local/include \
      -DSNAPPY_LIBRARY=/usr/local/lib/libsnappy.so \
      ..

# 编译并安装
make -j4 && make install
```

##### 验证安装

```bash
# 检查 ARM64 架构的 libleveldb.so 是否生成
file /usr/local/lib/libleveldb.so
# 预期输出：ELF 64-bit LSB shared object, ARM aarch64...

# 更新系统库缓存（确保后续编译能找到库）
ldconfig
```

### 模块 2：编译 `leveldbjni-all-1.8.jar`（核心编译阶段）

#### 2.1 准备 LevelDB JNI 源码

```bash
# 克隆 LevelDB JNI 源码
git clone https://github.com/fusesource/leveldbjni.git && cd leveldbjni
```

#### 2.2 配置依赖路径（关键！链接 Snappy 和 LevelDB）

编辑 `leveldbjni-linux64-aarch64/pom.xml`，添加 Snappy 和 LevelDB 的链接参数：

```xml
<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>native-maven-plugin</artifactId>
  <configuration>
    <!-- 新增：指定 Snappy 和 LevelDB 的库路径 -->
    <linkerArguments>
      <linkerArgument>-L/usr/local/lib</linkerArgument>  <!-- 包含 libsnappy.so 和 libleveldb.so -->
      <linkerArgument>-lsnappy</linkerArgument>         <!-- 链接 Snappy -->
      <linkerArgument>-lleveldb</linkerArgument>        <!-- 链接 LevelDB -->
    </linkerArguments>
  </configuration>
</plugin>
```

#### 2.3 解决编译常见问题

##### 问题 1：OSGi 相关警告与多平台库缺失

* ​**解决方案**​：参考前文，移除 `maven-bundle-plugin`，改用 `maven-assembly-plugin` 打包标准 JAR（配置见后文）。

##### 问题 2：Maven 模块未识别（`Could not find the selected project in the reactor`）

* ​**解决方案**​：父工程 `pom.xml` 中添加 `<module>leveldbjni-all</module>`，或直接指定路径编译：
  bash
  
  运行
  
  ```bash
  mvn clean package -f leveldbjni-all/pom.xml -am -DskipTests
  ```

##### 问题 3：原生库符号链接损坏（`broken symbolic link`）

* ​**解决方案**​：修复链接或直接引用物理文件（如 `libleveldbjni-99-master-SNAPSHOT.so`），确保打包时包含有效原生库。

### 模块 3：Hadoop 依赖补充与部署

#### 3.1 补充 LevelDB Java 接口与 JNI 工具包

bash

运行

```bash
# 下载 leveldb-api（Java 接口）
wget https://maven.aliyun.com/repository/public/org/iq80/leveldb/leveldb-api/0.9/leveldb-api-0.9.jar -P $HADOOP_HOME/share/hadoop/yarn/lib/

# 下载 hawtjni-runtime（JNI 工具）
wget https://maven.aliyun.com/repository/public/org/fusesource/hawtjni/hawtjni-runtime/1.8/hawtjni-runtime-1.8.jar -P $HADOOP_HOME/share/hadoop/yarn/lib/
```

#### 3.2 部署原生库（解决 `UnsatisfiedLinkError`）

```bash
# 从 leveldbjni-all-1.8.jar 提取原生库
jar xf leveldbjni-all/target/leveldbjni-all-1.8.jar META-INF/native/linux64/aarch64/libleveldbjni.so -C /tmp/

# 复制到 JVM 搜索路径（如 /usr/lib64）
cp /tmp/META-INF/native/linux64/aarch64/libleveldbjni.so /usr/lib64/
chmod 755 /usr/lib64/libleveldbjni.so

# 确保 Snappy 和 LevelDB 原生库也在系统路径中（已通过 ldconfig 生效）
```

### 模块 4：启动与验证 NodeManager

```bash
# 重启服务
yarn --daemon stop nodemanager
yarn --daemon start nodemanager

# 查看日志，确认无依赖或原生库错误
tail -f $HADOOP_HOME/logs/yarn-*-nodemanager-*.log
```

## 三、最终关键配置文件

### 1. `leveldbjni-all/pom.xml`（完整配置）

xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <parent>
    <groupId>org.fusesource.leveldbjni</groupId>
    <artifactId>leveldbjni-project</artifactId>
    <version>1.8</version>
  </parent>
  <modelVersion>4.0.0</modelVersion>

  <artifactId>leveldbjni-all</artifactId>
  <packaging>jar</packaging>
  <name>LevelDB JNI All (Linux AArch64)</name>

  <dependencies>
    <dependency>
      <groupId>org.fusesource.leveldbjni</groupId>
      <artifactId>leveldbjni</artifactId>
      <version>1.8</version>
    </dependency>
    <dependency>
      <groupId>org.fusesource.leveldbjni</groupId>
      <artifactId>leveldbjni-linux64-aarch64</artifactId>
      <version>1.8</version>
      <type>jar</type>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <artifactId>maven-source-plugin</artifactId>
        <configuration>
          <skipSource>true</skipSource>
        </configuration>
      </plugin>

      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.3.0</version>
        <executions>
          <execution>
            <id>uber-jar</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
            <configuration>
              <descriptors>
                <descriptor>${basedir}/src/main/descriptors/uber-leveldb.xml</descriptor>
              </descriptors>
              <appendAssemblyId>false</appendAssemblyId>
              <finalName>leveldbjni-all-1.8</finalName>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

### 2. `uber-leveldb.xml`（聚合描述符）

xml

```xml
<assembly xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3 http://maven.apache.org/xsd/assembly-1.1.3.xsd">
  <id>uber</id>
  <formats>
    <format>jar</format>
  </formats>
  <includeBaseDirectory>false</includeBaseDirectory>

  <dependencySets>
    <dependencySet>
      <unpack>true</unpack>
      <scope>runtime</scope>
      <includes>
        <include>org.fusesource.leveldbjni:leveldbjni</include>
        <include>org.fusesource.leveldbjni:leveldbjni-linux64-aarch64</include>
      </includes>
    </dependencySet>
  </dependencySets>

  <fileSets>
    <fileSet>
      <directory>${project.build.outputDirectory}</directory>
      <outputDirectory>/</outputDirectory>
    </fileSet>

    <fileSet>
      <directory>${project.basedir}/../leveldbjni-linux64-aarch64/target/native-build/target/lib/</directory>
      <outputDirectory>META-INF/native/linux64/aarch64/</outputDirectory>
      <includes>
        <include>libleveldbjni-99-master-SNAPSHOT.so</include>
      </includes>
      <fileMode>0755</fileMode>
    </fileSet>
  </fileSets>
</assembly>
```

## 四、最终部署验证步骤

### 1. 编译 JAR 包

```bash
cd /opt/leveldbjni
mvn clean package -f leveldbjni-all/pom.xml -am -DskipTests
```

* 生成路径：`leveldbjni-all/target/leveldbjni-all-1.8.jar`

### 2. 补充 Hadoop 依赖

```bash
# 下载并复制 leveldb-api 和 hawtjni-runtime
wget https://maven.aliyun.com/repository/public/org/iq80/leveldb/leveldb-api/0.9/leveldb-api-0.9.jar -P $HADOOP_HOME/share/hadoop/yarn/lib/
wget https://maven.aliyun.com/repository/public/org/fusesource/hawtjni/hawtjni-runtime/1.8/hawtjni-runtime-1.8.jar -P $HADOOP_HOME/share/hadoop/yarn/lib/
```

### 3. 部署原生库

```bash
# 提取原生库并部署到系统路径
jar xf leveldbjni-all/target/leveldbjni-all-1.8.jar META-INF/native/linux64/aarch64/libleveldbjni-99-master-SNAPSHOT.so -C /tmp/
cp /tmp/META-INF/native/linux64/aarch64/libleveldbjni-99-master-SNAPSHOT.so /usr/lib64/libleveldbjni.so
chmod 755 /usr/lib64/libleveldbjni.so
```

### 4. 启动并验证 NodeManager

```bash
# 重启服务
yarn --daemon stop nodemanager
yarn --daemon start nodemanager

# 查看日志，确认无错误
tail -f $HADOOP_HOME/logs/yarn-*-nodemanager-*.log
```

* 无 `NoClassDefFoundError`、`UnsatisfiedLinkError` 即为成功。

## 五、问题与解决方案

### 模块 1：编译 `leveldbjni-all-1.8.jar`（核心编译阶段）

#### 问题 1：OSGi 相关警告与原生库缺失错误

* ​**错误信息**​：`Split package org/fusesource/leveldbjni`、`Native library not found in JAR`（多平台库缺失）。
* ​**原因​**​：默认使用 `maven-bundle-plugin` 打包为 OSGi bundle，严格校验包结构和全平台原生库。
* ​**解决方案**​：移除 OSGi 配置，改用 `maven-assembly-plugin` 生成标准 JAR。
  1. 修改 `leveldbjni-all/pom.xml`：
     * 打包类型改为 `jar`，移除 `maven-bundle-plugin`。
     * 添加 `maven-assembly-plugin` 配置，聚合核心依赖和 ARM64 原生库。
  2. 创建 `uber-leveldb.xml` 聚合描述符，指定原生库路径和依赖合并规则。

#### 问题 2：Maven 模块未识别（`Could not find the selected project in the reactor`）

* ​**原因​**​：父工程 `pom.xml` 的 `<modules>` 未声明 `leveldbjni-all` 模块。
* ​**解决方案**​：
  1. 编辑根目录父工程 `pom.xml`，在 `<modules>` 中添加 `<module>leveldbjni-all</module>`。
  2. 若仍未识别，直接通过绝对路径编译：`mvn clean package -f /opt/leveldbjni/leveldbjni-all/pom.xml -DskipTests`。

#### 问题 3：原生库符号链接损坏（`broken symbolic link`）

* ​**错误信息**​：`libleveldbjni.so` 是损坏的符号链接，指向不存在的 `libleveldbjni-99-master-SNAPSHOT.so`。
* ​**解决方案**​：
  1. 找到实际物理原生库（如 `libleveldbjni-99-master-SNAPSHOT.so`）。
  2. 修复符号链接：`ln -sf libleveldbjni-99-master-SNAPSHOT.so libleveldbjni.so`，或直接在打包时引用物理文件。

### 模块 2：Hadoop 依赖补充（NodeManager 启动准备）

#### 问题 1：类缺失错误（`NoClassDefFoundError: org/iq80/leveldb/DBException`）

* ​**原因​**​：缺少 LevelDB Java 接口依赖 `leveldb-api`。
* ​**解决方案**​：
  1. 下载 `leveldb-api-0.9.jar`（纯 Java 包，无需 ARM 适配）。
  2. 复制到 Hadoop 类路径：`cp leveldb-api-0.9.jar $HADOOP_HOME/share/hadoop/yarn/lib/`。

#### 问题 2：JNI 工具包缺失（`hawtjni` 依赖缺失）

* ​**原因​**​：`leveldbjni` 依赖 `hawtjni-runtime` 简化 JNI 调用。
* ​**解决方案**​：
  1. 通过阿里云镜像下载 `hawtjni-runtime-1.8.jar`（纯 Java 包）：
     
     ```bash
     wget https://maven.aliyun.com/repository/public/org/fusesource/hawtjni/hawtjni-runtime/1.8/hawtjni-runtime-1.8.jar
     ```
  2. 复制到 `$HADOOP_HOME/share/hadoop/yarn/lib/`。

### 模块 3：NodeManager 启动核心错误（原生库加载失败）

#### 问题：`java.lang.UnsatisfiedLinkError: Could not load library. Reasons: [no leveldbjni in java.library.path]`

* ​**原因​**​：JVM 在 `java.library.path` 中找不到 ARM64 原生库 `libleveldbjni.so`。
* ​**解决方案**​：
  1. 从 `leveldbjni-all-1.8.jar` 中提取原生库：
     
     ```bash
     jar xf /path/to/leveldbjni-all-1.8.jar META-INF/native/linux64/aarch64/libleveldbjni.so -C /tmp/
     ```
  2. 复制到 JVM 默认扫描路径（如 `/usr/lib64`）：
     bash
     
     运行
     
     ```bash
     cp /tmp/META-INF/native/linux64/aarch64/libleveldbjni.so /usr/lib64/
     chmod 755 /usr/lib64/libleveldbjni.so
     ```
  3. 重启 NodeManager：`yarn --daemon stop nodemanager && yarn --daemon start nodemanager`。

## 六、备注

* `leveldb-api` 和 `hawtjni-runtime` 为纯 Java 包，无需 ARM 适配，通用所有架构。
* 原生库需确保为 ARM64 架构（通过 `file libleveldbjni.so` 验证，输出含 `ARM aarch64`）。
* 若 Hadoop 自定义了 `java.library.path`，需将原生库部署到对应路径。
