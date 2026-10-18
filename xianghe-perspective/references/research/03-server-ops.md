# 湘河思维蒸馏 · AI 服务器部署运维维度 · 素材分析（Agent C）

> 语料范围：已提炼《思维方式_服务器部署运维.md》+ 素材目录 12 份文档（农业AI知识库 Ubuntu 部署、WeKnora 部署/带图MD手册、Whisper、douyin-distiller、启动运维、Prometheus 监控、bge-reranker、Qwen2.5-VL-7B 方案、Qwen3-VL-8B 部署文档/压测报告/替换方案）。
> 分析目标：捕捉 HOW 思考（心智模型/决策启发式/表达DNA/价值观/反模式），不是 WHAT 说了什么。
> 脱敏：账号/密码/API key/token/邮箱/私有 URL 一律以「[凭据已脱敏]」代替；SSH 信息以「SSH 凭据已脱敏」代替；端口号、显存数、模型名保留。

---

## 一、心智模型候选（10 个）

### M1. 环境勘察优先："先摸清现状再动手，别瞎整"
- **一句话**：任何新部署开始前，先盘点现有系统装了什么、端口占了哪些、显存/内存余量、容器数、虚拟环境，并把"现有规范"当作对齐基准，绝不另起炉灶。
- **证据**：
  - 《农业AI知识库服务器部署文档（Ubuntu）》§0："顺序逻辑：GPU 软件栈（驱动→CUDA→Python）必须自底向上；数据库与中间件无 GPU 依赖；向量服务是核心、最易出错，放中间调试；最后联调。" 以及 "**不要贸然重装**（可能丢 OEM 的 RAID/驱动配置）"。
  - 《WeKnora 部署文档》§0："服务器现状（已核验）"逐项列系统/CPU/内存/磁盘/GPU/Docker/防火墙；§8："现有服务（5432/8080/8001/9000/3306/6030/8090）**全程未改动**，完全隔离。"
  - 《Qwen2.5VL7B_多模态部署方案》§一："现有规范（**已核实，作为对齐基准**）"；§三："用户反馈现有多套 vLLM 环境曾'弄乱套'，故新建专用环境"。
  - 《Whisper 部署文档》§3："目录与规范（**严格按现有服务器规范**）"。
- **出现文档**：全部部署类文档（12 份中 10 份以"现状/规范"开篇）。

### M2. 显存精算到小数位："32G 是硬总账，留 3G+ 余量"
- **一句话**：每个服务占多少 G 都列成清单算总账，加新服务前先问"还够不够"，不够就停/换/降参数，绝不是"先装上再说"。
- **证据**：
  - 《启动与运维文档》§四："Qwen3-VL-8B 13.78G + Embedding 10.76G + rerank 2.4G + Whisper 1.49G = **28.46G / 32G，free ~3.6G**"。
  - 《Qwen3-VL-8B 部署文档》§6.2："max seq len 4096 needs 0.56 GiB KV, available 0.47 GiB —— **KV cache 差 0.09G**；gpu-memory-utilization 0.55→**0.57**，KV 可用升至 1.09G"。
  - 《压测报告》§4.2："free 5G 在 8B 的并发场景下完全够用，且作为安全缓冲是合理的（**不必再压榨**）"。
  - 《Whisper 部署文档》§2："显存 used 26962 / free 5149 MiB → whisper ~76 MiB → used 28534 / free 3578 MiB"。
  - 《农业AI部署文档》显存表："Embedding ~8G + Reranker ~10G + **预留 ~14G** KV/显存管理余量"。
- **出现文档**：启动运维、Qwen3-VL-8B、压测、Whisper、农业AI、bge-reranker（6 份）。

### M3. 模型选型多维权衡："能力 vs 显存 vs 兼容性 vs 价格"
- **一句话**：面对候选模型/方案，固定从显存需求、量化方式（FP16/FP8/int8/AWQ）、推理能力、框架兼容性、下载源可达性五个维度打分，给"采用/不采用"的明确结论，且主方案必须配一个备选。
- **证据**：
  - 《Whisper 部署文档》§1.2 选型表：fedirz/faster-whisper-server「不采用——与现有规范冲突」；openai-whisper「不采用——国内源极慢」；faster-whisper+自写 FastAPI「采用——int8 显存小、与 vLLM 同范式」；§1.3："large-v3-turbo FP16 需 6G、INT8 仅 3G；语音场景 int8 精度损失无感 → 选 int8"。
  - 《Qwen3-VL-8B 部署文档》§4.1："modelscope **没有** Qwen3-VL-8B-Instruct-AWQ（404），但有官方 FP8 版……**故采用 FP8**"。
  - 《农业AI部署文档》§1："vLLM 为主方案；若 vLLM 在 sm_120 无法启动，**降级到已验证的自写服务**。两者接口均 OpenAI 兼容，平台对接零改动"。
  - 《Qwen2.5VL7B 方案》清理表："Qwen2.5-VL-32B 20G 完整但 32B 超显存，**OOM，不适用**；27B 空目录"。
