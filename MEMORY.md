# MEMORY.md - Long-term Operating Memory

本文件只记录长期稳定的背景、判断原则、安全红线和高优先级 SOP。
具体 IP、登录命令、命令模板、rayctl 最新用法、工具路径放到 `TOOLS.md`。

## Memory 更新原则

- `MEMORY.md` 只保存长期稳定的事实、判断原则、操作入口和安全规则。
- 单次查询结果、节点当前状态、机器数量、未关联 vcluster 的节点列表、故障状态等动态信息，不要直接写入 `MEMORY.md`。
- 动态查询结果应写入 `memory/YYYY-MM-DD.md`，并标注查询时间、查询命令和“该结果需要下次重新验证”。
- 如果某条查询结果要进入 `MEMORY.md`，必须先抽象成长期规则，而不是保存原始结果。

---

## 0. 最高优先级：先判断对象类型，再选择入口

处理任何请求前，必须先判断目标对象属于哪一类。

| 请求类型                                                                                                                             | 正确入口                                            | 说明                        |
| -------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | ------------------------- |
| Kubernetes / vcluster / host cluster / rayctl / kubectl / vcjob / Pod / PVC / PV / AFS / Service / Endpoint / Webhook / PodGroup | D 集群开发机                                         | 这是 Kubernetes 逻辑资源操作      |
| 查询某个物理节点 IP / hostname 属于哪个 vcluster                                                                                             | D 集群开发机 + rayctl                               | 这是 host cluster 视角的节点归属查询 |
| 查看某台 D 集群机器上的目录、文件、日志、进程、磁盘、内存、NPU、网卡、路由、DNS、本地脚本                                                                                | 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP         | 这是物理机器 / 操作系统级查询          |
| 在某台 D 集群机器上执行脚本、改配置、删除文件、重启服务                                                                                                    | 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP，并且必须先确认 | 这是机器写操作或高风险操作             |

关键判断：

* “看某个 IP / hostname 上的目录、文件、日志、进程、磁盘、内存、NPU、网卡、路由、DNS”属于 **D 集群机器类操作**。
* “看 Pod、vcjob、PVC、PV、AFS、vcluster、PodGroup、Service、Endpoint、Webhook”属于 **Kubernetes / 集群类操作**。
* “看某个节点属于哪个 vc / vcluster”属于 **节点归属查询**，走开发机上的 rayctl；具体命令参考 `TOOLS.md`。
* 不要因为用户说“D 集群节点”就默认走开发机。
* 不要把“物理机器文件系统查询”误判为“Kubernetes Node 查询”。

---

## 1. 平台背景

当前平台是基于 Kubernetes 的 AI 训练 / 推理平台。

底层包含宿主 Kubernetes 集群，上层部署多套 vcluster。
vcluster 与 host cluster 之间的 Pod、Node 等资源通过 syncer 同步。

常见对象关系：

* Host Cluster：真实物理节点、底层资源、宿主集群 kubeconfig
* vCluster：租户视角 Kubernetes 集群
* rayctl：平台优先使用的任务 / AFS / PVC / Node 查询与操作工具
* kubectl：rayctl 信息不足或能力不满足时的降级工具
* Volcano / vcjob：训练任务常见调度与任务 CRD
* D 集群机器：真实物理机 / 节点，通过跳板机上的 sensetime 用户使用 Ansible 操作

## 1.1 集群与机器类型基本事实
- 每个 vcluster 只对应一种主要机器类型，不要假设一个 vcluster 内混合多种机器类型。
- 判断任务模板、资源规格、机器类型时，优先根据 vcluster 名称推断，再用节点 label / allocatable 做验证。
- 不要仅凭 `ring-controller.atlas: ascend-910b` 判断机器类型，该标签可能是默认标签。
- 机器类型最终应结合 `resource.compute.sensecore.cn/machine-type`、节点 allocatable、rayctl 模板映射共同判断；具体模板映射参考 `TOOLS.md`。

### vcluster 到芯片/类型映射
| vcluster | 芯片 / 类型 |
|---|---|
| `a2-*` | 华为 / NPU 910B |
| `a3-*` | 华为 / NPU 910C |
| `c550-jiaofu` | 沐曦 / MUXI 风冷 |
| `c550-ai4s` | 沐曦 / MUXI 风冷 |
| `c550-h3c` | 沐曦 / MUXI 液冷 |
| `c550-mohe` | 沐曦 / MUXI 超节点 |

### D 集群 muxi 节点 IP 段对照（通过 host cluster 视角 rayctl 查询）
> **重要**：查询 muxi 节点时，不能仅靠 vcluster 关联过滤，因为部分节点可能未关联到任何 vcluster。必须同时按 IP 段过滤排查。

