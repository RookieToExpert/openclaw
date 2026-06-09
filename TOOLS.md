# TOOLS.md - Environment Tools and Command Templates

本文件记录具体工具、入口、机器、命令模板和执行注意事项。
长期原则、安全红线、对象生命周期规则放到 `MEMORY.md`。

---

## 0. 最高优先级：入口路由规则

执行任何命令前，必须先判断用户请求属于哪一类。

| 请求类型                                                                                                              | 正确入口                                         | 示例                            |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------- | ----------------------------- |
| Kubernetes / vcluster / rayctl / kubectl / vcjob / Pod / PVC / PV / AFS / Service / Endpoint / Webhook / PodGroup | D 集群开发机                                      | 查任务、查 Pod、查 PVC、创建任务、创建 PVC   |
| 查询某个 IP / hostname 属于哪个 vcluster                                                                                  | D 集群开发机 + rayctl，参考 `2.5 rayctl 节点查询`    | “10.12.x.x 属于哪个 vc”           |
| 查看某台 D 集群机器上的文件、目录、日志、进程、磁盘、内存、NPU、网卡、路由、DNS、本地脚本                                                                 | 堡垒机 → 跳板机 → sensetime → ansible 单 IP         | “看 10.12.x.x 的 ~sensetime 目录” |
| 在某台 D 集群机器上执行脚本、改配置、删除文件、重启服务                                                                                     | 堡垒机 → 跳板机 → sensetime → ansible 单 IP，并且必须先确认 | “在 10.12.x.x 上跑 mccl.sh”      |

禁止误判：

* “看某台机器目录 / 文件 / 日志”不是 Kubernetes Node 查询。
* “看某台机器 mx-smi / 磁盘 / 内存 / 网卡 / DNS”不是 Kubernetes 查询。
* “节点归属 vcluster 查询”走开发机上的 rayctl，具体命令参考 `2.5 rayctl 节点查询`。
* 物理机器 OS 查询不能走开发机。
* Ansible 不能在本地 OpenClaw exec 环境执行。
* Ansible 不能在开发机执行。

---

## 1. 操作入口总览

### 1.1 Kubernetes / 集群类操作入口

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
* 节点归属 vcluster 查询

统一入口：

```text
本地 → D 集群开发机 → rayctl 优先 → kubectl 兜底
```

开发机：

```text
10.140.158.149:32222
```

开发机用途：

* 查询 D 集群节点归属
* 执行 rayctl
* 执行 kubectl
* 查询任务状态
* 查询 vcluster 资源
* 创建任务
* 创建 PVC
* 检查 AFS / PV / PVC
* 排查 Volcano / vcjob / Pod 问题

注意：

* 端口是 `32222`，不是 `22`。
* 开发机网络可能不稳定。
* 出现 banner exchange timeout 时最多重试 3 次。
* 重试 3 次仍失败，应停止执行并向用户说明原因。
* 不要从本地直接执行 D 集群 kubeconfig / rayctl 操作。
* 不要在开发机执行 D 集群物理机器 Ansible 单机操作。
* 不要从开发机 SSH 到 D 集群目标物理节点。

---

### 1.2 D 集群机器类操作入口

适用对象：

* 某台 D 集群物理机 IP
* 某台 D 集群 hostname
* 机器上的文件或目录
* `~sensetime` 家目录
* 本地日志
* 本地进程
* `mx-smi`
* 磁盘、内存、CPU
* 网卡、路由、DNS
* 本地脚本，例如 `mccl.sh`

统一入口：

```text
本地 → 堡垒机 → 跳板机 → sensetime 用户(sudo su sensetime) → ansible 单 IP
```

机器关系：

| 角色  | 地址                   | 用途                                    |
| --- | -------------------- | ------------------------------------- |
| 堡垒机 | 10.140.3.216:5906    | JumpServer 入口，密码是6RV&QnwPrq                  |
| 跳板机 | 10.140.220.33        | D 集群内部中转机，也可在 JumpServer 中输入 `220.33` |
| 开发机 | 10.140.158.149:32222 | 只用于 Kubernetes / rayctl / kubectl 操作  |

注意：

* 开发机和跳板机是不同机器。
* Kubernetes 操作走开发机。
* D 集群机器操作走堡垒机 → 跳板机 → sensetime。
* 单机机器操作优先 Ansible，不要直接 SSH 到目标节点。


---

### 1.3 D 集群 Ansible 执行位置红线

`ansible all -i '<目标IP>,' -m shell -a '<命令>'` 只能在以下环境执行：

```text
跳板机 10.140.220.33 的 sensetime 用户 shell
```

执行 ansible 前必须已经满足：

```text
当前 shell 已经在跳板机
当前用户已经是 sensetime
当前目标是 D 集群物理机器 IP 或 hostname
```

禁止事项：

* 禁止在本地 OpenClaw `exec` 环境直接执行 `ansible ...`。
* 禁止在开发机 `10.140.158.149:32222` 上执行 D 集群机器类 `ansible ...`。
* 禁止在没有进入跳板机 sensetime 用户前执行 `ansible all -i '<IP>,' ...`。
* 如果无法确认当前 shell 是跳板机 sensetime 用户，必须停止并说明原因。

---

## 2. Kubernetes / rayctl / kubectl 操作

