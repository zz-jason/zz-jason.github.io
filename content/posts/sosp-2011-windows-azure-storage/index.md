---
title: "[SOSP 2011] Windows Azure Storage: A Highly Available Cloud Storage Service with Strong Consistency"
date: 2024-05-07T00:00:00Z
categories: ["Paper Reading", "Storage", "Blob Store"]
draft: true
---

## Introduction

Windows Azure Storage（WAS）2008 年上线，在微软内部用于 social networking search, serving video, music and game content, managing medical records 等场景中。

WAS 提供 3 种 API 供用户使用：Blob（文件）、Table（结构化存储）、Queue（消息传递），这三种数据结构和抽象足够大多数应用场景了。一个 common usage pattern：incoming and outgoing data being shipped via Blobs, Queues providing the overall workflow for processing the Blobs, and intermediate service state and final results being kept in Tables or Blobs.

WAS 的一些核心功能：
- **Strong Consistency** – Many customers want strong consistency: especially enterprise customers moving their line of business applications to the cloud. They also want the ability to perform conditional reads, writes, and deletes for optimistic concurrency control [12] on the strongly consistent data. For this, WAS provides three properties that the CAP theorem [2] claims are difficult to achieve at the same time: strong consistency, high availability, and partition tolerance (see Section 8).
- **Global and Scalable Namespace/Storage** – For ease of use, WAS implements a global namespace that allows data to be stored and accessed in a consistent manner from any location in the world. Since a major goal of WAS is to enable storage of massive amounts of data, this global namespace must be able to address exabytes of data and beyond. We discuss our global namespace design in detail in Section 2.
- **Disaster Recovery** – WAS stores customer data across multiple data centers hundreds of miles apart from each other. This redundancy provides essential data recovery protection against disasters such as earthquakes, wild fires, tornados, nuclear reactor meltdown, etc.
- **Multi-tenancy and Cost of Storage** – To reduce storage cost, many customers are served from the same shared storage infrastructure. WAS combines the workloads of many different customers with varying resource needs together so that significantly less storage needs to be provisioned at any one point in time than if those services were run on their own dedicated hardware.

## Global Partitioned Namespace
