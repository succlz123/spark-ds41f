# tp4-ring4 部署与调优记录

做了哪些改动、为什么改、改完实测多少，以及以后在另一批 GB10 上照着重现时需要准备什么。

## 0. 现状（2026-11-21）

| 项 | 当前 |
|---|---|
| 镜像 | `spark-ds41f:1120`（本机构建，`sha256:e871fa85…`；官方发布 digest `sha256:5c124725…`，构建差异只有 Dockerfile 里的 pip 镜像，见 §10） |
| 基线 | upstream main（本轮 rebase 到最新；`site` = main + 一个提交，不堆 merge） |
| 权重 | 官方 `deepseek-ai/DeepSeek-V4.1-Flash-1017` revision `1bae0e1088993b9b82de08f59de5be5a34e42170`，四台 48/48 分片 sha256 全对 |
| 形状 | tp4-switchless-ring，32 席，KV 16.5 GiB（官方值），步预算 16384，预填块 16192，capture 192，K=5，image 16 |
| GLP | 已启用：`deepseek-v4.1-flash-weightless`（向量文件四台都有，跟官方逐项一致） |
| near-tie | **已移除**（1120 换了 TileLang 拒绝采样内核，补丁要重写；未再启用，文档与 vendor 里都已删） |
| Yee 召回 | **未启用**（代码与文档保留在 §6/§12，1120 上需重新验证） |
| 与官方差异 | serve_args 只多 `--enable-per-request-metrics`；environment / mounts / docker_run_args **逐条一致** |

> §5/§6 里的数字是 1107 时代的实测记录，留着是为了「为什么最后选了官方默认值」的证据；1120 上重扫的结果见每节末尾的注。

- 硬件：4×DGX Spark GB10，CX7 四机环网（switch-less，`oneshot-ring4`）

## 1. 拓扑与节点表

环序（物理接线）：`nvidia01(.22) — h3c01(.45) — dell02(.38) — dell01(.30) — nvidia01`

| rank | 主机 | 管理 IP | 主机名 |
|---|---|---|---|
| 0 | nvidia01（head） | 192.168.8.22 | spark-2837 |
| 1 | h3c01 | 192.168.8.45 | DGX-a |
| 2 | dell02 | 192.168.8.38 | promaxgb10-0806 |
| 3 | dell01 | 192.168.8.30 | promaxgb10-cb8a |

> asus01（.37）原本是 rank0，2026-11-20 起换成 nvidia01；asus01 已无 CX7 网卡，退出环网。

CX7 地址（netplan，MTU 9000，每对邻居两张卡各一条，RoCE v2，GID index 3）：

| 主机 | enp1s0f0np0 | enP2p1s0f0np0 | enp1s0f1np1 | enP2p1s0f1np1 |
|---|---|---|---|---|
| nvidia01 | 10.40.3.1/24 | 10.41.3.1/24 | 10.40.0.1/24 | 10.41.0.1/24 |
| h3c01 | 10.40.0.2/24 | 10.41.0.2/24 | 10.40.1.2/24 | 10.41.1.2/24 |
| dell02 | 10.40.1.3/24 | 10.41.1.3/24 | 10.40.2.3/24 | 10.41.2.3/24 |
| dell01 | 10.40.2.4/24 | 10.41.2.4/24 | 10.40.3.4/24 | 10.41.3.4/24 |

> 节点表 `config/nodes-ring4.local.json` 被 `.gitignore` 的 `*.local.json` 排除，不入库。
> 新部署时按上表重建：`rank`/`management_ip`/`head`，`roce_peer_hcas` 指该主机连的两个邻居 rank，
> `roce_subnets` 填该主机四个口的网段。head 的 `management_ip` 必须等于 cluster.json 的
> `distributed.master_addr`，否则新版 doctor 会硬报错
> `ERROR: distributed.master_addr must match the API head's management_ip`（检查在 `scripts/topology.py`）。

