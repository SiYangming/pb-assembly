# pb-assembly 使用示例：FALCON 基因组组装与 FALCON-Unzip 分型

本文以 PacBio 长读长数据为例，说明用 `pb-assembly` 套件中的 FALCON 完成基因组组装、再用 FALCON-Unzip 进行单倍型分型的完整流程。安装方法见 [INSTALL.md](INSTALL.md)。

## 一、流程概述

FALCON 是面向 PacBio 长读长（SMRT）的 *de novo* 组装工具，遵循 HGAP（hierarchical genome assembly process）：

1. **预组装（pre-assembly）**：选取最长的种子序列（seed reads，长度阈值由 `length_cutoff` 控制），把较短的 reads 比对到种子上并做一致性纠错，得到高准确度（通常 > 99%）的 pread；
2. **pread 重叠**：pread 之间相互比对；
3. **contig 组装**：根据重叠关系构建图并输出 contig。

对应的工作目录：

| 目录 | 内容 |
|------|------|
| `0-rawreads` | raw read 重叠与一致性，即预组装（pre-assembly） |
| `1-preads_ovl` | pread 重叠（pread overlapping） |
| `2-asm-falcon` | contig 组装 |

FALCON-Unzip 在 FALCON 结果之上完成分型与打磨，对应目录：

| 目录 | 内容 |
|------|------|
| `3-unzip` | read 比对、SNP calling、read phasing，输出 primary contigs 与 haplotigs |
| `4-polish` | 分型打磨（phased polishing，BLASR + Arrow） |

## 二、准备输入

FALCON 需要两份输入：

- `input.fofn`：输入文件列表，每行一个 fasta 文件路径（PacBio subreads）；
- `fc_run.cfg`：INI 格式的配置文件。

```bash
mkdir -p path/to/falcon_run
cd path/to/falcon_run

# 每个 fasta 路径一行，写入 input.fofn
ls *.fasta > input.fofn
cat input.fofn
```

## 三、fc_run.cfg 关键配置项

配置分为 `[General]`（流程参数）与 `[job.*]`（计算资源）两部分。

| 配置项 | 说明 |
|--------|------|
| `input_fofn` | 输入文件列表（fofn 文件名） |
| `input_type` | 输入数据类型，取值 `raw` 或 `preads`；填 `preads` 会跳过 `0-rawreads` 预组装阶段 |
| `pa_DBsplit_option` / `ovlp_DBsplit_option` | 数据分块参数 |
| `pa_HPCTANmask_option` / `pa_HPCREPmask_option` / `pa_REPmask_code` | 重复序列屏蔽参数 |
| `genome_size` | 估计的基因组大小 |
| `seed_coverage` | 种子序列覆盖度 |
| `length_cutoff` | 种子序列长度阈值（`-1` 表示自动选取） |
| `pa_daligner_option` | 种子序列的 overlap 搜索参数 |
| `pa_HPCdaligner_option` | 传给 `daligner` 子命令的资源参数 |
| `falcon_sense_option` | 预组装一致性（falcon_sense）参数 |
| `length_cutoff_pr` | 校正后（pread）序列长度阈值 |
| `ovlp_daligner_option` | pread 重叠的 overlap 搜索参数 |
| `overlap_filtering_setting` | 重叠过滤参数 |
| `fc_ovlp_to_graph_option` | 由图生成 contig 的参数 |
| `[job.defaults]` | `job_type`、`pwatcher_type`、`submit`，以及默认的 `NPROC`、`MB`、`njobs` |
| `[job.step.da]` `[job.step.pda]` `[job.step.la]` `[job.step.pla]` `[job.step.cns]` `[job.step.asm]` | 各步骤单独的资源覆盖 |

以下为单机（本地）运行示例，流程参数取自仓库 `cfgs/fc_run_200kb.cfg`，资源与提交方式为本地模式：

```bash
cat > fc_run.cfg <<'CFG'
[General]
input_fofn=input.fofn
input_type=raw

#### 数据分块
pa_DBsplit_option=-x500 -s200
ovlp_DBsplit_option=-x500 -s200

#### 重复序列屏蔽
pa_HPCTANmask_option=
pa_REPmask_code=0,300;0,300;0,300

#### 预组装
genome_size=0
seed_coverage=20
length_cutoff=1000
pa_HPCdaligner_option=-v -B128 -M24
pa_daligner_option=-e.8 -l2000 -k18 -h480 -w8 -s100
falcon_sense_option=--output-multi --min-idt 0.70 --min-cov 2 --max-n-read 1800
falcon_sense_greedy=False

#### pread 重叠
ovlp_daligner_option=-e.9 -l2500 -k24 -h1024 -w6 -s100
ovlp_HPCdaligner_option=-v -B128 -M24

#### 最终组装
overlap_filtering_setting=--max-diff 100 --max-cov 100 --min-cov 2
fc_ovlp_to_graph_option=
length_cutoff_pr=1000

[job.defaults]
job_type=local
pwatcher_type=blocking
submit = bash -C ${CMD} >| ${STDOUT_FILE} 2>| ${STDERR_FILE}
MB=32768
NPROC=4
njobs=2
[job.step.da]
[job.step.pda]
[job.step.la]
[job.step.pla]
[job.step.cns]
[job.step.asm]
CFG
```

按数据规模调整参数的参考：

- `genome_size`、`seed_coverage` 需按待组装基因组设置；大基因组可参考 `cfgs/fc_run_human.cfg`（`genome_size = 2800000000`、`seed_coverage = 40`）。
- `pa_daligner_option` 控制种子序列的 overlap 搜索，基因组越大、越复杂时可放宽 `-k`、`-w`、`-h`；例如 `cfgs/fc_run_human.cfg` 使用 `-k18 -e0.80 -l1000 -h256 -w8 -s100`。
- `length_cutoff = -1` 时由流程自动选取种子序列长度阈值；`cfgs/fc_run_200kb.cfg` 则显式设为 `1000`。

