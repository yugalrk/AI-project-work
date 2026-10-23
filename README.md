# FR3 Duo handover: training and evaluation scripts

Siemens Healthineers Hackathon 2026, challenge C2. Scripts to train imitation-learning policies (ACT, SmolVLA)
on the FR3 Duo demonstrations, label the data with sub-tasks, and evaluate policies offline and closed-loop in
Isaac Sim.

**Models and data are not in this repository.** Put them next to the scripts (paths below) before running.

## Expected layout
```
Datasets/fr3_duo_handover_84_ep/fr3_duo_handover_v9_night_84/   handover dataset (LeRobot v3, 84 episodes)
Datasets/fr3_duo_rand500/fr3_duo_rand500/                       cube-stacking dataset (490 episodes)
Environment/.../Hackathon_FR3Duo_flattened.usd                  Isaac Sim scene
outputs/<run>/step_XXXXXX/                                      trained checkpoints (written by training)
```

## Scripts

| Stage | Script | What it does |
|---|---|---|
| Data | `fr3_data.py` | Dataset paths, train/val split (`splits.json`: 10 held-out episodes with extreme/random lighting), loader |
| Data | `label_rand500_colors.py` | Labels each cube episode with cube colours + gripper event frames → `rand500_labels.csv` |
| Data | `make_subtasks.py` | Per-frame sub-task labels from robot signals (LeRobot-native `subtask_index`), quality checks, review images |
| Train | `train_act_mixed.py` | ACT or SmolVLA (`--policy act/smolvla`) on handover and/or cubes (`--data handover/both`), 6 Hz, own norm stats |
| Train | `train_act.py` | Plain `lerobot-train` wrapper for ACT on the handover data (30 fps), with Windows fixes |
| Serve | `serve_policy.py` | Loads any LeRobot checkpoint and answers observations with joint targets over TCP; `--selftest` |
| Eval | `eval_offline.py` | Held-out episodes without a simulator: error per sub-task, horizon, gripper timing, DTW, smoothness, latency, robustness |
| Eval | `export_eval_assets.py` | Writes `eval_assets/handover_reference.json` (start pose, demo for replay, held-out lighting) |
| Eval | `eval_isaac.py` | Runs inside Isaac Sim: replay check of the demo, then closed-loop policy episodes with stage scoring |
| Misc | `main.py` | Quick look at the raw handover data |

Details on the simulator evaluation and the scene's gaps: [EVAL_README.md](EVAL_README.md).

## Setup (training / policy server side)
```bash
python -m venv .venv && source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install torch==2.10.0 torchvision==0.25.0 --index-url https://download.pytorch.org/whl/cu128
pip install -r requirements.txt
```
Isaac Sim's own Python needs nothing extra for `eval_isaac.py` (numpy + Pillow are bundled).

## Typical workflow
```bash
# train (each run ~25 min on an 8 GB laptop GPU)
python train_act_mixed.py --policy act     --data handover --steps 6000 --out outputs/act_handover
python train_act_mixed.py --policy smolvla --data handover --steps 3000 --batch 8 --out outputs/smolvla_handover

# offline evaluation on the held-out episodes -> eval_results/offline/summary.md
python eval_offline.py --models act_handover/step_006000 smolvla_handover/step_003000

# closed loop in Isaac Sim (server and simulator can be on different machines)
python export_eval_assets.py
python serve_policy.py --checkpoint outputs/smolvla_handover/step_003000 --port 5555
<isaac>/python.sh eval_isaac.py --mode replay --usd <scene.usd> --out eval_results/replay
<isaac>/python.sh eval_isaac.py --mode policy --usd <scene.usd> --server <server-ip>:5555 --episodes 10 --out eval_results/smolvla
```

## Notes
- Policies run at 6 Hz with 20-action chunks (3.3 s); the 30 fps handover data is sampled every 5th frame.
- SmolVLA needs the exact training instruction at inference (`eval_isaac.py` sends it).
- `eval_isaac.py` has not been run on a real Isaac Sim install yet; expect small API fixes on first run.