### 复杂远程命令规则

- 通过 SSH 到开发机执行复杂命令时，禁止生成超长单行命令。
- 如果命令包含 `awk`、`jq`、多层引号、管道、正则、变量 `$NF` 等，必须使用 heredoc：

```bash
cat <<'REMOTE' | ssh -p 32222 root@10.140.158.149 'bash -s'
<远程脚本内容>
REMOTE
```

### rayctl 使用索引

rayctl 是 D 集群 Kubernetes / vcluster 日常查询的优先工具。本节集中维护 rayctl 的具体使用方法，`MEMORY.md` 只保留入口原则和长期规则。

常用位置：

* 全局 kubeconfig 参数：见 `2.0 rayctl 全局用法`
* rayctl 集群环境切换：见 `2.0 rayctl 全局用法`
* ECS / AIS 查询和登录：见 `2.8 rayctl ECS / AIS`
* 节点归属、IP 段过滤、节点详情：见 `2.5 rayctl 节点查询`
* 节点 cordon / uncordon 调度控制：见 `2.5 rayctl 节点查询`
* Volcano Job 查询和诊断：见 `2.3 查询任务`；PodGroup 细节用 vcluster kubeconfig 下的 kubectl 查询
* AFS / PVC / PV 查询：见 `2.6 查询 AFS / PVC / PV`
* PVC 创建：见 `2.7 创建 PVC`
* 任务创建模板：见 `3. rayctl 任务模板`

### 2.0 rayctl 全局用法

rayctl 当前能力总览：

```text
rayctl cluster set <d|dcloud>
rayctl ecs check <ais-name-or-ecs-name-or-uid> [...]
rayctl ecs login <ais-name-or-ecs-name-or-uid>
rayctl node get [profile-or-selector]
rayctl node check <node-name-or-ip> [...]
rayctl node describe <node-name-or-ip> [...]
rayctl node cordon <node-name>
rayctl node uncordon <node-name>
rayctl job get <job-name-or-pod-name-or-uid> [...]
rayctl job create <template>
rayctl afs check <afs-name-or-uid> [...]
rayctl pvc check <pvc-name> [...]
rayctl pvc create
```

全局参数：

```bash
rayctl --kubeconfig /path/to/kubeconfig <command>
rayctl -k /path/to/kubeconfig <command>
```

优先通过环境变量或显式 `--kubeconfig` 控制目标集群：

```bash
export KUBECONFIG=/root/kubeconfig
export KUBECONFIG=/root/D/<vc-name>
rayctl -k /root/D/<vc-name> job get <job-name-or-pod-name-or-uid>
```

切换 rayctl 平台环境：

```bash
rayctl cluster set d
rayctl cluster set dcloud
```

注意：

* Kubernetes / vcluster 操作先查本 rayctl 小节，只有 rayctl 信息不足或能力不满足时才用 kubectl 兜底。
* 创建任务、创建 PVC、cordon / uncordon 等写操作，执行前必须先向用户确认。
* `rayctl job create --what-if` 只生成 YAML，不提交到集群，适合先预览。

### 2.1 登录开发机

```bash
ssh -p 32222 root@10.140.158.149
```

异常处理：

* 如果 banner exchange timeout，最多重试 3 次。
* 如果 3 次仍失败，停止并向用户反馈。
* 不要无限重试。

---

### 2.2 kubeconfig

D host cluster：

```bash
export KUBECONFIG=/root/kubeconfig
```

Dcluster host cluster：

```bash
export KUBECONFIG=/root/kubeconfigc
```

vcluster kubeconfig 目录：

```bash
/root/D/
```

示例：

```bash
export KUBECONFIG=/root/D/a3-llmit
export KUBECONFIG=/root/D/c550-h3c
export KUBECONFIG=/root/D/ai4chem
```

---

### 2.3 查询任务