- **出现文档**：Whisper、Qwen3-VL-8B、农业AI、Qwen2.5-VL-7B、Qwen3VL8B 替换方案。

### M4. 服务编排：启动顺序 + systemd 依赖固化
- **一句话**：多服务共卡一张 GPU 时，识别每个服务的"启动硬性门槛"，大占用先起、小占用后起，并用 systemd After/Requires 把顺序写进配置，防重启后错序。
- **证据**：
  - 《启动与运维文档》§二："总原则：先启动大显存占用服务（8B），后启动小占用服务（whisper）。原因是 8B 配置了 --gpu-memory-utilization 0.57，vLLM 启动时会**硬性检查空闲显存 ≥ 18.2G**，不够就直接拒绝启动"；"已给 whisper 服务添加依赖 **After=vl-8b**，开机自启时 whisper 会自动排在 8B 之后"。
  - 《启动与运维文档》§五.1：8B 启动失败 → "停掉占用显存的其他服务 → nvidia-smi 确认空闲 ≥18.2G → 重启 8B → 8B 起来后再启动 whisper"。
  - 《农业AI部署文档》§0：安装顺序总表 A1→F1，每步带"校验"列。
- **出现文档**：启动运维、Qwen3-VL-8B、Whisper（After=vl-8b）、农业AI。

### M5. 最小暴露面："只开 Web + 业务接口，其余内网"
- **一句话**：防火墙默认 deny incoming，只为"人要进的 Web"和"业务框架要调的 API"开口；中间件端口（neo4j/langfuse/searxng/MinIO/redis）一律不对外，靠 Docker 内网通信，"收紧后内部功能无损、更安全"。
- **证据**：
  - 《WeKnora 部署文档》§2："防火墙对外放行（仅保留必需）：**8100**（后端 API，供 BladeX 框架调用）、**8181**（Web 前端）。其余 neo4j(7474/7687)、langfuse(3000/9100)、searxng(8888) **不对外放行**——容器间通过 Docker 内网通信，不经宿主防火墙"。
  - 《Qwen3-VL-8B 部署文档》§2："防火墙：仅放行 WeKnora 前端 8100/8181；**8002 属内网不放行**"。
  - 《Whisper 部署文档》§10："端口 8003 已按「**只放必要的接口**」口径放行（BladeX 调用需要）；其余端口不放"；§11.2 进一步建议收紧为指定网段。
  - 《Prometheus 文档》§3："放行 9300/tcp（Grafana 对外）、9090/tcp（Prometheus 管理页），**其余监控端口内网访问**"。
- **出现文档**：WeKnora、Qwen3-VL-8B、Whisper、Prometheus、农业AI（端口放行清单）。

### M6. 隔离部署："一个服务一个 venv/容器，坏了不连累别人"
- **一句话**：新服务必须独立 venv + systemd 或独立容器；连 CUDA 库、torch 都要复制一份到独立目录，确保旧环境（甚至 vllm-venv 坏了）删除/损坏后新服务完全不受影响；Docker 不动全局 runtime。
- **证据**：
  - 《Whisper 部署文档》§4.6："为保证 whisper 运行时**不依赖 vllm-venv**（vllm-venv 曾出现问题、仅留备份），将 whisper 需要的 CUDA 运行库**复制到 whisper-venv 独立目录**……复制后即使 vllm-venv 被删除/损坏，whisper 不受影响"；§10："whisper 运行时完全独立"。
  - 《douyin-distiller 文档》§2："现有 /opt 规范：**每个服务独立 venv + systemd**（whisper-venv、vllm-venv-new、h3-venv 等）"。
  - 《WeKnora 部署文档》§0："PG 用 WeKnora **自带 ParadeDB（容器内 5432，不碰宿主 5432）**；MinIO 为 WeKnora **单独建公共桶 weknora**（与现有隔离）"。
  - 《Prometheus 文档》§4.3："服务器 Docker 未配置 nvidia-container-toolkit……为不影响现有 15 个容器，采用**手动挂载 GPU 设备 + 驱动库的隔离方式**接入，**不改 Docker 全局配置**"。