## 2. 宿主机前置条件（每台都要）

1. **64k 内核**：`7.0.0-1019-nvidia-64k` + `CONFIG_ARM64_64K_PAGES`，`getconf PAGESIZE` = 65536。
   GRUB 的 pin 写在 drop-in `/etc/default/grub.d/zz-spark-kernel-trial.cfg`（会覆盖 `/etc/default/grub` 里的 `GRUB_DEFAULT`）。
2. **memory saver**：DKMS 包 `dgx-spark-memory-saver 0.4.1`，装好 64k 内核后
   `dkms install -m dgx-spark-memory-saver -v 0.4.1 -k <kver>`；判据是 `modinfo nvidia-uvm | grep srcversion`
   = `37A4D127AE60F1F3AE60A02`（未打补丁的是 `09F2A78862D06017E9324B6`）。
   另需 `/swap-64k.img`（16 GiB，mkswap 后开机 swapon）与 `spark-ds41f-kernel-memory.service`。
3. **免密 sudo**：`/etc/sudoers.d/succlz123` = `succlz123 ALL=(ALL) NOPASSWD:ALL`（memguard 要用）。
4. **CX7 netplan**：四个 RoCE 口静态地址 + MTU 9000 + `link-local: []`，见上表；验证方式是邻居间
   `ping -M do -s 8972 -I <本机口> <对端口>`，非相邻直连应不通。
5. **权重**：`{home}/.cache/huggingface/hub/models--deepseek-ai--DeepSeek-V4.1-Flash-1017`，
   revision `1bae0e10…`，83 个文件 / 510,311,045,295 字节，48 个分片。
   判据：`sha256sum model-*-of-00048.safetensors` 与官方 LFS oid 逐条相等（2026-11-21 四台全部核对过）。
   历史坑：这台机器上曾经有一份 MoonMath 的「uncensored」副本，其中 00014–00036 这 23 个分片与官方不同；
   混着用会让四个 rank 的权重不一致、输出质量下降。现在四台都是官方版本。
   其余三台的目录名仍是 `models--MoonMath-ai--…-uncensored`，另建了
   `models--deepseek-ai--DeepSeek-V4.1-Flash-1017` 软链指向它，profile 统一用官方路径。
6. **镜像**：`spark-ds41f:1120` 四台都有（27 GB 压缩 / 更大解压后）。
7. **ssh 互信**：head 用 `~/.ssh/id_ed25519_spark` 免密到另外三台；每台 `~/.ssh/config` 加
   `Host <四台管理 IP>` + `IdentityFile ~/.ssh/id_ed25519_spark` + `IdentitiesOnly yes`。

## 3. GLP 权重转向（weightless steering）

官方做法是在 profile 里写 `glp` 块，由 recipe 渲染出第二个 served name、两个环境变量和只读挂载：

| profile 字段 | 值 |
|---|---|
| `glp.served_model_name` | `deepseek-v4.1-flash-weightless` |
| `glp.repo` | `msuiche/DeepSeek-V4.1-Flash-1017-abliterated-cyber-GLP-39-L1-39-a0.75` |
| `glp.revision` | `2715ffd51f59899115853e1118db080536966c44` |
| `glp.file` | `glp.deepseek-v41-flash-1017-GLP-39-L1-39-a0.75.gguf` |
| `glp.sha256` | `9c66d47701e0ceb42c33906f2d3789c0f1f16b21e7d97185f20f7f3f09fbf5cf` |

渲染出来是 `WEIGHTLESS_STEER_PATH=/glp/vector.gguf`、`WEIGHTLESS_STEER_MODEL=deepseek-v4.1-flash-weightless`
和 `[.../glp.…gguf, /glp/vector.gguf, ro]` 一条挂载。注意 profile 里同时要把它加进
`--served-model-name`（放在原模型名之后），否则 doctor 报
`glp.served_model_name … must be served once, after the stock name`。

