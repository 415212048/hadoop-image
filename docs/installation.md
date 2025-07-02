# Hadoop部署指南

## ‌一、环境准备
### 系统配置
> -  服务器：鲲鹏服务器
> -  操作系统：Huawei Cloud EulerOS 2.0 64bit 
> - CPU: 2vCPUs 或更高
> - RAM: 4GB 或更大
> - Disk: 至少 40GrB

## ‌二、安装部署部署

### 前置条件
JDK已经安装

### 下载并解压Hadoop
```bash
wget https://downloads.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6-aarch64.tar.gz  
tar -zxvf hadoop-3.3.6-aarch64.tar.gz
mv hadoop-3.3.6 /opt/hadoop
```


### 配置环境变量
```bash
echo 'export HADOOP_HOME=/opt/hadoop' >> ~/.bashrc
echo 'export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin' >> ~/.bashrc
source ~/.bashrc
```


### 配置core-site.xml
```babsh
cat << EOF > $HADOOP_HOME/etc/hadoop/core-site.xml
<configuration>
<property>
<name>fs.defaultFS</name>
<value>hdfs://localhost:9000</value>
</property>
</configuration>
EOF
```

### 配置hdfs-site.xml
```bash
cat << EOF > $HADOOP_HOME/etc/hadoop/hdfs-site.xml
<configuration>
<property>
<name>dfs.replication</name>
<value>1</value>
</property>
</configuration>
EOF
```

# 格式化HDFS
```bash
hdfs namenode -format
```

# 启动Hadoop服务
```bash
start-dfs.sh
start-yarn.sh
```