优先 rayctl。新版 `rayctl job get` 已合并旧 `job check` 能力，并删除了旧的 `job get job` / `job get pg` 子入口：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl job get <job-name-or-pod-name-or-uid>
rayctl -k /root/D/<vc-name> job get <job-name-or-pod-name-or-uid>
```

注意：

* 不要再使用旧命令 `rayctl job check <job-name>`。
* 不要再使用旧命令 `rayctl job get job <job-name>`。
* 不要再使用旧命令 `rayctl job get pg <podgroup-name-or-uid>`。
* `rayctl job get` 支持传 job 名、pod 名或 UID。
* `rayctl job get` 会按任务状态分支输出：
  * running：展示原来的任务详情、Pod、节点、日志等信息。
  * pending / startup：在第一张表中合并展示“结论”和“下一步”，不显示 `LATEST LOGS`。

如果需要进入 vcluster 再查 Pod / vcjob：

```bash
export KUBECONFIG=/root/D/<vc-name>
kubectl get vcjob -A | grep <job-name>
kubectl get pod -A | grep <job-name>
```

如果需要查看 PodGroup 细节或事件，先在 vcluster 中定位 PodGroup，再 describe：

```bash
export KUBECONFIG=/root/D/<vc-name>
kubectl get podgroup -A | grep <vcjob-name>
kubectl describe podgroup <podgroup-name> -n <namespace>
```

---

### 2.4 查询 vcjob

```bash
kubectl get vcjob -A
```

不要使用：

```bash
kubectl get jobs -A
kubectl get vcjobs -A
```

---

### 2.5 rayctl 节点查询

节点查询支持 profile、label selector、InternalIP 片段：

```bash
rayctl node get
rayctl node get ecp
rayctl node get ecs
rayctl node get 'node-role.compute.sensecore.cn/prod=ecs'
rayctl node get -A 10.12.138
rayctl node get -A -l 10.12.138
```

常用参数：

```text
-A, --all      显示全部节点；不加时默认限制展示前 100 条
-l, --long     显示更多列，例如 tenant
```

#### 查询节点归属 vcluster

```bash
export KUBECONFIG=/root/kubeconfig
rayctl node get -A <ip-fragment>
```

结果最后一列是 vcluster 名称。

示例：

```bash
rayctl node get -A 10.12.138.26
rayctl node get -A 10.12.138
rayctl node get -A '10.12.138|10.140.214'
```

兼容旧版 rayctl 或按 hostname 查询时：

```bash
rayctl node get -A | grep 10.12.138.26
rayctl node get -A | grep host-10-12-138-26
```

不要进入 vcluster 内部倒查节点归属。

#### 查询节点详情 / 检查节点资源

查询节点详情 / 检查节点资源与 vcluster Pod 时，可以直接传节点名或完整 IP：

```bash
rayctl node check host-10-140-214-222
rayctl node check 10.140.214.222
rayctl node describe 10.140.214.222
```

#### 节点调度控制：cordon / uncordon

Kubernetes Node cordon / uncordon 默认使用 rayctl，不要优先生成 `kubectl cordon` / `kubectl uncordon`。

执行前先做只读确认：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl node check <node-name-or-ip>
```

写操作必须先向用户确认。确认后执行：

```bash
rayctl node cordon <node-name>
rayctl node uncordon <node-name>
```

如果用户只给 IP，先用 rayctl 查询或检查解析节点：

```bash
rayctl node get -A <ip-fragment>
rayctl node check <full-ip>
```

只有在 rayctl 不可用、能力不满足或用户明确要求 kubectl 时，才使用 kubectl 兜底：

```bash
kubectl cordon <node-name>
kubectl uncordon <node-name>
```

---

### 2.6 查询 AFS / PVC / PV

查询 AFS：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl afs check <afs-name>
rayctl afs check -l <afs-name-or-uid>
```

查询 PVC：

```bash
export KUBECONFIG=/root/D/<vc-name>
rayctl pvc check <pvc-name>
rayctl pvc check -l <pvc-name>
kubectl get pvc <pvc-name> -o yaml
```

`-l` / `--long` 会显示额外详情，例如 tenant。

查询 Host PV：

```bash
export KUBECONFIG=/root/kubeconfig
kubectl get pv <host-pv-name>
kubectl get pv <host-pv-name> -o yaml
```

---

### 2.7 创建 PVC

创建 PVC 必须使用 vcluster kubeconfig。
执行前必须先获得用户确认。

命名必须严格为：

```text
pvc-<afs-name>
```

禁止追加后缀、时间戳、随机字符串或 UUID。

命令模板：

```bash
export KUBECONFIG=/root/D/<vc-name>

rayctl pvc create \
  --name pvc-<afs-name> \
  --uid <afs-uuid> \
  --secret <secret-name> \
  --ns <namespace> \
  --size 1000Mi
```

参数说明：

```text
--name      PVC 名称，必须是 pvc-<afs-name>
--uid       AFS UUID
--secret    AFS secretName
--ns        创建 namespace，默认 default
--size      PVC size，默认 1000Mi
```

创建后检查：

```bash
rayctl pvc check pvc-<afs-name>
kubectl get pvc pvc-<afs-name> -o yaml
```

如果 Pending，停止后续任务创建。

---

### 2.8 rayctl ECS / AIS

ECS / AIS 查询必须在开发机执行，通常使用 host cluster kubeconfig：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl ecs check <ais-name-or-ecs-name-or-uid>
rayctl ecs check <id-1> <id-2>
```

查询内容包括 HC 中对应的 VM、namespace、node、创建人和内网 IP。

登录 ECS / AIS 对应 VM：

```bash
rayctl ecs login <ais-name-or-ecs-name-or-uid>
```

注意：

* `rayctl ecs login` 会调用 `virtctl console`，开发机必须存在 `virtctl`。
* 登录属于交互操作；如果只是查询 VM / 节点 / 内网 IP，优先使用 `rayctl ecs check`。

---

## 3. rayctl 任务模板

### 3.1 vcluster 到模板映射

| vcluster    | 机器类型        | 芯片 / 类型      | 单机模板                 | 多机模板                |
| ----------- | ----------- | ------------ | -------------------- | ------------------- |
| a2-*        | h1ls.rp.k60a | 华为 910B      | 910b-single          | 910b-multi          |
| ai4chem     | h1ls.rp.k60a | 华为 910B      | 910b-single          | 910b-multi          |
| a3-*        | h2ls.ru.k10  | 华为 910C      | 910c-single          | 910c-multi          |
| c550-jiaofu | x2ls.ri.i80  | 沐曦/MUXI 风冷   | c550-default-single  | c550-default-multi  |
| c550-ai4s   | x2ls.ri.i80  | 沐曦/MUXI 风冷   | c550-default-single  | c550-default-multi  |
| c550-h3c    | x2ls.ri.i70  | 沐曦/MUXI 液冷   | c550-h3c-single      | c550-h3c-multi      |
| c550-mohe   | x3ls.ri.i80  | 沐曦/MUXI 超节点 | c550-superpod-single | c550-superpod-multi |