- **出现文档**：Whisper、douyin-distiller、WeKnora、Prometheus、Qwen2.5-VL-7B（新建 vllm-venv-vl）。

### M7. 可观测与预警："装监控本身也要省资源、不干扰现有服务"
- **一句话**：GPU/内存/服务状态必须有监控和告警阈值，且监控栈自身"不占显存、仅占约 1G 内存"；告警分级（warning/critical），装好后"回看 OOM 历史无需再翻日志"。
- **证据**：
  - 《Prometheus 文档》§1："为 AI 服务器部署显存/内存/CPU/磁盘实时监控与预警，重点预防 **RAM OOM（此前服务器曾因内存压力影响连接）**"；§1："5 个开源组件，**不占显存、仅占约 1G 内存**，不干扰现有 vLLM / WeKnora 服务"。
  - §5 告警表："MemAvailable < 15G warning；< 10G **critical OOM高风险**；GPU 显存空闲 < 8% **critical**；温度 >85℃；磁盘 >85%"。
  - §9："监控装好后，内存历史(GB)面板可直接回看，**无需再翻日志**"。
- **出现文档**：Prometheus（主）、Whisper §9.2（探活接入现有 dcgm/node_exporter，不单独埋点）、压测报告建议 3。

### M8. 交付即验证："每步带校验命令，清单逐项打勾"
- **一句话**：不接受"装完了"，只接受"curl /health 返回 ok + 维度数对 + 端口监听 + nvidia-smi free 对"。每份文档自带"自检清单/验收步骤/通过标准"。
- **证据**：
  - 《Whisper 部署文档》§9.1 部署核对清单：7 项（is-active / is-enabled / ss 端口 / ufw ALLOW / /health / 显存增量 / 进程属主）。
  - 《农业AI部署文档》§10 部署后自检清单 9 条 + "验收通过标准"（向量化无 400、提问能命中、语音往返正常）。
  - 《douyin-distiller 文档》§9 联调验收 7 步、§12.5"2 条真实记录成功落库"。
  - 《Qwen3-VL-8B 部署文档》§7 功能验证清单（/v1/models、文本推理、base64 OCR、ss 端口、nvidia-smi）。
- **出现文档**：几乎全部（12 份中 10 份含验证清单）。

### M9. 根因排障："症状→根因→解决"三列表，先取证再判断
- **一句话**：任何报错都先 journalctl / nvidia-smi / ss 取证，定位是资源不足、配置错误、依赖版本过新、还是网络，并且把每个坑的根因写进文档"常见坑表"，防止二次踩。
- **证据**：
  - 《Whisper 部署文档》§7 排障表 5 行，如："500 av open() unexpected metadata_errors → **av 19.0.1 过新，faster-whisper 1.2.1 调已移除参数** → 改源码去掉 metadata_errors"；"500 Invalid features (1,80) vs (1,128) → 缺 preprocessor_config.json，默认 80 mel → 补 128 mel"。
  - 《农业AI部署文档》§9.4 根因："JDK HttpClient 默认发起 HTTP/2 明文升级（h2c），uvicorn/h11 不支持该握手返回 400"。
  - 《douyin-distiller 文档》§14.4："SSH 无 tty 时 sudo 静默失败→服务仍跑旧代码；**已用 PID 变化验证修复**"；"整文件覆盖上传 settings.json 会清空已配凭据 → 已改为服务器侧修补，**勿再整文件覆盖**"。
- **出现文档**：Whisper、农业AI、douyin-distiller、启动运维 §五、WeKnora §3 踩坑记录。