| IP 段 | 对应 vcluster | 机器类型 | 备注 |
|---|---|---|---|
| `10.12.138.*` | `vc-c550-jiaofu-test` | 沐曦 风冷 | 主段 |
| `10.12.139.*` | `vc-c550-jiaofu-test` | 沐曦 风冷 | 主段 |
| `10.12.144.*` | `vc-c550-h3c-test` / `vc-c550-mohe-jiaofutest` | 沐曦 液冷 / 超节点 | 主段 |
| `10.12.145.*` | `vc-c550-h3c-test` | 沐曦 液冷 | 主段 |
| `10.12.146.*` | `vc-c550-h3c-test` | 沐曦 液冷 | 主段 |

---

### D 集群华为 910B 节点 IP 段对照（通过 host cluster 视角 rayctl 查询）
> **重要**：查询 910B 节点时，不能仅靠 vcluster 关联过滤，因为部分节点可能未关联到任何 vcluster。必须同时按 IP 段过滤排查。

| IP 段 | 对应 vcluster | 机器类型 | 备注 |
|---|---|---|---|
| `10.140.213.*` | 多个 a2-* vcluster | 华为 910B | 主段 |
| `10.140.214.*` | 多个 a2-* vcluster | 华为 910B | 主段 |


---

### D 集群华为 910C 节点 IP 段对照（通过 host cluster 视角 rayctl 查询）
> **重要**：查询 910C 节点时，不能仅靠 vcluster 关联过滤，因为部分节点可能未关联到任何 vcluster。必须同时按 IP 段过滤排查。

| IP 段 | 对应 vcluster | 机器类型 | 备注 |
|---|---|---|---|
| `10.140.215.*` | 多个 a3-* vcluster | 华为 910C | 主段 |
| `10.140.216.*` | 多个 a3-* vcluster | 华为 910C | 主段 |
| `10.140.217.*` | 多个 a3-* vcluster | 华为 910C | 主段 |

---

## 2. 操作入口原则

### 2.1 Kubernetes / 集群类操作入口

任何 Kubernetes 逻辑资源相关操作，统一走：

```text
本地 → D 集群开发机 → rayctl 优先 → kubectl 兜底
```

适用对象：

* vcluster
* host cluster
* rayctl
* kubectl
* vcjob
* Pod
* PVC / PV / AFS
* Service / Endpoint
* Webhook
* PodGroup
* Kubernetes Node 资源
* 查询某个 host cluster 节点属于哪个 vcluster

原则：

1. 先到 D 集群开发机操作。
2. 优先使用 `rayctl`。
3. 生成任何 Kubernetes / vcluster / Node / Job / PVC / AFS / ECS 操作命令前，先参考 `TOOLS.md` 的 “rayctl 使用索引”和对应 rayctl 小节。
4. 如果 `rayctl` 信息不足、能力不满足或排障需要，再使用 `kubectl`。
5. 读操作可以直接执行。
6. 写操作必须先输出计划并等待用户明确确认。

特别注意：

* “查询某个节点属于哪个 vcluster”走开发机，用 rayctl；具体命令参考 `TOOLS.md`。
* “cordon / uncordon Kubernetes Node”属于 Kubernetes Node 调度控制，默认走开发机上的 rayctl 节点调度控制；不要优先生成 `kubectl cordon` / `kubectl uncordon`，除非 rayctl 不可用或用户明确要求 kubectl。
* “查看某台机器上的目录、文件、日志、进程、磁盘、内存、NPU、网卡、路由、DNS”不是 Kubernetes 操作，不能走开发机。
* 不要把物理机器文件系统查询误判成 Kubernetes Node 查询。
* 开发机不是 D 集群物理机器操作入口。

---

### 2.2 D 集群机器类操作入口

任何对 D 集群物理机器 / 节点 / 单机操作系统环境的查询或操作，统一走：

```text
本地 → 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP
```

适用对象：

* 某台 D 集群物理 IP
* 某台 D 集群 hostname
* 机器上的文件、目录、日志
* `~sensetime` 家目录
* 本地进程
* 磁盘、内存、CPU
* NPU / `mx-smi`
* 网卡、路由、DNS
* 本地脚本，例如 `mccl.sh`

原则：

1. 不要从本地直接 SSH 到 D 集群目标节点。
2. 不要从开发机跳到 D 集群目标节点。
3. 不要在本地 OpenClaw exec 环境直接执行 ansible。
4. 不要在开发机执行 D 集群机器类 ansible。
5. 单机操作必须在跳板机 `10.140.220.33` 的 `sensetime` 用户下执行 Ansible 单 IP。
6. 默认不要依赖 inventory 文件做单机操作。
7. 对机器执行写操作、重启、改配置、跑影响业务的脚本前，必须先向用户确认。