### 3.1.1 模板资源范围

| 模板族 | 加速卡资源 | 单机卡数范围 | CPU 上限 | Memory 上限 | 多机每节点固定值 | 备注 |
| --- | --- | --- | --- | --- | --- | --- |
| 910c | `huawei.com/Ascend910` | 2-16，偶数 | 256 | 1920Gi | 16 卡 / 256 CPU / 1920Gi | `910c-multi` 支持 `--logical-supernodes` |
| 910b | `huawei.com/Ascend910` | 1-8 | 144 | 1920Gi | 8 卡 / 144 CPU / 1920Gi | 一般不设置 `--logical-supernodes` |
| c550-default | `metax-tech.com/gpu` | 1-8 | 224 | 1440Gi | 8 卡 / 224 CPU / 1440Gi | 风冷，额外资源 `rdma-training/roce=1`，使用 PCI link volume |
| c550-h3c | `metax-tech.com/gpu` | 1-8 | 224 | 640Gi | 8 卡 / 224 CPU / 640Gi | 液冷，额外资源 `rdma-training/roce=1` |
| c550-superpod | `metax-tech.com/gpu` | 1-8 | 120 | 1440Gi | 8 卡 / 120 CPU / 1440Gi | 超节点，额外资源 `rdma-training/roce=1` |

固定默认值：

```text
queue: default
priorityClass: normal
host-arch: huawei-arm
shm-size: 64Gi
framework: PyTorch
```

---

### 3.2 创建任务通用要求

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

通用参数：

```text
--what-if              只生成 YAML，不真正创建 Job
--name                 job 名称
--ns                   namespace，默认 default
--framework            PyTorch 或 MPI，默认 PyTorch
--image                完整镜像地址
--command              启动命令
--secret               imagePullSecret，官方镜像可留空
--data-pvc             文件存储 PVC
--aoss-pvc             对象存储 PVC
--shm-size             shm 大小，默认 64Gi
--priority-class       priorityClass，默认 normal
```

单机模板额外参数：

```text
--cpu
--memory               数字默认按 Gi 处理
--accelerators
```

多机模板额外参数：

```text
--nodes                多机总数量，至少 2
--logical-supernodes   仅 910c-multi 使用；必须整除总机器数
```

---

### 3.3 910C 单机任务

```bash
export KUBECONFIG=/root/D/<vc-name>

rayctl job create 910c-single \
  --name <job-name> \
  --ns <namespace> \
  --image <image> \
  --command "<command>" \
  --cpu <cpu-core> \
  --memory <memory-gi> \
  --accelerators <accelerator-count> \
  --priority-class normal
```

常见默认值：

```text
cpu: 256
memory: 1920
accelerators: 16
command: sleep inf
```

---

### 3.4 910B 单机任务

```bash
export KUBECONFIG=/root/D/<vc-name>

rayctl job create 910b-single \
  --name <job-name> \
  --ns <namespace> \
  --image <image> \
  --command "<command>" \
  --cpu <cpu-core> \
  --memory <memory-gi> \
  --accelerators <accelerator-count> \
  --priority-class normal
```

常见上限：

```text
cpu: 144
memory: 1920
accelerators: 8
```

---

### 3.5 多机任务

```bash
export KUBECONFIG=/root/D/<vc-name>

rayctl job create <template-name> \
  --name <job-name> \
  --ns <namespace> \
  --nodes <node-count> \
  --image <image> \
  --command "<command>" \
  --priority-class normal
```

注意：

* c550 多机模板没有 `--logical-supernodes`
* 910B 多机一般不设置 `--logical-supernodes`
* 910C 多机如需逻辑超节点，可追加：

```bash
--logical-supernodes <count>
```

---

### 3.6 C550 / MUXI 任务

风冷：

```bash
rayctl job create c550-default-single \
  --name <job-name> \
  --ns <namespace> \
  --image <image> \
  --command "<command>" \
  --cpu <cpu-core> \
  --memory <memory-gi> \
  --accelerators <accelerator-count>

rayctl job create c550-default-multi \
  --name <job-name> \
  --ns <namespace> \
  --nodes <node-count> \
  --image <image> \
  --command "<command>"
```

液冷 / 超节点只替换模板名：

```text
c550-h3c-single
c550-h3c-multi
c550-superpod-single
c550-superpod-multi
```

---

## 4. D 集群单机操作

### 4.1 标准链路

所有 D 集群物理机器单机查询，必须走：

```text
本地 OpenClaw exec
  → ssh 堡垒机 10.140.3.216:5906
  → JumpServer 中输入 220.33 进入跳板机
  → sudo su sensetime
  → 在 sensetime 用户下执行 ansible 单 IP
```

正确执行位置：

```text
sensetime@host-10-140-220-33
```

只有到达该位置后，才允许执行：

```bash
ansible all -i '<目标IP>,' -m shell -a '<命令>'
```