### M10. 接口兼容优先："OpenAI 兼容 = 业务框架零改造成本"
- **一句话**：自研服务一律对齐 OpenAI 兼容路径（/v1/embeddings、/v1/chat/completions、/v1/audio/transcriptions），因为下游 BladeX 框架是按 OpenAI 协议调的；部署前先想"目标框架怎么调、要不要鉴权、跨机端口放不放"。
- **证据**：
  - 《Whisper 部署文档》§1.1："「两者都要」：1. 集成 WeKnora；2. 独立 OpenAI 兼容接口供 BladeX 框架调用"。
  - 《农业AI部署文档》§7.1："两种服务方式加载同一模型，接口均 OpenAI 兼容，**平台对接零改动**"。
  - 《WeKnora 部署文档》§1.1：现有 vLLM 8080/8001 直接被 WeKnora 复用为 Embedding/Rerank。
  - 《bge-reranker 文档》§三：/v1/models、/v1/score 接口示例。
- **出现文档**：Whisper、农业AI、WeKnora、bge-reranker、douyin-distiller（asr_client 三通道）。

---

## 二、决策启发式候选（if-then，12 条）

| # | 启发式 | 案例证据 |
|---|---|---|
| H1 | **如果要换/升框架版本，先做"零风险代码层前置验证"，不支持就不升** | Qwen3-VL-8B：先用 `ModelRegistry.get_supported_archs()` 查 vllm 0.12 是否含 Qwen3VL——"验证不支持：按用户决策**不升级 vllm、沿用 7B**"；"前置验证先做，确认支持方式后再走下载，**避免白下载**"（替换方案 §三） |
| H2 | **如果显存 OOM/启动被拒，先停小占用服务腾空间，再起大模型** | 启动运维 §五.1："停 whisper → nvidia-smi 确认空闲 ≥18.2G → 重启 vl-8b → 8B 起来后再起 whisper" |
| H3 | **如果 KV cache 差一点点（0.09G 级），微调 gpu-memory-utilization 而不是换模型** | Qwen3-VL-8B §6.2：0.55 时 KV 0.47G vs 需 0.56G → 提到 0.57 → KV 1.09G，"文档验证过最优"，"保持现配置，无需调整"（压测 §6） |
| H4 | **如果国内源拉不到镜像/模型，不硬等，探测多个加速器换通道** | WeKnora §3.1："直连 Docker Hub 与 GitHub 不通；经探测，**docker.1panel.live 是唯一对 WeKnora 镜像返回 200 的加速器**"；daocloud"1 小时只到几十 MB" → 换 1panel 秒级完成。Whisper §4.2：HF 不通、snapshot_download 静默失败 → **curl 逐文件**从 ModelScope 下 |
| H5 | **如果要删模型/服务，先 mv 到 /data/backup 而非直接 rm，确认无误再删** | Qwen2.5VL7B 方案 §二："删除命令（执行前逐项确认，**误删可回滚：先 mv 到 /data/backup 而非直接 rm**）" |
| H6 | **如果替换旧模型，旧模型+旧 service 完整保留只 disable，保证秒回滚** | Qwen3-VL-8B 部署 §9："7B 模型与 vl-7b.service **完整保留**（未删除，仅 disable），可随时回滚"；替换方案 §九："8B 效果不如预期 → **秒回滚**" |
| H7 | **如果 venv 曾经坏过，新服务的 CUDA/torch 全部复制一份独立目录，不依赖它** | Whisper §4.6："vllm-venv 曾出现问题、仅留备份……复制后即使 vllm-venv 被删除/损坏，whisper 不受影响" |
| H8 | **如果服务"改了配置但没生效"，用可观测手段验证真生效，不信 systemctl 返回码** | douyin-distiller §14.4："SSH 无 tty 时 sudo 静默失败→服务仍跑旧代码；**已用 PID 变化验证修复**" |
| H9 | **如果模型下载中断，断点续传 + 校验无 .incomplete 残留才算完** | Qwen2.5VL7B §四："HF_ENDPOINT=hf-mirror --resume-download……完成后校验：目录应 ~8-9G，**无 .incomplete 残留**"；Qwen3-VL-8B §4.2 下载结果列"完整性：0 个 .incomplete" |
| H10 | **如果长上下文需求 vs 显存不足冲突，本地模型压短，长文走云端 API** | 启动运维 §六："不要随意增大 --max-model-len（如 8192）：当前 32G 上显存不足会 OOM。**知识库长文本问答请用 qwen-plus（阿里 128K context）**" |
| H11 | **如果新端口和现有端口撞了，避让现有端口，不图方便** | Prometheus §2："Grafana 常用 3000 已被 langfuse-web 占用 → 改 **9300**；node_exporter 常用 9100 已被 langfuse-minio 占用 → 改 **9102**" |
| H12 | **如果要动现有服务的配置文件，服务器侧修补，绝不整文件覆盖上传** | douyin-distiller §14.4："整文件覆盖上传 settings.json 会清空服务器已配的 cookie/飞书凭据 → 已改为服务器侧修补，**勿再整文件覆盖**" |