文件 803,392 字节，仓库是 **gated(auto)**，每台节点都要有：

```bash
# 先在网页上点一次 Agree（或账号已在授权列表），然后
export HF_ENDPOINT=https://hf-mirror.com
hf download msuiche/DeepSeek-V4.1-Flash-1017-abliterated-cyber-GLP-39-L1-39-a0.75 \
  glp.deepseek-v41-flash-1017-GLP-39-L1-39-a0.75.gguf --revision 2715ffd51f59899115853e1118db080536966c44
# 或直接 curl -sSL（注意必须 -L，否则只拿到 302 的 1282 字节）
```

请求打到 `deepseek-v4.1-flash-weightless` 才走这层残差改写，原模型名不受影响。
它只改第 1–39 层之后残差方向的投影（α=0.75），不动权重；README 也明确「未在其他 revision / 量化版上验证」，
且会削弱 prompt 注入抵抗力。

## 4. profile 相对 upstream 的差异

### 4.1 serve_args

**现在只剩一处**：

| 参数 | 上游默认 | 本 profile | 理由 |
|---|---|---|---|
| `--enable-per-request-metrics` | 无 | **开** | 压测时按请求看指标；不影响调度与采样 |

其余逐项相同：`--port`（我们 8888、官方 8000，站点约定）、`--served-model-name`（多一个 weightless 名）、
模型路径、`--max-num-seqs 32`、`--kv-cache-memory-bytes 17716740096`、`--max-num-batched-tokens 16384`、
`--long-prefill-token-threshold 16192`、`--max-cudagraph-capture-size 192`、`--speculative-config`（K=5）、
`--limit-mm-per-prompt {"image":16}`、`--override-generation-config {"temperature":1.0,"top_p":1.0}` 等。

### 4.2 environment

**完全一致**（含 `WEIGHTLESS_STEER_*`、`SPARKNET_ROCE_*`、`VLLM_DS41_L2_PREFETCH_*`）。
本批机器的管理网卡是 `wlP9s9`，官方是 `enP7s7`，`NCCL_SOCKET_IFNAME` / `GLOO_SOCKET_IFNAME` /
`TP_SOCKET_IFNAME` 三处按实际网卡名填。

### 4.3 mounts 与 run_args

**逐条一致**。官方 1120 起不再需要 `/dev/dri/card0`（carve-out 走 `/dev/nvidiactl`），
`--device-cgroup-rule` 那两条也已删掉。

### 4.4 其它字段

- `host.kernel` = `{ "release": "7.0.0-1019-nvidia-64k", "page_size": 65536, "memory_saver_required": true }`
- `host.fan_control` = `false`（ASUS GX10 的 EC 风扇不受内核管，`dgx-spark-fan-control` 的 DMI 白名单只认
  `NVIDIA/NVIDIA_DGX_Spark/P4242`，在 GX10 上 `No such device`，已卸载不要重装）
- `distributed.master_addr` = head 的**管理 IP**（本批是 `192.168.8.22`）
- `deployment.repository` 指向 fork（走 ghfast.top 代理），`deployment.branch` = `site`

## 5. 参数扫描的结论（为什么最后回到官方默认值）

> 以下是 1107 时代的实测；2026-11-21 在 1120 上复核过 A/B，结论见本节的「1120 复核」。

**并发席位 16 → 32**：c32 聚合吞吐明显上台阶，KV 池不变。graph 内存每 rank 约 +1 GiB。1120 已把它收进官方默认。

**KV 10.5 GiB → 20 GiB**：单请求速度不变（KV 不参与解码计算），收益只在并发上下文容量与 prefix cache 保留量。
20 GiB 时 KV = 22,875,808 tokens（1M 上下文并发 21.8 路）。1120 上因为多了 engram / L2 prefetch 常驻，
按官方 16.5 GiB 即可，已回退。

