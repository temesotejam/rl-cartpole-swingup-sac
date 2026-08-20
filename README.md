# rl-cartpole-swingup-sac

[![CI](https://github.com/temesotejam/rl-cartpole-swingup-sac/actions/workflows/ci.yml/badge.svg)](https://github.com/temesotejam/rl-cartpole-swingup-sac/actions/workflows/ci.yml)

**台車型倒立振子を真下から振り上げ、倒立捕捉し、最後に中央付近で安定化する課題を Soft Actor-Critic (SAC) で学習する実験リポジトリです。**

このリポジトリは [`rl-cartpole-swingup-ppo`](https://github.com/temesotejam/rl-cartpole-swingup-ppo) の比較対象です。物理環境、reward、センサノイズ、カリキュラム、評価条件、学習step数をできるだけ同一にして、**学習アルゴリズムだけ PPO → SAC に変更**します。

## 何を比較するのか

PPO版では400k stepで、真下からのcapture 100%、最終安定化91.7%まで到達しました。SACでは次を比較します。

- 何stepで初めてSwing-upが成功するか
- 何stepでcapture rate 100%へ到達するか
- 最終安定化率
- captureまでの時間
- ±10°以内の滞在率
- 台車の中央からのずれ
- RMSモータ力
- 同じ環境step数に対する学習効率

## SACとは

SACは連続行動向けのoff-policy Actor-Criticです。PPOが収集したrolloutを更新後に基本的に使い切るのに対して、SACは経験を**Replay Buffer**へ保存し、過去の遷移を繰り返し再利用します。

```text
Cart-Pole physics
       │ transition
       ▼
 Replay Buffer ──────┐
       │             │
       ├─ sample ─→ Q networks
       │             │
       └──────────→ stochastic actor
                         │
                         ▼
                    continuous force
```

SACは期待returnだけでなく方策のentropyも考慮し、探索性を保ちながら学習します。本実験ではStable-Baselines3の`ent_coef=auto`を使い、entropy係数を自動調整します。

## 物理環境

PPO版と同一です。

| 項目 | 値 |
|---|---:|
| 台車質量 | 1.0 kg |
| 振り子質量 | 0.1 kg |
| 振り子重心長 | 0.5 m |
| 重力 | 9.81 m/s² |
| 最大モータ力 | ±10 N |
| 制御周期 | 20 ms / 50 Hz |
| レール範囲 | ±2.4 m |
| モータ時定数 | 50 ms |
| 1 episode | 最大20 s |

状態は`x, x_dot, theta, theta_dot`、PPO/SACへ渡す観測は`x, x_dot, cos(theta), sin(theta), theta_dot`です。

### 大角度でも終了しない

Swing-upなので振り子が真下や横向きでもepisodeは継続します。早期終了は**台車が±2.4 mを超えた場合だけ**です。

## センサノイズ

AIはシミュレータ真値を見ません。PPO版と同じ民生用センサ相当のノイズを使います。

| 観測 | 白色ノイズσ | episode bias σ |
|---|---:|---:|
| 台車位置 | 1 mm | 2 mm |
| 台車速度 | 0.01 m/s | 0.01 m/s |
| 振り子角度 | 0.25° | 1.0° |
| 角速度 | 0.10°/s | 0.30°/s |

真値は評価指標の計算にだけ使います。

## Reward

rewardもPPO版の修正版と同一です。真下で静止していても通常step rewardが大きな負値にならないようにし、意図的にレール端へ衝突してepisodeを早期終了する抜け道を防いでいます。

主な要素は、

- 上向きに近いほど高reward
- 倒立時に台車中央へ近いほど追加reward
- 真下付近では適度な角速度を持つことを少し評価
- 大きな台車速度・角速度・モータ力を小さく罰する
- レール端に近づくと強く罰する
- ±10°、さらに±8°で低角速度・中央付近ならbonus

です。

## カリキュラム

SACにもPPOと同じ4段階を使います。

```text
0–25%      near_upright
25–50%     wide
50–75%     full
75–100%    downward_mix
```

評価だけは最初から最後まで`evaluation_downward`、つまりほぼ真下スタート固定です。

**SAC特有の点:** Replay Bufferはカリキュラム段階をまたいで保持します。物理法則は同一なので、以前の初期状態分布で得た経験を捨てずに再利用します。

## 学習量

| preset | steps | 用途 |
|---|---:|---|
| quick | 20,000 | CI / 動作確認 |
| normal | 400,000 | PPOとの主要比較 |
| long | 800,000 | 追加学習 |

`normal`をPPO版と同じ400kにしているのは、**同じ環境step数でどちらが早く学ぶか**を見るためです。

## SAC主要設定

normalでは初期値として、

```text
learning_rate = 3e-4
buffer_size = 500,000
learning_starts = 5,000
batch_size = 256
gamma = 0.99
tau = 0.005
train_freq = 1
gradient_steps = 1
ent_coef = auto
network = [256, 256]
```

を使います。

## 評価指標

### Capture

振り子が、

- ±12°以内
- 低角速度
- 0.5秒以上連続

を満たしたらcapture成功です。

### Final stable

episode最後の2秒の80%以上で、

- ±10°以内
- 台車±0.50 m以内

なら最終安定化成功です。

ほかにReturn、capture時間、±10°滞在率、RMS角度、RMS台車位置、RMSモータ力を記録します。

## 生成物

```text
results/
├── videos/
│   ├── 00_random.mp4
│   ├── 01_25_percent.mp4
│   ├── 02_50_percent.mp4
│   ├── 03_75_percent.mp4
│   └── 04_100_percent.mp4
├── models/
│   ├── 25_percent.zip
│   ├── 50_percent.zip
│   ├── 75_percent.zip
│   └── 100_percent.zip
├── plots/
├── metrics.csv
├── metadata.json
└── summary.md
```

動画はすべて同じ評価seed・ほぼ真下スタートです。

## GitHub Actions

`Actions → Train RL Agent → Run workflow`から`quick / normal / long`を選べます。`main`に初回トリガーファイルが入ると、normal / seed 42が一度自動実行されます。

Public repositoryのGitHub-hosted CPU runnerだけを使用します。GPUは不要です。

## GitHub Pages

成功した最新学習結果は、

`https://temesotejam.github.io/rl-cartpole-swingup-sac/`

へ公開するworkflowを用意しています。5段階動画、評価表、学習曲線、capture rate、最終安定化率、台車位置、モータ力をブラウザで確認できます。

新規リポジトリでは最初の1回だけSettings → Pages → Sourceを`GitHub Actions`へ変更する必要がある場合があります。

## PPOとの公平性

比較で固定しているもの:

- 物理パラメータ
- reward式
- センサノイズ
- モータ一次遅れ
- 50 Hz制御周期
- カリキュラム分布
- 真下評価条件
- capture/final-stable定義
- 評価seed生成規則
- 400k normal学習量

異なるもの:

- 学習アルゴリズム
- PPO固有/ SAC固有ハイパーパラメータ
- SACはReplay Bufferを使用
- SACはentropy係数を自動調整

これにより、最終的に**PPO vs SACの学習速度・成功率・制御入力の使い方**を比較できます。