---

### 4.2 本地 OpenClaw 执行 expect 的要求
如果需要由 OpenClaw 自动进入跳板机，必须生成 expect 脚本，并满足：

1. 使用明文堡垒机密码6RV&QnwPrq登录。
2. 先登录堡垒机 `10.140.3.216:5906`。
3. 在 JumpServer 中输入 `220.33` 进入跳板机。
4. 进入跳板机后必须执行 `sudo su sensetime`。
5. 到达 `sensetime@host-10-140-220-33` prompt 后，才允许执行 ansible 单 IP 命令。
6. expect timeout 必须设置上限，不能无限等待。
7. 执行完只读命令后必须正常退出。

标准 expect 逻辑：

```expect
# 登录阶段，每一步最多 30 秒
set timeout 30

spawn ssh -p 5906 test6@10.140.3.216

expect {
    "password:" {
        send "<6RV&QnwPrq>\r"
    }
    timeout {
        puts "bastion login timeout"
        exit 1
    }
}

expect {
    "Opt>" {
        send "220.33\r"
    }
    timeout {
        puts "JumpServer Opt prompt timeout"
        exit 1
    }
}

expect {
    -re {test6@host-10-140-220-33.*[$#]} {
        send "sudo su sensetime\r"
    }
    timeout {
        puts "jump host prompt timeout"
        exit 1
    }
}

# 普通只读 ansible 查询最多等 120 秒
set timeout 120
expect {
    -re {sensetime@host-10-140-220-33.*[$#]} {
        # 到达跳板机 sensetime 用户后，才允许执行 ansible 单 IP 命令
    }
    timeout {
        puts "switch to sensetime timeout"
        exit 1
    }
}
```

---

### 4.3 只读单机查询模板

以下命令都是“在跳板机 sensetime 用户下执行的命令”，不是本地命令，也不是开发机命令。

查看目标机器 `~sensetime` 家目录：

```bash
ansible all -i '<目标IP>,' -m shell -a 'ls -la ~sensetime/'
```

查看 NPU：

```bash
ansible all -i '<目标IP>,' -m shell -a 'mx-smi'
```

查看主机名：

```bash
ansible all -i '<目标IP>,' -m shell -a 'hostname'
```

查看时间：

```bash
ansible all -i '<目标IP>,' -m shell -a 'date'
```

查看磁盘：

```bash
ansible all -i '<目标IP>,' -m shell -a 'df -h'
```

查看内存：

```bash
ansible all -i '<目标IP>,' -m shell -a 'free -h'
```

查看网卡：

```bash
ansible all -i '<目标IP>,' -m shell -a 'ip addr'
```

查看路由：

```bash
ansible all -i '<目标IP>,' -m shell -a 'ip route'
```

查看 DNS：

```bash
ansible all -i '<目标IP>,' -m shell -a 'cat /etc/resolv.conf'
```

查看进程：

```bash
ansible all -i '<目标IP>,' -m shell -a 'ps -ef | head -50'
```

查看指定日志：

```bash
ansible all -i '<目标IP>,' -m shell -a 'tail -n 100 <log-path>'
```

---

### 4.4 单机命令转义规则

普通命令：

```bash
ansible all -i '10.12.x.x,' -m shell -a 'hostname'
```

命令本身包含单引号时，优先改用双引号并正确转义：

```bash
ansible all -i '10.12.x.x,' -m shell -a "awk '{print \$1}' /etc/hosts"
```

复杂命令、长命令、多行命令，优先让用户确认后再写临时脚本。

创建临时脚本属于写操作，必须先确认。

---

### 4.5 单机执行脚本模板

执行脚本可能影响机器状态，执行前必须向用户确认。

执行 MCCL 脚本：

```bash
ansible all -i '<目标IP>,' -m shell -a 'cd ~sensetime && sudo bash mccl.sh 8'
```

查看 MCCL 日志：

```bash
ansible all -i '<目标IP>,' -m shell -a 'cd ~sensetime && tail -n 100 mccltest.log'
```

---

### 4.6 不推荐方式

不要优先使用：

```bash
ssh <目标IP>
```

不要优先使用：

```bash
ansible <目标IP> -i /home/sensetime/ansible/d-cluster.ini ...
```

不要在本地执行：

```bash
ansible all -i '<目标IP>,' -m shell -a '<命令>'
```

不要在开发机执行：

```bash
ansible all -i '<目标IP>,' -m shell -a '<命令>'
```

原因：

* 本地 OpenClaw 环境通常没有 ansible。
* 开发机是 Kubernetes / rayctl / kubectl 操作机，不是 D 集群机器操作入口。
* sensetime 用户在跳板机上对 D 集群节点免密，Ansible 单 IP 应在跳板机执行。
* 单机排查时 inventory 文件可能不存在、过期或解析失败。
* 直接 SSH 容易超时。

---

## 5. 批量 Ansible 操作

批量 Ansible 也必须在跳板机 `sensetime` 用户下执行，不能在本地或开发机执行。

如果确实需要批量操作，可以使用 inventory。

inventory 参考路径：

```bash
/home/sensetime/ansible/d-cluster.ini
```

Ansible config：

```bash
export ANSIBLE_CONFIG=/home/sensetime/ansible/ansible.cfg
```

