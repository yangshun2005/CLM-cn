# 微调

这是一个让 LLM 自主改进 CLM 微调效果的实验。

## 环境搭建

与用户协作完成以下事项：

1. 确定使用 `clm` 还是 `choice`、一个运行标签，以及固定的训练和评估命令。
2. 创建一个新的 `autofinetune/<tag>` 分支。
3. 阅读 `README.md`、`train/finetune.py`、`train/adapters.py`、`train/embed_utils.py`、`preprocessing/hf_embeddings.py` 和 `evaluation/bon_eval.py`。
4. 验证数据集、检查点、折数和 GPU 是否可用。
5. 创建一个未跟踪的 `results.tsv`，内容为：

   ```text
   commit	score	status	description
   ```

6. 与用户确认环境搭建无误，然后运行未修改的基线。

## 实验

使用约定的命令运行训练并重定向输出：

```bash
python train/finetune.py ... --out-dir runs/<tag>/<commit> > run.log 2>&1
```

对于 `clm`，随后运行约定的留出评估：

```bash
python evaluation/bon_eval.py ... --n <N> --window <K> >> run.log 2>&1
```

**你可以：**

- 修改 `train/finetune.py`。
- 更改头部、目标函数、优化器、调度器、批处理和超参数。

**你不可以：**

- 修改任何其他文件。
- 更改数据、嵌入、划分、折数、评估集、`--n` 或 `--window`。
- 安装依赖。
- 在评估数据上训练，或在 `choice` 测试集上调参。

目标是最大化 `clm` 的留出 best-of-N 率，或 `choice` 的验证准确率。当分数相同时，优先选择更简单的改动。

首次运行必须始终建立未修改的基线。

## 记录结果

每次运行后追加一行制表符分隔的记录：

```text
commit	score	status	description
a1b2c3d	0.796500	keep	baseline
b2c3d4e	0.805300	keep	task-blocked batches
c3d4e5f	0.787600	discard	constant learning rate
d4e5f6a	0.000000	crash	double width (OOM)
```

不要提交 `results.tsv`、`run.log` 或运行输出。

## 实验循环

无限循环：

1. 检查当前提交和之前的结果。
2. 对 `train/finetune.py` 做一处有针对性的修改。
3. 提交该修改。
4. 运行训练和评估。
5. 记录结果。
6. 如果分数提高，保留该提交。
7. 否则，重置到上一个保留的提交。
8. 继续尝试下一个想法。

如果运行崩溃，检查 `tail -n 80 run.log`。修复简单的 bug 并重新运行；否则记录崩溃并重置。如果超过约定的超时时间，停止运行，记录超时，然后重置。

一旦循环开始，不要询问是否继续。持续实验，直到用户打断你。