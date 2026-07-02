## 修复总结

### 问题描述
Redis Operator 在更新 Deployment/StatefulSet 时，即使 spec 未变化也会触发更新，导致 generation 字段持续增长（达到 288896），影响性能和可观测性。Operator 每30秒触发一次协调循环，每次都会导致不必要的资源更新。

### 根本原因
这是典型的 Operator 空值序列化 bug。Kubernetes API Server 在存储资源时会自动填充默认值，但 Operator 内部序列化时将这些默认值视为零值/nil，导致每次协调都在两者之间切换。

具体差异字段：
1. **ConfigMapVolumeSource.DefaultMode**: K8s 默认填充 420（八进制644），Operator 设为 nil
2. **Probe 字段**: SuccessThreshold=1, FailureThreshold=3, PeriodSeconds=10（K8s默认值），Operator 设为 0
3. **PodSpec.RestartPolicy**: K8s 默认填充 "Always"，Operator 设为 ""
4. **StatefulSetSpec.PersistentVolumeClaimRetentionPolicy**: K8s 默认填充，Operator 设为 nil

### 修复方案
使用 `cmpopts.IgnoreFields` 在多个嵌套结构层级忽略 Kubernetes 自动填充的默认字段：

#### 修改文件

**1. service/k8s/deployment.go**
- 添加 `github.com/google/go-cmp/cmp` 和 `github.com/google/go-cmp/cmp/cmpopts` 导入
- 在 CreateOrUpdateDeployment 中添加 spec 比较逻辑，忽略以下自动填充字段：
  - DeploymentSpec: Strategy, RevisionHistoryLimit, ProgressDeadlineSeconds, MinReadySeconds
  - PodSpec: SchedulerName, DNSPolicy, SecurityContext, TerminationGracePeriodSeconds, RestartPolicy
  - Container: ImagePullPolicy, TerminationMessagePath, TerminationMessagePolicy
  - ConfigMapVolumeSource: DefaultMode
  - Probe: SuccessThreshold, FailureThreshold, PeriodSeconds

**2. service/k8s/statefulset.go**
- 添加相同的导入和比较逻辑
- 同时比较 spec 和 annotations
- 额外忽略 StatefulSetSpec.PersistentVolumeClaimRetentionPolicy

### 验证修复
```bash
# 监控 Deployment generation
watch -n 5 'kubectl get deployment rfs-redisfailover-persistent -n base -o jsonpath="{.metadata.generation}"'

# 监控 StatefulSet generation
watch -n 5 'kubectl get statefulset rfr-redisfailover-persistent -n base -o jsonpath="{.metadata.generation}"'
```

修复后，Generation 应停止增长，Operator 只会在实际配置变更时触发资源更新。