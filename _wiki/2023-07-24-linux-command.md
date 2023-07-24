---
layout: wiki
title: linux如何对四大指标进行监控
categories: [Linux]
description: Linux系统性能的四个指标：CPU、内存、磁盘、网络
keywords: linux
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

Linux系统性能的四个指标：CPU、内存、磁盘、网络


在Linux系统中,可以通过以下常用命令来查看CPU、内存、磁盘和网络的使用情况:

CPU性能指标:

- top - 实时查看CPU使用率，内存使用率及占用进程
- mpstat - 报告CPU使用情况统计信息
- sar -u - 报告CPU使用情况历史数据

内存性能指标:

- top - 实时查看CPU使用率，内存使用率及占用进程

- free -m - 查看内存使用情况
- vmstat - 查看内存使用和交换区使用状况

磁盘性能指标:

- iostat -xz - 每秒报告一次磁盘I/O统计信息
- iotop - 默认按IO次数排序显示进程IO使用情况
- df -h - 查看磁盘使用情况

网络性能指标:

- ifconfig - 用于显示网络接口的配置信息，包括IP地址、MAC地址、网络流量统计等
- netstat -an|grep -i tcp - 查看当前TCP网络连接情况
- iftop - 实时监控服务器的网络带宽使用情况（需要安装）