批量执行：

```bash
ansible d-cluster -i /home/sensetime/ansible/d-cluster.ini -m shell -a '<命令>'
```

批量 playbook：

```bash
ansible-playbook -i /home/sensetime/ansible/d-cluster.ini playbook.yml
```

注意：

* 批量写操作必须先确认。
* 批量重启 / 删除 / 修改文件 / 改配置属于高风险操作。
* 优先先抽一台验证，再扩大范围。

---

## 6. 运维故障记录表

用途：

* 查询节点维修记录
* 查询节点故障状态
* 查询历史处理情况

链接：

```text
https://iqeubg8au73.feishu.cn/sheets/IC10sYPdjhvLnpt2GVGc1EZNngh
```

常用工作表：

```text
故障记录表（南洋维护）
```

查询关键字：

* 节点 IP
* hostname
* 故障时间
* 维修状态

---

## 7. MCCL 测试辅助命令

MCCL 测试必须先区分场景：

* 单机 MUXI MCCL 维修验收测试：物理机单机测试，走跳板机 `sensetime` 用户下的 ansible 单 IP。
* 多机 / 平台纳管 MCCL 测试：平台 vcjob / Volcano Job 测试，才可能涉及 Kubernetes node label、PodGroup、动态排除节点等。

不要把两个场景混用。

### 7.1 单机 MUXI MCCL 维修验收测试

适用场景：维修机器、故障机器恢复后，在单台 D 集群物理机上确认 8 卡 MCCL 是否正常。

执行入口：

```text
本地 OpenClaw exec
  → ssh 堡垒机 10.140.3.216:5906
  → JumpServer 中输入 220.33 进入跳板机
  → sudo su sensetime
  → 在 sensetime 用户下执行 ansible 单 IP
```

禁止事项：

* 不要在本地或开发机执行 ansible。
* 不要创建 vcjob。
* 不要打 Kubernetes node label。
* 不要删 Kubernetes node label。
* 不要为了单机测试先 uncordon。
* 节点如果原本处于维修 / cordon 状态，单机测试期间应保持原状态。
* 只有用户明确要求“通过后 uncordon”时，测试通过后才走开发机用 `rayctl node uncordon`。

#### 7.1.1 标准单机 `mccl.sh`

如果目标机 `~sensetime/mccl.sh` 不存在，按下面内容写入。写入脚本属于写操作，必须先确认。

```bash
#!/bin/bash
export MACA_PATH=/opt/maca
export LD_LIBRARY_PATH=${MACA_PATH}/lib:${MACA_PATH}/ompi/lib
export FORCE_ACTIVE_WAIT=2
export MCCL_PCIE_BUFFER_MODE=0

GPU_NUM=4
if [[ $1 -gt 0 && $1 -lt 65 ]]; then
  GPU_NUM=$1
fi
TEST_DIR=${MACA_PATH}/samples/mccl_tests/perf/mccl_perf
BENCH_NAMES="all_reduce_perf all_gather_perf reduce_scatter_perf sendrecv_perf alltoall_perf"
#BENCH_NAMES=all_reduce_perf
MPI_PROCESS_NUM=${GPU_NUM}
MPI_RUN_OPT="--allow-run-as-root -mca pml ^ucx -mca osc ^ucx -mca btl ^openib"
for BENCH in ${BENCH_NAMES}; do
echo -n "The test is ${BENCH}, the maca version is " && realpath ${MACA_PATH}
${MACA_PATH}/ompi/bin/mpirun -x MCCL_PCIE_BUFFER_MODE -np ${MPI_PROCESS_NUM} ${MPI_RUN_OPT} ${TEST_DIR}/${BENCH} -b 1K -e 1G -d bfloat16 -f 2 -g 1 -n 10
done
```

检查目标机是否已有脚本：

```bash
ansible all -i '<目标IP>,' -m shell -a 'test -f ~sensetime/mccl.sh && ls -l ~sensetime/mccl.sh || echo MISSING'
```

写入后权限应为：

```bash
chmod 755 /home/sensetime/mccl.sh
chown sensetime:sensetime /home/sensetime/mccl.sh
```

#### 7.1.2 标准五轮八卡测试命令

推荐将输出写入目标机 `~sensetime` 下的日志文件，避免长输出在 OpenClaw 会话中截断：

```bash
ansible all -i '<目标IP>,' -m shell -a 'cd ~sensetime && LOG=mccl-$(date +%Y%m%d-%H%M%S).log; for i in 1 2 3 4 5; do echo "===== MCCL RUN ${i}/5 START $(date) =====" | tee -a "$LOG"; sudo bash mccl.sh 8 >> "$LOG" 2>&1; rc=$?; echo "===== MCCL RUN ${i}/5 END rc=${rc} $(date) =====" | tee -a "$LOG"; if test "$rc" -ne 0; then echo "LOG=$LOG"; exit "$rc"; fi; done; echo "LOG=$LOG"'
```

#### 7.1.3 读取日志和判断结果

读取日志摘要，优先看 `Avg bus bandwidth`：

```bash
ansible all -i '<目标IP>,' -m shell -a 'cd ~sensetime && grep -E "MCCL RUN|The test is|Avg bus bandwidth|Out of bounds|ERROR|Error|error|failed|Failed|timeout|Timeout|rc=" <log-file>'
```

