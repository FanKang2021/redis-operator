## 修复总结
### 问题描述
Redis Operator 在更新 Deployment/StatefulSet 时，即使 spec 未变化也会触发更新，导致 generation 字段持续增长（达到 288896），影响性能和可观测性。

### 根本原因
Kubernetes API Server 在存储资源时会自动填充默认值（如 schedulerName 、 terminationGracePeriodSeconds 、 imagePullPolicy 等），导致 cmp.Equal 比较失败，无法跳过更新。

### 修改文件
1. service/k8s/deployment.go

- 添加 github.com/google/go-cmp/cmp 和 github.com/google/go-cmp/cmp/cmpopts 导入
- 在 CreateOrUpdateDeployment 中添加 spec 比较逻辑，忽略 K8s 自动填充的默认字段
2. service/k8s/statefulset.go

- 添加相同的导入和比较逻辑
- 同时比较 spec 和 annotations，确保注解变更不会被跳过
### 忽略的字段