标准单机操作方式：

```bash
ansible all -i '<目标IP>,' -m shell -a '<命令>'
```

注意：这条 ansible 命令只能在跳板机 `sensetime` 用户下执行，不能在本地或开发机执行。

---

## 3. Kubernetes / vcluster / rayctl 资源生命周期管理

### 3.1 vcjob / Volcano 任务查询

VC 集群中的训练任务统一优先用 rayctl 查询；需要 kubectl 兜底查 `vcjob` / `PodGroup` / Pod 时，必须先切到对应 vcluster kubeconfig。

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
kubectl get vcjob -A
```

红线：

* `vcjob` / `PodGroup` 是 vcluster 内资源，禁止用 host cluster kubeconfig `/root/kubeconfig` 查询或兜底。
* rayctl 输出的 `VC` 显示名不一定等于 `/root/D/` 下的 kubeconfig 文件名。不要把 `vc-c550-jiaofu-test` 直接拼成 `/root/D/vc-c550-jiaofu-test`。
* 需要先在开发机上 `ls -1 /root/D` 定位实际 kubeconfig 文件名，例如 `vc-c550-jiaofu-test` 对应的文件可能是 `/root/D/c550-jiaofu`。

禁止用以下命令判断 Volcano 任务：

```bash
kubectl get jobs -A
kubectl get vcjobs -A
```

说明：

* `jobs` 是 Kubernetes 原生 Job，不等于 Volcano Job。
* 正确资源名是 `vcjob`。

---

### 3.2 任务查询顺序

查询 D 集群 / a2 / a3 / c550 等任务时：

1. 到开发机。
2. 先用 host cluster kubeconfig / rayctl 做任务入口查询。
3. 优先使用 `rayctl job get <job-name-or-pod-name-or-uid>` 查询任务；具体命令参考 `TOOLS.md` 的 “2.3 查询任务”。
4. `rayctl job get` 已合并旧 `job check` 能力；不要再使用旧的 `rayctl job check`、`rayctl job get job`、`rayctl job get pg`。
5. 如果 rayctl 输出了 `VC` 名称，需要查 `/root/D` 目录确认实际 vcluster kubeconfig 文件名；`VC` 显示名不保证等于 kubeconfig 文件名。
6. 如果需要查看 PodGroup 细节或事件，再进入对应 vcluster kubeconfig，用 `kubectl get podgroup -A | grep <vcjob-name>` 定位 PodGroup，并用 `kubectl describe podgroup <pg-name> -n <namespace>` 查询。
7. 如果 rayctl 信息不足，再进入对应 vcluster kubeconfig，用 kubectl 查询 Pod、vcjob、PodGroup、Event 和日志。
8. 禁止在 vcluster kubeconfig 不存在或查不到时回退到 host cluster kubeconfig 查询 `vcjob` / `PodGroup`；这类资源必须在对应 vcluster 中查。
9. rayctl 结果足够详细时，以 rayctl 为准。
10. 对 `NotEnoughResources` 的精确判断应并入任务查询流程：先用 rayctl / PodGroup 定性，再用 vcluster kubeconfig 下的 kubectl 逐节点匹配目标 Pod requests 与 node allocatable / 已调度 Pod requests。
11. 不要只凭 vcluster 总资源量或概览表判断资源是否足够；调度要求同一台节点同时满足目标 Pod 的全部资源和调度约束。

---

### 3.3 任务创建原则

提交任务必须使用 vcluster kubeconfig。
禁止使用 host cluster kubeconfig 提交 vcluster 任务。

任务创建优先使用 rayctl；具体模板、参数和命令参考 `TOOLS.md` 的 “3. rayctl 任务模板”。

创建任务属于写操作，必须先向用户确认。

确认前需要说明：

* 目标 vcluster
* namespace
* job name
* template
* image
* command
* CPU / memory / accelerator / nodes
* 是否挂载 PVC
* 可能影响

模板选择和参数细节不要写死在 `MEMORY.md`，以 `TOOLS.md` 中 rayctl 任务模板为准。

---

### 3.4 AFS / PVC / PV 查询

查询 AFS / PVC / PV 时：

1. 先使用 rayctl 查询；具体命令参考 `TOOLS.md` 的 “2.6 查询 AFS / PVC / PV”。
2. AFS 名称由用户提供，不要擅自拼接或修改前缀。
3. 如果 rayctl 返回 Host PV 名称，再直接查询该 PV。
4. 已经有完整资源名时，不要再用 grep 模糊匹配。

禁止行为：

* 不要擅自给 AFS 名称加 `pvc-` 前缀。
* 不要擅自给资源名追加时间戳、UUID 或其他后缀。
* 不要在已有完整 PV / PVC 名时再用 grep 猜测匹配。

---

### 3.5 PVC 创建原则

在 vcluster 中创建 PVC 时：

* 禁止直接用 `kubectl create pvc`
* 必须使用 rayctl 创建 PVC；具体命令参考 `TOOLS.md` 的 “2.7 创建 PVC”。
* 必须使用 vcluster kubeconfig
* 不能使用 host cluster kubeconfig 创建 vcluster PVC

PVC 命名必须严格为：

```text
pvc-<afs-name>
```

禁止私自追加任何后缀、时间戳、随机字符串或 UUID。

创建 PVC 属于写操作，必须先向用户确认。

创建 PVC 后必须检查；具体检查命令参考 `TOOLS.md` 的 “2.7 创建 PVC”。

如果 PVC 是 Pending：

1. 立即中断流程。
2. 不要继续创建任务。
3. 返回 Pending 原因给用户。
4. 等用户确认后再继续处理。

PVC Pending 时禁止创建挂载该 PVC 的任务。

---

### 3.6 节点归属查询

查询某个 D host cluster 节点属于哪个 vcluster 时：

1. 到开发机。
2. 使用 host cluster kubeconfig。
3. 使用 rayctl 从 host cluster 视角查询节点归属。
4. 具体命令模板和 rayctl 最新用法以 `TOOLS.md` 的 “2.5 rayctl 节点查询” 为准。

不要进入 vcluster 内部倒查节点归属。

原因：

* vcluster 内部视角的 Node 信息可能不包含真实物理宿主信息。
* 从 vcluster 内部倒查容易误判节点和真实宿主机的关系。
* 节点归属应以 host cluster 视角的 rayctl 结果为准。

---

## 4. 专项排障与测试原则

### 4.1 机器类型判断

不要用以下标签判断机器类型：

```text
ring-controller.atlas: ascend-910b
```

该标签可能是默认标签，不能代表真实机器类型。

判断机器类型应优先看：

```text
resource.compute.sensecore.cn/machine-type
```

---

### 4.2 Megatron-LM 日志定位

Megatron-LM 训练进度日志通常不是 global rank 0 打印。

训练进度日志，如 iteration、loss、MFU、grad norm 等，通常由：

```text
global rank == world_size - 1
```

的节点打印。

例如：

* 32 节点 × 8 卡 = 256 卡
* world_size = 256
* 打印训练进度的是 global rank 255
* global rank 255 对应最后一个 worker 节点

查看训练进度日志时，优先查看最后一个 worker 节点。

---

### 4.3 节点维修 / 故障记录查询

当用户询问某节点维修、故障、处理记录时，优先查询运维故障记录表。

查询关键字：

* 节点 IP
* hostname
* 故障时间
* 维修状态

返回时汇总：

* 故障时间
* 故障现象
* 当前状态
* 处理结论
* 是否仍需要进一步确认

---

### 4.4 MCCL 测试场景区分

MCCL 测试必须先区分场景：

1. 单机 MUXI MCCL 维修验收测试。
2. 多机 / 平台纳管 MCCL 测试。

这两个场景不能混用流程。

#### 4.4.1 单机 MUXI MCCL 维修验收测试

适用场景：维修机器、故障机器恢复后，在单台 D 集群物理机上确认 8 卡 MCCL 是否正常。

入口必须是：

```text
本地 → 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP
```

关键原则：

* 不创建 vcjob。
* 不打 Kubernetes node label。
* 不删 Kubernetes node label。
* 不为了测试先 uncordon。
* 测试前如果节点处于维修 / cordon 状态，应保持原状态。
* 只在用户明确要求“通过后 uncordon”时，测试通过后再走开发机用 `rayctl node uncordon`。
* 单机测试默认运行目标机上的 `~sensetime/mccl.sh 8`，连续 5 轮。
* 如果目标机没有 `~sensetime/mccl.sh`，按 `TOOLS.md` 中记录的标准脚本写入；写入属于写操作，必须确认。
* 输出必须落到目标机 `~sensetime/mccl-*.log`。
* 判断重点是每轮每个 benchmark 的 `# Avg bus bandwidth`、`rc=0`、`# Out of bounds values : 0 OK`，以及是否出现 error / failed / timeout / hang。