**步预算 16K → 40K / 预填块 16K → 32K**：1107 上只影响预填，25K 级上下文从「切两段」变成一段，
冷预填 7563 tok 1.19 s（6379 tok/s）、23555 tok 2.93 s（8027 tok/s），比原来快 25~80%。

**1120 复核（2026-11-21）**：`40960/32768` 与官方 `16384/16192` 在同一镜像、同 32 席、同 16.5 GiB KV 下对比，
decode（T=0 c1/c8/c32 = 79.2/436.6/839.7 vs 73.8/459.4/852.6，T=1 = 76.2/221.5/431.9 vs 77.9/221.1/430.3）、
预填（7.5K 6237 vs 6779 tok/s、23.6K 8607 vs 8570、55.3K 9202 vs 9022）、tok/步（2.41/2.55 两边相同）
全在噪声内 → **1120 的调度自己就够了，取官方 16384/16192**，少一个偏离项。

**草稿数 K**（1107，T=1 数据）：

| K | T=0 c1/c8/c32 | T=1 c1/c8/c32 | 结论 |
|---|---|---|---|
| 5 | 75.0 / 366.4 / 989.6 | 77.7 / 212.4 / 405.7 | 保留（官方也是 5） |
| 6 | 76.1 / 438.8 / 759.9 | 73.2 / 222.3 / 387.9 | 无收益，回退 |
| 12 | 62.6 / 542.7 / 912.1 | 70.4 / 250.4 / 436.9 | 低并发明显变慢，回退 |

### 5.1 哪些改动会降质

| 改动 | 会不会降质 | 依据 |
|---|---|---|
| 32 席 / 16.5 GiB KV / 16384 步预算 / 16192 预填块 | 不降 | 纯调度与内存预算，不影响采样 |
| GLP 权重转向（`deepseek-v4.1-flash-weightless`） | 只在用那个名字时生效 | 残差投影，不改权重；原模型名不受影响 |
| 召回 + qhash（见 §6） | 不降 | 草稿仍由目标模型逐 token 验证，命中错了只是被拒 |

## 6. Yee 召回（vllm_d_recall，当前未启用）

同 prompt 重复请求（copy-heavy 的最好情况），512 输出，服务端 `vllm:generation_tokens_total` 差值，**1107 实测**：

| 配置 | T=0 c1 | T=0 c8 | T=0 c32 | T=1 c1 | T=1 c8 | T=1 c32 |
|---|---|---|---|---|---|---|
| 无召回 | 75.0 | 366.4 | 989.6 | 77.7 | 212.4 | 405.7 |
| 召回 + `NONGREEDY=1` | 144.9 | 749.0 | 1104.8 | 99.0 | 295.8 | 525.1 |
| 召回，`NONGREEDY=0` | 75.3 | 641.6 | 961.6 | 102.0 | 304.9 | 518.0 |

结论：召回真正赚的是**贪心 / copy-heavy** 流量（T=0 c1 75→145、c8 366→642）；温度 > 0 那一档的「提速」
当年其实来自 near-tie。`NONGREEDY=1` 会把被覆写位置的分布压成单点（官方文档自报总变差 0.39–0.90），
掉质量不换速度，因此设 `0`。

召回命中（压测后 `/recall/stats.json`）：`ids 263101`、`hot_hits 2215`、`cold_hits 1625`、
`landed_tokens 16525`、`deduped_ids 24807`、`gamma 5`；启动日志
`[vllm-d-recall] engine ready dir=/recall backend=disk … gamma=5`。

全新内容（每个请求不同 prompt）：T=0 79.0 / 245.2 / 446.5，T=1 91.4 / 288.3 / 503.6 —— 没有退化。

**1107 上 KAD 必须关**：KAD 的变长 `propose` 包装器签名是 0.28.1rc1 的，而 1107 的 runner 会传
`num_speculative_tokens=`，打开就
`TypeError: DFlashSpeculator.propose() got an unexpected keyword argument 'num_speculative_tokens'`，
引擎起不来、CLI 自动回滚。**1120 的 `DFlashSpeculator.propose` 已带这个参数**，理论上可以开，待验证。

