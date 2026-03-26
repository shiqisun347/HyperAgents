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

## 运行逻辑（含简单示例）
下面以最小化的 `paper_review` 任务为例，说明整体流程如何运行、存储结果与循环自我改进：

1) **准备初始数据与基线**  
   运行 `bash setup_initial.sh`（默认只开启 `paper_review`），会生成 `outputs/initial_paper_review_filtered_100_{train,val,test}_0/`，作为首代评测基线。

2) **启动生成循环**  
   例如只跑 1 代小样本测试：  
   ```bash
   python generate_loop.py \
     --domains paper_review \
     --max_generation 1 \
     --eval_samples 10 \
     --eval_workers 2 \
     --parent_selection latest
   ```  
   关键行为：  
   - `setup_initial_gen` 会把当前仓库和初始评测结果复制到 `outputs/generate_<run_id>/gen_initial/`，作为可编辑的工作副本。  
   - 每一代在 Docker 容器内运行，先应用父代的补丁，再调用 **MetaAgent**（见 `run_meta_agent.py`）让模型修改代码，产出 `agent_output/model_patch.diff`。  
   - 若补丁非空则进入评测，调用 `domains/harness.py` 运行 **TaskAgent**（`task_agent.py`）在 `paper_review` 数据集上生成预测，并由 `domains.report` 计算得分。评测产物保存在对应 `gen_<id>/paper_review_eval*`。

3) **记录与选择下一父节点**  
   - 当前代元数据存于 `outputs/generate_<run_id>/gen_<id>/metadata.json`，包含补丁、是否评测成功等标记。  
   - `archive.jsonl` 记录所有代的得分，用于 `select_parent` 选择下一轮的父代（示例里 `latest` 即总是选最近一代）。  
   - 如启用 `ensemble` 选项，会对历史代进行集成评估（`get_ensemble_scores_container`）。

4) **查看结果**  
   - 代码改动：`gen_<id>/agent_output/model_patch.diff`。  
   - 交互日志：`gen_<id>/agent_output/meta_agent_chat_history.md`。  
   - 预测与评分：`gen_<id>/paper_review_eval*/predictions.csv` 与 `report.json`。  
   - 进度图：自动生成在 `gen_<id>/` 内（如 `progress_paper_review_agent_train.png`），用于可视化表现。

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