判断重点：

* 5 轮都应出现 `END rc=0`。
* 每轮应覆盖 `all_reduce_perf`、`all_gather_perf`、`reduce_scatter_perf`、`sendrecv_perf`、`alltoall_perf`。
* 主要关注每个 benchmark 输出的 `# Avg bus bandwidth`。
* `# Out of bounds values : 0 OK` 表示校验正常。
* 如果出现非 0 rc、error / failed / timeout、长期无日志推进或缺少 benchmark，应停止后续动作并汇报原始日志。

### 7.2 多机 / 平台纳管 MCCL 测试

适用场景：通过平台创建 vcjob / Volcano Job，对多台机器进行纳管式 MCCL 测试。

该场景才可能涉及 Kubernetes node label、PodGroup、动态排除节点等。不要把本节流程用于单机维修验收测试。

#### 7.2.1 节点打标

写操作，执行前必须确认。

```bash
export KUBECONFIG=/root/kubeconfig
kubectl label node <node-name> mccl-test=<date> --overwrite
```

#### 7.2.2 节点删标

写操作，执行前必须确认。

```bash
export KUBECONFIG=/root/kubeconfig
kubectl label node <node-name> mccl-test- --overwrite
```

#### 7.2.3 查询标签

```bash
kubectl get node <node-name> --show-labels | grep mccl-test
```

批量查看：

```bash
kubectl get node -L mccl-test
```

#### 7.2.4 多机测试排障原则

* 出现坏节点时，可以动态排除。
* 排除坏节点后必须同步调整 `minAvailable`、worker replicas、`CARD_NUM`。
* 多机场景按当前最新 `mccl-test` skill 或专门工具模板执行。
* 测试结果输出时，保留原始日志，不自行改写。

---

## 8. 常用排查命令

### 8.1 查看 Kubernetes 节点资源

该命令只用于 Kubernetes Node 资源查询，必须在开发机执行。

优先使用 custom-columns，避免复杂 jq 转义。

```bash
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory,ASCEND:.status.allocatable.huawei\.com/Ascend910,MACHINE:.metadata.labels.resource\.compute\.sensecore\.cn/machine-type'
```

如果 custom-columns 不满足需求，再考虑 jq。
复杂 jq 命令优先固化为脚本，不要在交互中手写超长 jq。

建议脚本名：

```text
check_node_resource.sh
```

脚本应放在开发机，并由用户确认后再创建或修改。

---

### 8.2 查看 Pod 事件

```bash
kubectl describe pod <pod-name> -n <namespace>
```

---

### 8.3 查看 Pod 日志

```bash
kubectl logs <pod-name> -n <namespace>
```

多容器：

```bash
kubectl logs <pod-name> -n <namespace> -c <container-name>
```

---

### 8.4 查看 Volcano 任务

```bash
kubectl get vcjob -A
kubectl describe vcjob <job-name> -n <namespace>
```

---

### 8.5 查看 PodGroup

```bash
kubectl get podgroup -A
kubectl describe podgroup <podgroup-name> -n <namespace>
```

### 8.6 Pending 任务资源碎片化检查

适用场景：`rayctl job get` 或 PodGroup 显示 `NotEnoughResources`，但用户认为 vcluster 总量应该有资源。