#### 4.4.2 多机 / 平台纳管 MCCL 测试

适用场景：通过平台创建 vcjob / Volcano Job，对多台机器进行纳管式 MCCL 测试。

关键原则：

* 该场景才可能涉及 Kubernetes node label。
* 测试前打标、测试后删标都属于写操作，必须先确认。
* 出现坏节点时，可以动态排除。
* 排除坏节点后必须同步调整：

  * `minAvailable`
  * worker replicas
  * `CARD_NUM`

* 多机场景按最新 `TOOLS.md` 或当前有效 `mccl-test` skill 执行。
* 测试结果输出时，保留原始日志，不自行改写。
* 不擅自计算或汇总用户要求保留原始输出的 MCCL 结果；如需摘要，应基于日志中的原始 `Avg bus bandwidth` 行。

---

## 5. Skills 使用原则

当前 D 集群日常操作不再依赖旧的 `d-cluster` skill。

D 集群日常操作以本文件和 `TOOLS.md` 中的入口规则为准：

* Kubernetes / vcluster / rayctl / kubectl 操作：开发机执行。
* D 集群机器操作：堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP。

如果 OpenClaw 日志出现类似：

```text
Skipping escaped skill path outside its configured root
reason="symlink-escape"
```

说明该 skill 没有被安全加载，不能假设该 skill 生效。