---

## 三、表达 DNA 观察

1. **指令风格：命令式短句 + 明确禁令，用"别/勿/不要/⚠️"开头**
   - "别下 0.6B"（农业AI §C1）；"**不要贸然重装**"（§4.1）；"**勿再整文件覆盖**"（douyin §14.4）；"**勿再升级/降级** av 版本"（Whisper §10）；"不要随意增大 --max-model-len"（启动运维 §六）。
   - 特征：每条禁令后紧跟后果（"可能丢 OEM 的 RAID/驱动配置"、"会清空已配凭据"、"显存不足会 OOM"）——不是空喊，是"禁令+代价"成对出现。

2. **下达任务前先锁状态："待你确认，确认前不执行任何动作"**
   - Qwen2.5VL7B 方案开头："状态：**方案待你确认，确认前不执行任何清理/下载/启动**"；替换方案同；结尾"你逐项确认后我再执行。执行顺序：前置验证 → 下载模型 → 停 7B → 起 8B → 验证"。
   - 他把"执行顺序"写成流水线清单交回来，本质是把指挥权留在自己手里——AI 可以出方案，但"按哪几步、什么顺序"由他点头。

3. **汇报格式：状态先行 + 已验证/待办两段切分**
   - WeKnora 文档开头："状态：**已安装完成并验证**。全部服务已启动、健康，核心 RAG 链路已验证"；§6 独立一节"**待你完成的操作**"（填 API key、配摘要模型、接 Langfuse、传真实文档）。
   - 特征：已完成的不啰嗦，未完成的列成 checklist 交回，且每条带具体路径（"编辑 /home/ubuntu/weknora/.env……重启 app"）。

4. **追问到小数位：对显存/KV 这类数字有"差 0.09G 也要记下来"的较真**
   - "max seq len 4096 needs 0.56 GiB KV, available 0.47 GiB —— **KV cache 差 0.09G**"（Qwen3-VL-8B §6.2）；
   - 压测报告把 free 精确到 MiB："used 26962 / free 5149"，并追问"free 5G 够不够用"。
   - 他不接受"大概够"，要求用数字证明"够"或"不够"。

5. **排障要求：先看日志/指标取证，再区分责任方**
   - 通用排障四件套固定出现：`journalctl -u <服务> -f`、`nvidia-smi --query-gpu=memory.free`、`ss -tlnp | grep <端口>`、`curl 127.0.0.1:port/health`。
   - 区分责任方的典型：Whisper 文档明确"人声被音乐盖过、多人叠声、方言时转写精度下降——这是所有语音转写模型的共性，**与模型本身无关**"（§5.3）；监控文档明确"告警仍会在 Prometheus 中记录，只是不推送"（区分邮件通道 vs 采集通道）。

6. **需求表达用"两者都要"锁边界**
   - Whisper §1.1："「**两者都要**」：1. 集成 WeKnora；2. 独立 OpenAI 兼容接口供 BladeX 调用"——先把需求边界钉死，不允许 AI 二选一简化。

---

## 四、价值观 / 反模式候选

### 看重的（价值观）
1. **可回滚**：旧模型/旧 service 不删只 disable（H6）；删文件先 mv 到 backup（H5）；坏 venv 留备份（M6）。
2. **隔离不污染**：一个服务一个 venv/桶/端口；"全程未改动现有服务，完全隔离"是最高频的验收话术（WeKnora §8、Qwen3-VL-8B §2）。
3. **可查证**：每个坑都进"排障表/常见坑表"，带症状→根因→命令；文档带路径、端口、版本号，供人/AI 续用。
4. **最小暴露**：端口只开 Web+业务 API，中间件一律内网（M5）。
5. **性价比/资源克制**：int8 代替 FP16（省 3G）、FP8 代替 AWQ（显存更省且 modelscope 有货）、监控栈"不占显存仅 1G 内存"、复用已部署的 whisper 不重复占显存（douyin §13）。
6. **验证闭环**：每步有校验命令、有通过标准、有实测数字（"2 条真实记录成功落库"、"8 秒音频 0.23s"）。

