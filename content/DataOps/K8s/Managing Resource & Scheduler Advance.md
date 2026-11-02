
Khi môi trường thực sự lớn và có nhiều node, việc quản lý tài nguyên và scheduling hiệu quả là rất quan trọng. Kubernetes cung cấp nhiều cơ chế tinh chỉnh giúp bạn kiểm soát, phân bổ và tối ưu tài nguyên một cách thông minh:

- Resource Requests & Limits: Xác định mức tối thiểu và tối đa CPU/RAM cho container để tránh quá tải hoặc lãng phí tài nguyên.
- Affinity / Anti-Affinity (Node Affinity): Xác định các quy tắc để Pods chạy trên node phù hợp hoặc tránh node không phù hợp.
- Taints & Tolerations: Dùng để ngăn pods không phù hợp chạy trên node bị gắn “taint”.
- Priority Classes và Preemption: Xác định độ ưu tiên của các pod, cho phép loại bỏ (evict) những pod có mức ưu tiên thấp khi cần tài nguyên cho các pod có ưu tiên cao hơn.