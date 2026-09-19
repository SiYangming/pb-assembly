# pb-assembly 安装说明

`pb-assembly` 是 PacBio 官方维护的组装工具套件（PacBio Assembly Tool Suite），覆盖：

- **FALCON**：基于 overlap-layout-consensus（OLC）的 PacBio 长读长 *de novo* 基因组组装流程；
- **FALCON-Unzip**：在 FALCON 组装结果基础上进行单倍型分型，并用 Arrow 做分型打磨（phased polishing）；
- **FALCON-Phase**：借助 HiC 数据在 unzip 的分型区块之间延伸 phasing（需要 HiC 数据）。

套件以 bioconda meta-package 形式发布，安装后会得到 `fc_run.py`（FALCON 主程序）、`fc_unzip.py`（FALCON-Unzip）、`fc_phase.py`（FALCON-Phase）等命令。FALCON 与 FALCON-Unzip 均以 INI 配置文件作为唯一输入参数。

安装方式二选一：

- **方式一**：使用本仓库 Release 附件中预置的 conda 环境包，解压即用、可离线运行；
- **方式二**：使用 bioconda 在线安装。

---

## 方式一：使用预置 conda 环境包

Release 附件 `miniconda3_for_pb-assembly.tar.gz` 是一个已经安装好 `pb-assembly` 的 Miniconda3 环境（基于 Python 3.7），解压后即可离线使用。

```bash
# 1. 下载 Release 附件 miniconda3_for_pb-assembly.tar.gz 后，解压到安装目录
mkdir -p /path/to/install/
tar zxf miniconda3_for_pb-assembly.tar.gz -C /path/to/install/
```

```bash
# 2. 将环境加入 PATH
#    可将下面一行追加到 ~/.bashrc.pacbio，方便后续重复使用
echo 'export PATH=/path/to/install/miniconda3_for_pb-assembly/bin:$PATH' >> ~/.bashrc.pacbio
source ~/.bashrc.pacbio
#    也可以只在当前会话临时生效
export PATH=/path/to/install/miniconda3_for_pb-assembly/bin:$PATH
```

```bash
# 3. 激活环境
source activate /path/to/install/miniconda3_for_pb-assembly

# 4. 验证安装
which fc_run.py
fc_run.py --help
```

## 方式二：bioconda 安装

```bash
# 1. 添加 conda 频道
conda config --add channels default
conda config --add channels conda-forge
conda config --add channels bioconda

# 2. 安装 pb-assembly
conda install pb-assembly
```

没有管理员权限时，可以安装到独立环境中：

```bash
conda create -n pb-assembly
source activate pb-assembly
conda install pb-assembly
```

更新套件：

```bash
conda update --all
```

HiFi（CCS）数据的组装同样由该套件支持，示例配置见仓库 `cfgs/fc_run_HiFi.cfg`、`cfgs/fc_unzip_HiFi.cfg`，用法见 [Example.md](Example.md)。

## 依赖说明

`pb-assembly` 是一个 meta-package，本身即涵盖运行 FALCON 全流程所需的代码与依赖。安装包中的 recipe 包括：

- `pb-falcon`：FALCON / FALCON-Unzip 本体（`fc_run.py`、`fc_unzip.py`）；
- `pb-dazzler`：dazzler 系列比对工具，FALCON 的子读长重叠检测依赖其中的 `daligner`；
- `genomicconsensus`：Arrow / Quiver 一致性打磨工具；
- 以及其他全部依赖。

FALCON 流程的调度由 pypeFLOW 引擎驱动，其配置文件即 pypeFLOW 的配置格式。使用 bioconda 或预置环境包安装时，上述依赖均已解析就绪，无需手工编译。

仓库内附带的资源：

| 路径 | 说明 |
|------|------|
| `cfgs/` | 多种示例配置：`fc_run_200kb.cfg`、`fc_run_human.cfg`、`fc_unzip.cfg`、`fc_phase.cfg`，以及 HiFi 版配置 |
| `tests/` | 内置测试配置（ecoli、yeast 的 `fc_run.cfg` 与 `input.fofn`） |
| `scripts/get_asm_stats.py` | 统计组装结果（contig 数、N50、总长等）的脚本 |
| `snake/` | 基于 Snakemake 的 FALCON 流程定义（CCS / CLR） |

## 环境变量

| 变量 | 说明 |
|------|------|
| `PATH` | 需要包含 conda 环境的 `bin` 目录（如 `/path/to/install/miniconda3_for_pb-assembly/bin`），否则找不到 `fc_run.py`、`fc_unzip.py` |

FALCON 不通过命令行参数接收线程数；线程、内存等资源在配置文件的 `[job.defaults]` 与 `[job.step.*]` 小节中设置（`NPROC`、`MB`、`njobs`），详见 [Example.md](Example.md)。

## 下一步

安装完成后，请参考 [Example.md](Example.md) 运行 FALCON 基因组组装与 FALCON-Unzip 分型，其中也说明了 Release 附件 `FALCON.fasta` 的含义与用法。
