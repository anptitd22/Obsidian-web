- Kiến trúc Kubernetes: Brain & Body
- Chia làm 2 thực thể tách biệt là Control Plane (Điều khiển tập trung) và Worker Nodes (Thực thi phân tán)

![[Pasted image 20261125014538.png]]

1. Control Plane (Bộ não điều khiển)

- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager
- cloud-controller-manager

1. Worker Node

- kubelet
- kube-proxy
- Container Runtime

[[Component]]