先用 rayctl 定位任务、vcluster、namespace、PodGroup 和 inspect Pod：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl job get <job-name-or-pod-name-or-uid>
```

如果需要 PodGroup 事件，切到目标 vcluster 后用 kubectl 查询：

```bash
export KUBECONFIG=/root/D/<vc-name>
kubectl get podgroup -A | grep <vcjob-name>
kubectl describe podgroup <podgroup-name> -n <namespace>
```

切到目标 vcluster 后查看 Pod 请求和调度约束：

```bash
export KUBECONFIG=/root/D/<vc-name>
kubectl get pod <pod-name> -n <namespace> -o json | jq -r '
  "nodeName=" + (.spec.nodeName // ""),
  "schedulerName=" + (.spec.schedulerName // ""),
  "priorityClass=" + (.spec.priorityClassName // ""),
  "nodeSelector=" + ((.spec.nodeSelector // {})|tojson),
  "tolerations=" + ((.spec.tolerations // [])|tojson),
  "affinity=" + ((.spec.affinity // {})|tojson),
  (.spec.containers[] | "container=" + .name + " requests=" + ((.resources.requests // {})|tojson) + " limits=" + ((.resources.limits // {})|tojson))
'
```

按节点扣减已调度 Pod requests，判断是否存在单节点可容纳目标 Pod。下面脚本是只读检查，可按目标 Pod request 修改 `need_*` 和机器类型：

```bash
python3 - <<'PY'
import json, subprocess

def get_json(args):
    return json.loads(subprocess.check_output(args))

def cpu_to_m(v):
    if not v:
        return 0
    s = str(v)
    return int(s[:-1]) if s.endswith('m') else int(float(s) * 1000)

def mem_to_ki(v):
    if not v:
        return 0
    s = str(v)
    units = {"Ki":1, "Mi":1024, "Gi":1024**2, "Ti":1024**3, "K":1, "M":1000, "G":1000**2, "T":1000**3}
    for u, m in units.items():
        if s.endswith(u):
            return int(float(s[:-len(u)]) * m)
    return int(s) // 1024

def qty_int(v):
    if not v:
        return 0
    return int(float(str(v)))

# 按目标 Pending Pod 的 requests / affinity 修改。
need_cpu_m = 64000
need_mem_ki = 480 * 1024**2
need_accelerator = 2
accelerator_key = 'huawei.com/Ascend910'
need_machine_type = 'h2ls.ru.k10'

nodes = get_json(['kubectl', 'get', 'nodes', '-o', 'json'])['items']
pods = get_json(['kubectl', 'get', 'pods', '-A', '-o', 'json'])['items']

alloc = {}
for n in nodes:
    name = n['metadata']['name']
    labels = n['metadata'].get('labels', {})
    a = n['status'].get('allocatable', {})
    alloc[name] = {
        'machine': labels.get('resource.compute.sensecore.cn/machine-type', ''),
        'unsched': bool(n.get('spec', {}).get('unschedulable')),
        'cpu': cpu_to_m(a.get('cpu')),
        'mem': mem_to_ki(a.get('memory')),
        'acc': qty_int(a.get(accelerator_key)),
        'pods': 0,
        'used_cpu': 0,
        'used_mem': 0,
        'used_acc': 0,
    }

for p in pods:
    node = p.get('spec', {}).get('nodeName')
    phase = p.get('status', {}).get('phase')
    if not node or node not in alloc or phase in ('Succeeded', 'Failed'):
        continue
    alloc[node]['pods'] += 1
    for c in p.get('spec', {}).get('containers', []):
        req = c.get('resources', {}).get('requests', {}) or {}
        alloc[node]['used_cpu'] += cpu_to_m(req.get('cpu'))
        alloc[node]['used_mem'] += mem_to_ki(req.get('memory'))
        alloc[node]['used_acc'] += qty_int(req.get(accelerator_key))

rows = []
fit = []
reason_counts = {'accelerator': 0, 'cpu': 0, 'memory': 0, 'unsched': 0, 'nonmachine': 0}
for name, d in alloc.items():
    if need_machine_type and d['machine'] != need_machine_type:
        reason_counts['nonmachine'] += 1
        continue
    free_cpu = d['cpu'] - d['used_cpu']
    free_mem = d['mem'] - d['used_mem']
    free_acc = d['acc'] - d['used_acc']
    bad = []
    if d['unsched']:
        bad.append('unsched')
        reason_counts['unsched'] += 1
    if free_acc < need_accelerator:
        bad.append('accelerator')
        reason_counts['accelerator'] += 1
    if free_cpu < need_cpu_m:
        bad.append('cpu')
        reason_counts['cpu'] += 1
    if free_mem < need_mem_ki:
        bad.append('memory')
        reason_counts['memory'] += 1
    row = (name, free_acc, free_cpu / 1000, free_mem / 1024**2, d['used_acc'], d['used_cpu'] / 1000, d['used_mem'] / 1024**2, d['pods'], ','.join(bad) or 'FIT')
    rows.append(row)
    if not bad:
        fit.append(row)

print(f'NEED: accelerator={need_accelerator} cpu={need_cpu_m/1000:g} memoryGi={need_mem_ki/1024**2:g} machine={need_machine_type or "<any>"}')
print('candidate nodes:', len(rows), 'fit:', len(fit), 'reason_counts_overlap:', reason_counts)
print('TOP_FREE_CANDIDATES name free_acc free_cpu free_memGi used_acc used_cpu used_memGi pods status')
for r in sorted(rows, key=lambda x: (x[1], x[2], x[3]), reverse=True)[:30]:
    print('%s\t%s\t%.3f\t%.1f\t%s\t%.3f\t%.1f\t%s\t%s' % r)
print('FIT_NODES')
for r in sorted(fit):
    print('%s\tfree_acc=%s\tfree_cpu=%.3f\tfree_memGi=%.1f\tpods=%s' % (r[0], r[1], r[2], r[3], r[7]))
PY
```

判断口径：

* `node.status.allocatable` 作为节点可分配总量。
* 已使用量按已调度且未完成 Pod 的 `resources.requests` 累加。
* 只统计 `spec.nodeName` 非空且 phase 不是 `Succeeded` / `Failed` 的 Pod。
* 目标 Pod 可调度需要同一台节点同时满足 accelerator、CPU、memory、machine-type、nodeSelector / affinity、非 unschedulable 等条件。
* 如果 `fit: 0`，但各类资源在不同节点上分别还有剩余，通常就是资源碎片化。

---

## 9. Skills 说明

当前 D 集群日常操作不依赖旧的 `d-cluster` skill。

如果 OpenClaw 日志出现：

```text
Skipping escaped skill path outside its configured root
reason="symlink-escape"
```

说明该 skill 没有被安全加载，不应假设该 skill 生效。

建议：

* 禁用或移走旧的 `d-cluster` symlink。
* D 集群日常操作以 `MEMORY.md` / `TOOLS.md` 中的入口规则为准。
* `mccl-test` skill 仅在 MCCL 测试场景下使用。
---

## 10. 本地偏好

Preferred TTS voice:

```text
Nova
```

Default speaker:

```text
Kitchen HomePod
```