## 7. 部署 / 重部署步骤

```bash
# head 上
cd ~/projects/spark-ds41f
git fetch origin site && git checkout --detach origin/site
./bin/spark --cluster-config experiments/2026-11-19-gb10-ring4/cluster.json cluster sync --apply
./bin/spark --cluster-config experiments/2026-11-19-gb10-ring4/cluster.json cluster stop --apply
./bin/spark --cluster-config experiments/2026-11-19-gb10-ring4/cluster.json cluster start --replace --apply --change-shape
```

- `--change-shape` 在条目里定义了 `shape` 且 KV/席位形状变了时必须加，否则报
  `ERROR: this start changes the engine's kv …; pass --change-shape`。
- `cluster sync` 会要求工作区干净：profile 目录必须已提交（untracked 会报
  `ERROR: deployment requires a clean local repository`）。
- **先 checkout 再 sync**：`deployment` 用的提交就是本地 HEAD，如果刚 force-push 过 site，
  节点 checkout 还停在旧 sha，sync 会报
  `refusing to deploy <旧sha>: it is not published on origin/site`。
- 就绪约 200 s；TileLang 内核缓存被清空时第一次会到 20 分钟以上（1120 首次实测 `+1319.5s`），
  之后 `cache/` 有缓存就是 ~200 s。
- 验证：`curl -s localhost:8888/v1/models`（应有两个名字），再发一条 1500 `max_tokens` 的请求
  （thinking 模型 `max_tokens` 给小了只会返回 reasoning、`content: null`）。

## 8. 踩坑清单

1. **master_addr 被还原**：Mac 工作区的 profile 会莫名回到 `192.168.8.37`（旧 md5 `15c90e9c…`），
   每次提交前核对；错的话 doctor 报 `master_addr must match the API head's management_ip`。
2. **权重不完整比镜像问题更像 tokenizer 问题**：`Couldn't instantiate the backend tokenizer …`
   实际是某个节点缺 shard（悬空符号链接、blob 数不到 56）。
3. **权重版本混用**：目录名是 `MoonMath…uncensored` 不代表内容一致；00014–00036 这 23 片可能和官方不同，
   四个 rank 混用会掉质量。判据只有 sha256 对官方 LFS oid。
4. **节点间 ssh 用错 key**：默认 `id_ed25519` 不通，互信的是 `id_ed25519_spark`，要写进 `~/.ssh/config`。
5. **git 拉取**：节点直连 github 超时，四台都配代理前缀（构建用 `ghfast.top`，见 §11）。
6. **macOS 的 sed**：必须 `sed -i '' …`，否则 `invalid command code`；改 JSON 建议用 python。
7. **`ssh host 'setsid nohup … &'`** 有时让本地 ssh 挂住，长任务用后台作业跑。
8. **doctor 的假阴性**：不带 GPU 的容器里跑 `yee doctor` 会全 FAIL。

## 9. 回滚

- 想回到上游默认吞吐形状：`--max-num-seqs 16`、删掉 `--kv-cache-memory-bytes`、`--max-num-batched-tokens 16384`、
  `--long-prefill-token-threshold 16192`、capture size 96，然后带 `--change-shape` 重启。
- 想去掉 GLP：删 `glp` 块、`--served-model-name` 里的 weightless 名、两个 `WEIGHTLESS_STEER_*` 和那条挂载。
- 想回去跑 1107：`container.image` 改回 `spark-ds41f:1107` 并把 expected_labels 一起改回去。

## 10. 在受限网络里构建 1120 镜像（本机实测的三处绕行）

`bin/spark build prepare` + `build image --apply` 默认假设能直连 GitHub 与 PyPI，国内必须绕行：

