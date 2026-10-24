---
title: BJSON
tags:
  - MongoDB
  - RDBMS
---
## **BSON là gì?** 

MongoDB lưu trữ dữ liệu dưới dạng tài liệu (document), nhưng thay vì dùng chuẩn JSON thông thường, nó mã hóa thành dạng nhị phân gọi là **BSON**.
    
## **Tại sao cần BSON?** 

JSON thông thường chỉ có một kiểu số chung chung (`number`). BSON mở rộng thêm nhiều kiểu dữ liệu chính xác hơn như:
    
- **Ngày tháng & thời gian:** `datetime`, `date`.
        
- **Số học chi tiết:** `int` (số nguyên 32-bit), `long` (số nguyên 64-bit), `floating point` (số thực dấu phẩy động), `decimal128` (số thập phân độ chính xác cao cho tài chính).
        
- **Chuỗi nhị phân:** `byte array`.
        
## **Lợi ích:** 

Tránh nhầm lẫn kiểu dữ liệu, tối ưu hóa tốc độ xử lý cho máy tính và hỗ trợ chính xác các phép toán tài chính/kỹ thuật.
    