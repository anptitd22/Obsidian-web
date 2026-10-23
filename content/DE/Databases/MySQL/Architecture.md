---
title: Architecture
tags:
  - MySQL
---
## Overview

![[Pasted image 20261023233614.png]]

Dữ liệu luôn được sắp xếp tổ chức theo cây B-tree giúp truy vấn nhanh nhưng đổi lại việc insert, update diễn ra rất chậm

Hỗ trợ nhiều Storage Engine như InnoDB(mặc định), MyISAM, Memory, ...

## InnoDB

![[Pasted image 20261024003623.png]]

Mỗi 1 bảng (không phân vùng) tương ứng với 1 file .ibd ( bao gồm dữ liệu và index)