1. **git 源**：`gh-proxy.com` 拉不动 `NVIDIA/cutlass` 这类大仓库（`fetch-pack: unexpected disconnect` /
   `early EOF`，submodule 反复 retry）。改成 **`ghfast.top`** 前缀即可（实测 cutlass 244 MB 在 90 s 内完整克隆）：
   `git config --global url."https://ghfast.top/https://github.com/".insteadOf "https://github.com/"`。
2. **pip（宿主机）**：`prepare_wheels` 用宿主 python 下载 `nvidia-cutlass-dsl*` 等 wheel，默认索引不通会报
   `JSONDecodeError` / `No matching distribution`。在 `~/.config/pip/pip.conf` 写
   `index-url = https://mirrors.cloud.tencent.com/pypi/simple`。
3. **pip（镜像内）**：Dockerfile 的 `tilelang-builder` / `sparknet-builder` / `vllm-native-builder` 阶段会联网装
   build 依赖（`cython>=3.1.0` 等），容器里同样拉不到 PyPI，报
   `Could not find a version that satisfies the requirement cython>=3.1.0`。已在 `docker/Dockerfile` 三个 stage
   的 `FROM` 之后加：
   ```dockerfile
   ENV PIP_INDEX_URL=https://mirrors.cloud.tencent.com/pypi/simple \
   PIP_TRUSTED_HOST=mirrors.cloud.tencent.com
   ```
   只改索引地址，包内容不变；这是与上游 Dockerfile 的唯一差异。

构建期间 `build image --apply` 会拒绝在有 DS4.1 服务运行时动手（`a DS4.1 service is running on this host`），
要先 `cluster stop --apply`；它还会检查工作区是否干净（`the image records the deployment commit; commit changes first`）。

## 11. 1120 之后的三处运维坑（2026-11-21）

- **仓库地址必须与节点 origin 的「改写后」URL 完全一致。** 四台节点都有
  `git config --global url."https://ghfast.top/https://github.com/".insteadOf https://github.com/`
  （构建 1120 时为了拉 `NVIDIA/cutlass` 子模块加的）。`cluster sync` 拿
  `git remote get-url origin` 的**改写后**结果去比 `deployment.repository`，两边字符串不等就报
  `origin '<url>' is not the configured repository`。现在两边统一写成
  `https://ghfast.top/https://github.com/succlz123/spark-ds41f.git`。
- **节点上不能 push**：ghfast.top 是只读代理、节点也没有 GitHub 凭据。改完提交后从 Mac 推：
  ```bash
  git fetch succlz123@192.168.8.22:projects/spark-ds41f HEAD
  git push --force-with-lease=site:<旧sha> origin FETCH_HEAD:site
  ```
- **`git checkout` 会被本地改动的 `docker/Dockerfile` 挡住**（`Your local changes would be overwritten`）：
  补丁已经在 site 提交里，先 `git checkout -- docker/Dockerfile` 再 fetch/checkout。

## 12. Yee 3.8 召回 / qhash 要点

### 12.1 它是什么

`vllm_d_recall` 是给 vLLM 装的跨请求召回层：把历史「提示 → 生成」记录落盘，再遇到相同或相似提示时，
直接把这些 token 当草稿交给投机解码器，由目标模型逐 token 验证——**不改分布**。qhash 是它的提问哈希索引，
需要 `disk` 后端。

### 12.2 vLLM 的开关（`vllm_d_recall`）

| 变量 | 用法 | 说明 |
|---|---|---|
| `VLLM_D_RECALL` | `1` | 总开关，靠挂在容器里的 `.pth` 自动 import |
| `VLLM_D_RECALL_DIR` | `/recall` | 记录目录，宿主机 `{home}/spark-recall` |
| `VLLM_D_RECALL_BACKEND` | `disk` | qhash 必须 disk；`memory` 重启即丢 |
| `VLLM_D_RECALL_GIB` | `1` | 记录容量上限 |
| `VLLM_D_RECALL_QHASH` | `1` | 提问哈希索引 |
| `VLLM_D_RECALL_NONGREEDY` | `0` | 关，理由见 §6 |
| `VLLM_D_RECALL_KAD` | `0` | 1107 上必须关；1120 理论上能开，待验证 |