### 反对的（反模式黑名单）
1. **"瞎整"/"弄乱套"**：对环境被搞乱有明确历史创伤——"用户反馈现有多套 vLLM 环境曾'弄乱套'"，因此隔离是第一优先级。
2. **贸然重装/删库**："不要贸然重装（可能丢 OEM RAID/驱动）"；"直接 rm"被禁止。
3. **装了不算完**：反对"systemctl 返回 0 就算成功"，要求 PID 变化、/health、维度数、free 显存全部对得上。
4. **整文件覆盖配置**："勿再整文件覆盖"——会把线上已配凭据冲掉。
5. **占着显存不榨干也不放**："不必再压榨"——free 5G 是安全缓冲，反对为多装服务把余量压到危险线。
6. **照搬默认端口/默认参数**：3000、9100 撞了就避让；--max-model-len 默认 8192 会 OOM 就主动压到 4096。

---

## 五、矛盾与张力（3 对）

1. **「功能全家桶」vs「一块 32G 卡」**
   - WeKnora 一上就是 13 容器、neo4j 图谱、langfuse 观测、searxng 搜索；同时 4 个 vLLM 服务已占 28.46G。
   - 解法不是砍功能，而是**分层**：LLM 走云端阿里 API（不占本机）、监控不占显存、WeKnora 内部件不发布端口、长文本走云端 qwen-plus 128K。张力被"本地做检索/语音/OCR，生成式 LLM 上云"化解。

2. **「安全隔离」vs「接口必须打通」**
   - 要隔离（neo4j/langfuse/中间件不开端口、venv 互不依赖），又要通（BladeX 跨机调 8002/8003/8100、WeKnora 容器回调宿主 8080/8001/9000）。
   - 解法：**防火墙只开两个入口（Web 8181 + API 8100），容器与宿主之间用 host.docker.internal + SSRF 白名单打通**；"收紧后内部功能无损、更安全"——隔离和联通在他这里不是取舍，而是"端口层最小化、应用层白名单化"。

3. **「不碰现有环境」vs「必须升级换代」**
   - 铁律是"不碰 embedding(8080)、rerank(8001)、不碰宿主 PG"，但业务上又要求多模态从 7B 升 8B、模型从 AWQ 换 FP8。
   - 解法：**前置验证分支 A/B**（支持就复用、不支持就新建独立 venv）+ 旧服务只 disable 不删除；升级动作被限制在"8002 这一个端口、/opt/models 这一个目录"内闭环，回滚零损失。

---

## 六、关键决策案例（6 个）

### 案例 1：Whisper 启动顺序——After=vl-8b 的由来
- **背景**：8B 模型配 `--gpu-memory-utilization 0.57`，vLLM 启动瞬间硬性检查空闲显存 ≥18.2G；whisper 虽只占 1.5G，但若它先起来，就会挤掉这 18.2G 门槛。
- **考虑因素**：开机自启时 systemd 并行拉起，顺序不受控；重启机器后错序会导致 8B 反复 restart 循环。
- **选择**：给 whisper.service 加 `After=vl-8b`，把"大占用先起"写进依赖图；人工排障手册同步写成"停 whisper→腾显存→起 8B→再起 whisper"。
- **结果**：重启后自动按 8B→whisper 顺序拉起，"无需手动干预"（启动运维 §二）。

### 案例 2：Qwen3-VL-8B 替换 7B——先验证后下载、FP8 替代 AWQ
- **背景**：vllm 0.12 发布于 2025-08，Qwen3-VL-8B 发布于 2025-11，版本线是否支持未知；且原计划下 AWQ 4bit。
- **考虑因素**：①直接升级 vllm 有"弄乱套"风险；②白下 9.9G 模型再发现不支持是浪费；③modelscope 上 AWQ 404。
- **选择**：①先用 `ModelRegistry.get_supported_archs()` 零下载代码层验证——确认支持 Qwen3VLForConditionalGeneration，分支 A 命中，复用 vllm-venv-new 零升级；②下载源自动切到官方 FP8 版"性能接近 BF16，显存更省"；③保留 7B 仅 disable。
- **结果**：OCRBench 864→896、MathVista 68.2→~77，多占 1.8G 但 free 仍 5.1G，"替换收益大于显存成本"（压测 §5），7B 随时可秒回滚。

