# T-Rex runner：CLM 对比 Jev

同一个已部署的 head，保持不变，在 Chrome 离线恐龙游戏中与 TypeSafe 托管的 Jev（`jev-latest`）实时对战。CLM 的 `POST /v1/systemone` 使用 TypeSafe 的通信格式，因此两个玩家通过同一个客户端（[`examples/common.py`](../common.py)，基于 `clm.CLMClient` 构建）发送相同的请求；只有 base URL 和 API key 不同。

**任务。** 让恐龙存活 60 秒。[`trex/engine.py`](trex/engine.py) 是 Chromium 恐龙游戏的确定性 Python 克隆（相同的常量、跳跃物理、碰撞箱、障碍规则和速度曲线；来自 [laya-vs-jev](https://github.com/virajbhartiya/laya-vs-jev)，Apache-2.0），以固定的 60 FPS 步长实时推进。物理规划器会根据它观察到的该玩家的回答延迟，为模型答案落地的那一刻将每个动作（`jump`、`duck`、`run`）标记为安全或不安全，并指出时序余量最佳的那个动作。模型读取这些标签；当答案落地时，执行其概率最高的动作。多个请求同时进行，相隔几帧发出，因此玩家每隔几帧就能获得一次行动机会，而不是每次往返才行动一次。五个固定种子的赛道（原版游戏的障碍规则），每条 60 秒；碰撞后 1.5 秒重启赛道。"存活" = 窗口期内零死亡。测试框架的护盾处于开启状态，与上游发布时一致：规划器标记为不安全的答案会被替换为模型概率最高的安全动作，并且紧急检查可以在碰撞发生前采取行动。报告会统计每一次此类干预，因此存活数字衡量的是整个系统，而一致性和干预行衡量的才是模型。

**请求。** 每个决策一个 Choice，两个模型完全相同：

```
state:    Dino runner game. 2 large cacti ahead, 96 px away.
question: Choose the best safe action for the dinosaur.
  jump: Safe. Clears the 2 large cacti. Best.
  duck: Unsafe. Hits the 2 large cacti. Collision.
  run:  Unsafe. Hits the 2 large cacti. Collision.
```

CLM 于 2026-09-23 运行，使用 `clm-latest` = `CLM_v0.1-8B.pt`（Qwen3-8B 编码器，单张 RTX 4090）；Jev 于 2026-09-22 运行，使用 `jev-latest`（应答为 `jev-1.13.0`）。延迟为客户端侧每次请求的延迟。

**复现。**

```bash
pip install -r requirements.txt
# from the repo root, with clm-serve running (see Quickstart in the main README); for Jev put TYPESAFE_API_KEY=... in <repo>/.env
python examples/t_rex/run.py --model clm                 # 5 seeds x 60 s, real time, shield on
python examples/t_rex/run.py --model jev
python examples/t_rex/run.py --model clm --no-shield     # the model's answer stands
python examples/t_rex/run.py --model clm --lockstep 6    # game freezes while the model answers
```

`CLM_BASE_URL`（默认 `http://127.0.0.1:8700`）、`CLM_API_KEY`、`TYPESAFE_BASE_URL` 和 `TYPESAFE_MODEL` 可覆盖端点。每个玩家在自己的进程（`trex/brain.py`）中运行，因此游戏循环无法抢占它的时间。`results/<model>_realtime.json` 保存每个种子的报告（死亡次数、得分、决策数、回答延迟、与规划器最佳动作的一致性、被丢弃的迟到答案、错误）。实时运行取决于主机：报告中的 `host_stall_seconds_dropped` 表示游戏不得不跳过的实际时间。`--no-shield` 和 `--lockstep` 是你自己实验用的开关，不属于表格的一部分。