> 当前 profile **没有**这些变量和对应挂载（1120 上还没重新验证）。

### 12.3 记录目录里有什么

`chunk-%08d.ids`（int32 小端 token id）、`identity`（分词器摘要 + 词表大小 + 模型名，不匹配会报
`RecallIdentityError`）、`lock` / `generation`（多进程共享写入）、`stats.json`、`qhash/`。
升级或换镜像时**同分词器可以直接读旧目录**；换模型/分词器必须另开目录。

### 12.4 启动日志判据

`[vllm-d-recall] hooks installed, <N> module(s) pending import` →
`patched vllm.v1.worker.gpu.spec_decode.dflash.speculator.DFlashSpeculator.propose (covers DSpark)` →
`[vllm-d-recall] engine ready dir=/recall backend=disk ids=<条数> state=<状态> gamma=5`。

### 12.5 装法（每台宿主都要有，镜像内不带）

```bash
mkdir -p ~/spark-patches ~/spark-recall
python3 -m zipfile -e vendor/yee-3.8.0-py3-none-any.whl ~/spark-patches/yee/
# 解出来是 recall_accel/ vllm_d_recall/ yee/ yee-3.8.0.dist-info/ 四个目录
cp vendor/yee_autoload.pth ~/spark-patches/
```

`yee_autoload.pth` 两行：`/opt/yee` 与 `import vllm_d_recall.autoload`；
profile 里加两条挂载（`{home}/spark-patches/yee → /opt/yee` 只读、
`.pth → /usr/local/lib/python3.12/dist-packages/zzz-yee-autoload.pth` 只读）和 `{home}/spark-recall → /recall` 读写。

### 12.6 验证（`yee doctor`）

必须在**跑着的容器里**跑，无 GPU 的 `docker run` 会假阴性（`vllm._C` 缺失导致 `PassConfigKey` AttributeError）：

```bash
docker exec -e PYTHONPATH=/opt/yee:/opt/spark-ds41f/candidate/vllm \
  spark-ds41f python3 -m yee doctor
```

要看到 `GPUModelRunner`、`RequestState`、`InputBatch`、`BaseSpeculator.propose`、`get_tp_group` 全 OK，
且 `DSparkSpeculator present`。

### 12.7 回退

去掉 `VLLM_D_RECALL` 与那几条挂载（或把 `VLLM_D_RECALL=0`）即可，`/recall` 里的记录留着不动。

## 13. 图片上限与「丢最旧」的滑窗（待实现）

上游 1120 的 tp4-ring 默认是 `--limit-mm-per-prompt {"image":16}`：**单个请求**最多 16 张图，
第 17 张直接 400。vLLM 没有「超出就丢最旧」的策略——它只做硬限制
（`vllm/v1/engine/input_processor.py:522`）。

要做「历史里超过 16 张就自动丢最早那张」的滑窗，只能在服务前面加一层代理：

- 位置：head 节点 `127.0.0.1:8890`，systemd 常驻，转发到 `127.0.0.1:8888`；new-api 渠道 base_url 改成 `http://192.168.8.22:8890`。
- 逻辑：对 `/v1/chat/completions`、`/v1/responses` 解析 `messages`，按出现顺序数 `image_url` 部分；
  总数 > 16 时从**最早**的消息开始删图片部分，直到 16；如果最后一条消息（本轮新输入）自己就 > 16 张，
  直接返回 400 `too many images in this turn`。
- 只改请求体，响应流原样透传（SSE 不能缓冲）。
- 代价：多一跳、所有流量都过它；好处是任何客户端都不用自己裁历史。

未实现。
