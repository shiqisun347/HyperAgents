<div align="center">

<!-- Logo/Banner placeholder - uncomment and add your image -->
<!-- <img src="assets/banner.png" alt="HyperAgents Banner" width="800"> -->

<h1>HyperAgents</h1>

<p>Self-referential self-improving agents that can optimize for any computable task</p>

<p>
<a href="LICENSE.md"><img src="https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg?style=for-the-badge" alt="License: CC BY-NC-SA 4.0"></a>
<a href="https://arxiv.org/abs/2603.19461"><img src="https://img.shields.io/badge/arXiv-2603.19461-b31b1b.svg?style=for-the-badge&logo=arxiv" alt="arXiv"></a>
<a href="https://ai.meta.com/research/publications/hyperagents/"><img src="https://img.shields.io/badge/-Blog-%238D6748?style=for-the-badge&logo=Website&logoColor=white"></a>
<a href="https://x.com/jennyzhangzt/status/2036099935083618487"><img src="https://img.shields.io/badge/twitter-%230077B5.svg?&style=for-the-badge&logo=twitter&logoColor=white&color=00acee"></a>
</p>

---

</div>

## Setup
```bash
# API keys, put these into .env file
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
GEMINI_API_KEY=...
```

```bash
# Install things
sudo dnf install -y python3.12-devel
sudo dnf install -y graphviz graphviz-devel cmake ninja-build bzip2-devel zlib-devel ncurses-devel libffi-devel
```

```bash
# Create virtual environment
python3.12 -m venv venv_nat
source venv_nat/bin/activate
pip install -r requirements.txt
pip install -r requirements_dev.txt
# To build the docker container
docker build --network=host -t hyperagents .
```

```bash
# Setup initial agents
bash ./setup_initial.sh
```

## Running HyperAgents

```bash
# See the script for args, and baseline selections
python generate_loop.py --domains <domain>
```

By default, outputs will be saved in `outputs/` directory.

## 项目运行逻辑与示例

下面以中文简述 HyperAgents 的核心运行流程，并给出一个最简单的单任务示例（`paper_review` 域）：

- **总体流程（`generate_loop.py`）**  
  1. 初始化：`setup_initial_gen` 会把 `outputs/initial_*` 里的基线评测复制到新的输出目录（`outputs/generate_<run_id>/gen_initial`），并创建 `archive.jsonl` 记录世代节点。  
  2. 每一代（`generate`）：在 Docker 中复制仓库，应用上一代的补丁，然后调用 `run_meta_agent.py` 生成新的补丁，输出到 `gen_<id>/agent_output/model_patch.diff` 和聊天记录。  
  3. 评测：如果模型补丁非空，就用 `domains/harness.py`（或 `domains/polyglot/harness.py`）运行 `task_agent.py`，把预测与评分写入 `gen_<id>/<domain>_eval/` 下的 `predictions.csv`、`report.json`，并更新 `gen_<id>/metadata.json`。  
  4. 选父代：`select_parent` 根据得分（默认 `score_child_prop`）从 `archive.jsonl` 里选择下一代的父节点，继续迭代直到达到 `max_generation`。

- **最简示例：在 `paper_review` 域跑一代**  
  1. 准备：安装依赖并执行 `bash setup_initial.sh`（脚本默认只启用 `paper_review`，会在 `outputs/initial_paper_review_*` 下生成基线结果）。确保 Docker 可用并已写入所需 API Key 到 `.env`。  
  2. 运行：  
     ```bash
     python generate_loop.py --domains paper_review --max_generation 1 --eval_samples 10 --eval_workers 2 --run_id demo_paper
     ```  
  3. 结果查看（位于 `outputs/generate_demo_paper/`）：  
     - `gen_initial/`：拷贝的基线评测。  
     - `gen_1/agent_output/`：`model_patch.diff` 与 `meta_agent_chat_history.md`。  
     - `gen_1/paper_review_eval/`：模型在 `paper_review` 数据集上的 `predictions.csv` 与 `report.json`。  
     - `gen_1/metadata.json`：记录本代的父节点、补丁列表、是否完成全量评测等元数据。  
     - `archive.jsonl`：跨世代的节点与得分轨迹。

## File Structure
- `agent/` code for using foundation models
- `analysis/` scripts used for plotting and analysis
- `domains/` code for each domain
- `utils/` common code used in the repo
- `run_meta_agent.py` script to help run the meta agent and get the diffs
- `meta_agent.py` main implementation of the meta agent
- `task_agent.py` main implementation of the task agent
- `generate_loop.py` entry point for running the algorithm

## Logs from Experiments

The experiment logs are stored as a multi-part ZIP archive. To extract them, ensure all .z01, .z02, etc., files are in the same directory as the .zip file, then run:
```bash
zip -s 0 outputs_os_parts.zip --out unsplit_logs.zip
unzip unsplit_outputs.zip
```

## Safety Consideration
> [!WARNING]  
> This repository involves executing untrusted, model-generated code. We strongly advise users to be aware of the associated safety risks. While it is highly unlikely that such code will perform overtly malicious actions under our current settings and with the models we use, it may still behave destructively due to limitations in model capability or alignment. By using this repository, you acknowledge and accept these risks.

## Citing
If you find this project useful, please consider citing:
```bibtex
@misc{zhang2026hyperagents,
      title={Hyperagents}, 
      author={Jenny Zhang and Bingchen Zhao and Wannan Yang and Jakob Foerster and Jeff Clune and Minqi Jiang and Sam Devlin and Tatiana Shavrina},
      year={2026},
      eprint={2603.19461},
      archivePrefix={arXiv},
      primaryClass={cs.AI},
      url={https://arxiv.org/abs/2603.19461}, 
}
```