`mccl-test` skill 仅在 MCCL 测试场景下使用。
如果后续重新启用某个 skill，必须确认：

1. skill 内容是最新的。
2. skill 路径没有 symlink-escape。
3. skill 规则不与 MEMORY.md / TOOLS.md 冲突。

---

## 6. 安全红线与交互偏好

### 6.1 写操作强制阻断机制

任何写操作执行前，必须先输出：

1. 将要执行的操作。
2. 影响范围。
3. 可能风险。
4. 回滚方式或停止方式。
5. 需要用户确认的问题。

在用户明确回复类似以下内容之前，禁止执行真实修改：

```text
确认
同意
可以执行
执行
yes
```

如果用户只是问“怎么做”，只能给命令和解释，不得代为执行。

写操作包括但不限于：

* `kubectl delete`
* `kubectl patch`
* `kubectl create`
* `kubectl apply`
* `kubectl label`
* `kubectl annotate`
* `kubectl scale`
* `kubectl rollout restart`
* `rayctl job create`
* `rayctl pvc create`
* 删除 / 修改本地文件
* 修改 OpenClaw 配置
* 修改 gateway / agent / model 配置
* 修改 OpenClaw skill 文件
* 修改 `MEMORY.md`
* 修改 `TOOLS.md`
* 创建临时脚本
* 重启服务
* 关机 / 重启机器
* 删除 PVC / PV / Pod / Job / Webhook
* 执行会影响业务任务的脚本

---


### 6.1.1 批量删除 Kubernetes Pod 的确认规则

批量删除 Pod 是高风险 Kubernetes 写操作，必须严格分阶段执行：

1. 先切到正确的 vcluster kubeconfig，禁止误用 host cluster kubeconfig。
2. 先只读展示筛选结果，输出 namespace、pod name、phase、creationTimestamp 等关键字段。
3. 再只读统计数量。
4. 将待删除对象保存到远端临时文件，例如 `/tmp/failed-pods-YYYYMMDD.tsv`。
5. 用 `head` 和 `wc -l` 确认文件内容与数量。
6. 停下来向用户展示删除命令，等待明确确认。
7. 用户确认后才能执行删除。
8. 删除后必须用同一筛选条件复查剩余数量。

时间筛选要明确时区：如果用户说“北京时间某天”，需要转换为 UTC 时间窗口后再写入 `creationTimestamp` 条件。

默认优先使用串行删除，降低冲击面；只有用户明确要求或数量很大且风险可控时，才使用并发 `xargs -P` 删除。

### 6.2 OpenClaw 配置修改红线

禁止未经确认修改以下内容：

* `openclaw.json`
* `config.yaml`
* gateway 相关配置
* agents 配置
* model 默认配置
* OpenClaw runtime 配置
* OpenClaw skill 文件
* `MEMORY.md`
* `TOOLS.md`

如果用户要求更新 memory、tools 或 skill：

1. 先展示建议修改内容。
2. 等用户确认。
3. 再执行写入。

---

### 6.4 用户偏好

* 默认使用中文回复。
* 排障类问题优先给结论，再给验证命令。
* Kubernetes / vcluster / Volcano / 物理机器问题必须先判断操作入口。
* 能用只读命令确认时，优先只读确认。
* 高风险操作必须先说明影响，再等待用户确认。
* 总结类内容希望归纳成清晰层级，不要零散堆列表。
