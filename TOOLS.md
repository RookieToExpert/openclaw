# TOOLS.md - Environment Tools and Command Templates

本文件记录具体工具、入口、机器、命令模板和执行注意事项。
长期原则、安全红线、对象生命周期规则放到 `MEMORY.md`。

---

## 0. 最高优先级：入口路由规则

执行任何命令前，必须先判断用户请求属于哪一类。

| 请求类型                                                                                                              | 正确入口                                         | 示例                            |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------- | ----------------------------- |
| Kubernetes / vcluster / rayctl / kubectl / vcjob / Pod / PVC / PV / AFS / Service / Endpoint / Webhook / PodGroup | D 集群开发机                                      | 查任务、查 Pod、查 PVC、创建任务、创建 PVC   |
| 查询某个 IP / hostname 属于哪个 vcluster                                                                                  | D 集群开发机 + `rayctl node get -A`               | “10.12.x.x 属于哪个 vc”           |
| 查看某台 D 集群机器上的文件、目录、日志、进程、磁盘、内存、NPU、网卡、路由、DNS、本地脚本                                                                 | 堡垒机 → 跳板机 → sensetime → ansible 单 IP         | “看 10.12.x.x 的 ~sensetime 目录” |
| 在某台 D 集群机器上执行脚本、改配置、删除文件、重启服务                                                                                     | 堡垒机 → 跳板机 → sensetime → ansible 单 IP，并且必须先确认 | “在 10.12.x.x 上跑 mccl.sh”      |

禁止误判：

* “看某台机器目录 / 文件 / 日志”不是 Kubernetes Node 查询。
* “看某台机器 mx-smi / 磁盘 / 内存 / 网卡 / DNS”不是 Kubernetes 查询。
* 只有“节点归属 vcluster 查询”才走开发机上的 `rayctl node get -A`。
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

优先 rayctl：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl job get job <job-name>
rayctl job check <job-name>
```

如果需要进入 vcluster 再查：

```bash
export KUBECONFIG=/root/D/<vc-name>
kubectl get vcjob -A | grep <job-name>
kubectl get pod -A | grep <job-name>
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

### cp2.5 查询节点归属 vcluster

```bash
export KUBECONFIG=/root/kubeconfig
rayctl node get -A | grep <ip-or-hostname>
```

结果最后一列是 vcluster 名称。

示例：

```bash
rayctl node get -A | grep 10.12.138.26
rayctl node get -A | grep host-10-12-138-26
```

不要进入 vcluster 内部倒查节点归属。

---

### 2.6 查询 AFS / PVC / PV

查询 AFS：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl afs check <afs-name>
```

查询 PVC：

```bash
export KUBECONFIG=/root/D/<vc-name>
rayctl pvc check <pvc-name>
kubectl get pvc <pvc-name> -o yaml
```

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
  --secret <secret-name>
```

创建后检查：

```bash
rayctl pvc check pvc-<afs-name>
kubectl get pvc pvc-<afs-name> -o yaml
```

如果 Pending，停止后续任务创建。

---

## 3. rayctl 任务模板

### 3.1 vcluster 到模板映射

| vcluster    | 机器类型        | 芯片       | 单机模板                 | 多机模板                |
| ----------- | ----------- | -------- | -------------------- | ------------------- |
| a2-*        | 910B        | 910B     | 910b-single          | 910b-multi          |
| ai4chem     | 910B        | 910B     | 910b-single          | 910b-multi          |
| a3-*        | 910C        | 910C     | 910c-single          | 910c-multi          |
| c550-jiaofu | x2ls.ri.i80 | 910B 风冷  | c550-default-single  | c550-default-multi  |
| c550-ai4s   | x2ls.ri.i80 | 910B 风冷  | c550-default-single  | c550-default-multi  |
| c550-h3c    | x2ls.ri.i70 | 910B 液冷  | c550-h3c-single      | c550-h3c-multi      |
| c550-mohe   | x3ls.ri.i80 | 910B 超节点 | c550-superpod-single | c550-superpod-multi |

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

### 7.1 节点打标

写操作，执行前必须确认。

```bash
export KUBECONFIG=/root/kubeconfig
kubectl label node <node-name> mccl-test=<date> --overwrite
```

---

### 7.2 节点删标

写操作，执行前必须确认。

```bash
export KUBECONFIG=/root/kubeconfig
kubectl label node <node-name> mccl-test- --overwrite
```

---

### 7.3 查询标签

```bash
kubectl get node <node-name> --show-labels | grep mccl-test
```

批量查看：

```bash
kubectl get node -L mccl-test
```

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
