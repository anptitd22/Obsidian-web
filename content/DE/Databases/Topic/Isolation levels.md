---
title: Isolation levels
tags:
  - MySQL
  - PostgreSQL
  - Topic
---

## Dirty Read

T1 write bảng vào bảng A chưa commit, T2 read bảng A sẽ thấy được cả dữ liệu của T1 nhưng nếu T1 rollback thì dữ liệu trên bảng sẽ mất nhưng T2 vẫn read dữ liệu đó

## Non-repeatable Read

T1 read bảng A lần 1, T2 write vào bảng A, T1 read bảng A lại lần nữa (vẫn trong transaction lần 1) thấy dữ liệu bị thay đổi

## Phantom

T1 truy vấn với điều kiện vào bảng A lần 1, T2 write dữ liệu nằm trong điều kiện đó vào bảng A, T1 truy vấn lại lần 2 (vẫn trong transaction lần 1) thì thấy dữ liệu thay đổi

## Phương pháp

Có 4 isolation level chính và tùy vào các database thì nó có chức năng khác nhau:

- Uncommitted Read

- Committed Read

- Repeatable Read

- Serializable

Ngoài ra còn có:

- Snapshot