### 案例 3：显存总账 28.46G/32G 的核算与 3.6G 余量纪律
- **背景**：4 个服务（8B 13.78 + embedding 10.76 + rerank 2.4 + whisper 1.49）挤在一张 32G 卡上。
- **考虑因素**：每个服务峰值显存不同；KV cache 要预留；系统/驱动要占。
- **选择**：不拍脑袋，把每个服务实测占用列成总账，free 3.6G 即"封顶"，新服务想进来先看总账还够不够；压测专门验证"free 5G 够不够"，结论"够，不必再压榨"。
- **结果**：压测全程显存"used 26962 / free 5149 固定不变"，16 并发文本、8 并发 OCR 零失败——证明 vLLM 启动时预分配 KV 后并发不再申请新显存，总账模型成立。

### 案例 4：KV cache 0.09G 差值的微调剂
- **背景**：8B-FP8 比 7B 权重更大，gpu-memory-utilization 从 0.35 提到 0.55 后仍报 "max seq len 4096 needs 0.56 GiB KV, available 0.47 GiB"。
- **考虑因素**：差 0.09G——换模型不值得、降 max-model-len 会牺牲上下文、升 utilization 怕挤占其他服务。
- **选择**：0.55→0.57 微提一档，KV 可用从 0.47G 跳到 1.09G（超出需求且支持并发），free 仍 5.15G 健康；同时加 `--enforce-eager` 绕开 vllm 0.12 对 Qwen3-VL 的 torch.compile bug、`VLLM_MM_INPUT_CACHE_GIB=1` 把多模态缓存从 4G 压到 1G 反哺 KV。
- **结果**：三个问题（KV 负、tracer_output 编译崩溃、KV 差 0.09G）全部闭环，参数组合"文档验证过最优"。

### 案例 5：端口只开 8100 Web + 8100 API，其余全内网
- **背景**：WeKnora 13 个容器、Prometheus/Grafana/Alertmanager、neo4j、langfuse、searxng，端口一堆。
- **考虑因素**：浏览器要人进 8181；BladeX 框架要跨机调 8100/8003；neo4j/language/searxng/MinIO/redis 只是容器内部互调。
- **选择**：ufw 只放行 8100（API）+ 8181（Web）+ 9300/9090（监控看板）+ 8003（Whisper 给 BladeX）；8002/7474/7687/3000/9100/8888 一律不开；容器间靠 Docker 内网，宿主防火墙不见流量。
- **结果**："收紧后内部功能无损、更安全"（WeKnora §2）；8002 多模态 API 不对外，仅 WeKnora 容器经 host.docker.internal 白名单调用。

### 案例 6：GPU 监控从 textfile 脚本过渡到官方 dcgm-exporter
- **背景**：要监控 GPU 显存/温度/功耗，但国内源拉不到 nvidia/dcgm-exporter 镜像；且 Docker 没配 nvidia-container-toolkit，配全局 runtime 怕影响现有 15 个容器。
- **考虑因素**：①临时方案 node_exporter textfile + nvidia-smi cron 能出指标但不是官方；②配全局 runtime 有污染风险；③用户可提供镜像 tar。
- **选择**：先用 textfile 方案过渡跑通；等用户给镜像 tar 后 `docker load` 导入，改用官方 dcgm-exporter，**手动挂载 /dev/nvidia0 等设备 + libnvidia-ml.so.1**，不改 Docker 全局配置；旧 cron 任务停用。
- **结果**：dcgm 抓取 0.009s，识别 20 个 DCGM 官方指标（显存空闲/已用、温度、功耗、利用率），"效果等同，无需额外镜像"，现有 15 容器零影响。

---

## 附：语料覆盖核对
- 已读：思维蒸馏文档 1 份 + 素材 12 份（农业AI Ubuntu 27.5KB、WeKnora 12.4KB、Whisper 15.8KB、douyin-distiller 13.6KB、启动运维 4.8KB、Prometheus 10.6KB、bge-reranker 2.4KB、Qwen2.5-VL-7B 6.3KB、Qwen3-VL-8B 部署 8.4KB、压测 4.9KB、Qwen3-VL-8B 替换方案 6.5KB、WeKnora 带图MD手册 6.8KB）。
- 凭据脱敏：MinIO 账号密码、Grafana/WeKnora 管理员密码、SMTP/邮箱地址、Embedding api-key、飞书 app 凭据、SSH/sudo 凭据均未写入本文，统一以「[凭据已脱敏]」指代。
