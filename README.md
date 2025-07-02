<h1 align="center">Hadoop Distributed Computing Platform</h1>
<p align="center">
    <strong>English</strong> | <a href="README_ZH.md">简体中文</a>
</p>

## Table of Contents

- [Repository Introduction](#project-introduction)
- [Prerequisites](#prerequisites)
- [Image Description](#image-description)
- [Get Help](#get-help)
- [How to Contribute](#how-to-contribute)

## Project Introduction

[Hadoop](https://github.com/apache/hadoop) is an open - source distributed computing platform used for processing large - scale data storage and computing. This product is based on the Huawei Cloud EulerOS 2.0 64 - bit system of Kunpeng servers, providing an out - of - the - box Hadoop computing platform.

## Core Components and Functions

1. HDFS (Hadoop Distributed File System)
-
- **Storage Architecture**: It adopts a master - slave structure. The NameNode manages metadata, and the DataNode stores actual data blocks (default 128MB/block), supporting multi - replica redundancy (default 3 copies) to ensure fault tolerance.
- **Applicable Scenarios**: It is suitable for large - file batch processing (such as video storage and log analysis), but has low efficiency for low - latency access and small - file storage.

2. MapReduce
-
- **Computing Model**: It realizes distributed computing through two stages of Map (data sharding processing) and Reduce (result aggregation), simplifying the complexity of parallel programming.

3. YARN (Resource Scheduling System)
-
- **Function**: It dynamically allocates cluster resources (CPU, memory), supports concurrent execution of multiple tasks (such as MapReduce and Spark), and improves resource utilization.

The open - source image product [**Hadoop Distributed Computing Platform**](https://marketplace.huaweicloud.com/intl/hidden/contents/8c0929fa-a100-4792-a671-24ce36dd51d4) provided by this project has pre - installed Hadoop version 3.3.6 and its related operating environment, and provides deployment templates. Come and refer to the usage guide to easily start the "out - of - the - box" efficient experience!

> **System requirements are as follows:**
> - CPU: 2vCPUs or higher
> - RAM: 4GB or larger
> - Disk: At least 40GB

## Prerequisites
[Register a Huawei account and activate Huawei Cloud](https://support.huaweicloud.com/usermanual-account/account_id_001.html)

## Image Description

| Image Specification                                                                                                       | Feature Description | Remarks |
|------------------------------------------------------------------------------------------------------------| --- | --- |
| [hadoop - 3.3.6 - kunpeng](https://github.com/HuaweiCloudDeveloper/hadoop-image/tree/hadoop-3.3.6-kunpeng) | Installed and deployed based on Kunpeng servers + Huawei Cloud EulerOS 2.0 64 - bit |  |

## Get Help
- For more questions, you can contact us via [issue](https://github.com/HuaweiCloudDeveloper/hadoop-image/issues) or the service support of the specified product in the Huawei Cloud Marketplace.
- For other open - source images, see [open - source - image - repos](https://github.com/HuaweiCloudDeveloper/open - source - image - repos)

## How to Contribute
- Fork this repository and submit a merge request.
- Synchronously update README.md based on your open - source image information.