## 四、运行 FALCON

```bash
source activate /path/to/install/miniconda3_for_pb-assembly

fc_run.py fc_run.cfg &> FALCON.log

# 套件同时提供 fc_run 命令，等价写法
# fc_run fc_run.cfg
```

## 五、产出目录与结果文件

运行结束后，工作目录下会出现 `0-rawreads/`、`1-preads_ovl/`、`2-asm-falcon/` 三个子目录。`2-asm-falcon/` 中的主要结果文件：

| 文件 | 说明 |
|------|------|
| `p_ctg.fasta` | primary contigs，即主组装结果（README 中亦写作 `p_ctg.fa`） |
| `a_ctg.fasta` | associated contigs，代表与某个 primary contig 同源的离散等位变异序列 |
| `a_ctg_base.fasta` | 每个 a-contig 所对应的 p-contig 区间序列；每条 `a_ctg_base.fasta` 序列都是某条 primary contig 的连续子序列 |

统计组装指标：

```bash
python scripts/get_asm_stats.py 2-asm-falcon/p_ctg.fasta
```

输出为 JSON，字段包括 `asm_contigs`（contig 数）、`asm_total_bp`（总长）、`asm_n50`、`asm_max`、`asm_min`、`asm_esize` 等。通常 `asm_n50` 大于 1 Mb 说明连续性较好；`asm_total_bp` 是否符合预期基因组大小可用于判断组装完整度。

## 六、FALCON-Unzip 分型与打磨

FALCON-Unzip 会在 FALCON 组装结果上做变异检测、按单倍型给 reads 分组，并执行单倍型特异的再组装与打磨，最终输出 primary contigs 与 haplotigs。它在同一工作目录下运行，使用另一份配置文件 `fc_unzip.cfg`；若需要打磨，还需提供 subreads BAM 列表 `input_bam.fofn`。

```bash
fc_unzip.py fc_unzip.cfg &> run1.std &
```

`fc_unzip.cfg` 结构（参考仓库 `cfgs/fc_unzip.cfg`）：

```ini
[General]
max_n_open_files = 1000

[Unzip]
input_fofn=input.fofn
input_bam_fofn=input_bam.fofn
polish_include_zmw_all_subreads = true

[job.defaults]
# job_type / pwatcher_type / submit / NPROC / MB / njobs 的含义与 fc_run.cfg 一致

[job.step.unzip.track_reads]
[job.step.unzip.blasr_aln]
[job.step.unzip.phasing]
[job.step.unzip.hasm]
[job.step.unzip.quiver]
```

其中 `max_n_open_files` 用于限制 read tracking 阶段同时打开的 `.sam` 文件数，在文件系统延迟较高的环境下可适当调大；`input_fofn` 与 `fc_run.cfg` 中的设置保持一致。

HiFi（CCS）数据的分型：

```bash
fc_unzip.py --target='ccs' fc_unzip.cfg
```

Unzip 的产出：

| 路径 | 说明 |
|------|------|
| `3-unzip/all_p_ctg.fa` | primary contigs |
| `3-unzip/all_h_ctg.fa` | haplotigs，分型后的单倍型序列 |
| `3-unzip/all_h_ctg.paf` | haplotig placement，每个 haplotig 比对到 primary contig 的位置（PAF 格式），每个比对区间对应一个 phase block |
| `4-polish/cns-output/cns_p_ctg.fasta` | 打磨后的 primary contigs，即最终结果 |
| `4-polish/cns-output/cns_h_ctg.fasta` | 打磨后的 haplotigs，即最终结果 |

```bash
python scripts/get_asm_stats.py 3-unzip/all_p_ctg.fa
python scripts/get_asm_stats.py 3-unzip/all_h_ctg.fa
head 3-unzip/all_h_ctg.paf
```

通常 `3-unzip/all_h_ctg.fa` 的总长短于 `all_p_ctg.fa`，且更加碎片化；两者总长之比可用于估计基因组被成功分型的比例。

## 七、Release 附件 FALCON.fasta 的含义与用法

Release 附件 `FALCON.fasta` 是一份 **FALCON 组装结果示例**：对 *Malassezia sympodialis*（基因组约 8 Mb）的 PacBio subreads 运行 FALCON 后得到的 primary contigs。

- 文件共 **12 条序列**，序列名依次为 `falcon01` … `falcon12`；
- 总长约 **7,767,151 bp（约 7.77 Mb）**，最长序列 1,508,427 bp，最短 5,265 bp，N50 为 1,202,138 bp；
- 其内容对应 FALCON 工作目录中的 `2-asm-falcon/p_ctg.fasta`，仅对序列名做了统一前缀处理。

用法示例：

```bash
# 1. 查看序列条数
grep -c '^>' FALCON.fasta

# 2. 建立索引并查看每条序列的长度
samtools faidx FALCON.fasta
cut -f1,2 FALCON.fasta.fai

# 3. 使用套件自带脚本统计组装指标
python scripts/get_asm_stats.py FALCON.fasta

# 4. 导出单条序列
samtools faidx FALCON.fasta falcon01 > falcon01.fasta
```

该文件可作为组装结果格式与内容的参考：自行运行 FALCON 得到的 `2-asm-falcon/p_ctg.fasta` 与之结构一致，即可按同样的方式做统计与下游使用（比对、评估，或作为 FALCON-Unzip 的输入）。
