# Cart-Pole Swing-up SAC training result

- Preset: `normal`
- Seed: `42`
- Best checkpoint: `100_percent`
- Best final-stable rate: `75.0%`

## Downward-start evaluation

| Stage | Timesteps | Return | Capture | Time to capture | Final stable | Upright ±10° | RMS cart x | RMS force |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| random | 0 | 9.4 | 0.0% | - | 0.0% | 0.0% | 1.062m | 2.58N |
| 25_percent | 100,000 | 1194.1 | 100.0% | 3.17s | 8.3% | 83.0% | 0.691m | 2.25N |
| 50_percent | 200,000 | 1414.0 | 100.0% | 3.10s | 16.7% | 85.1% | 0.525m | 2.33N |
| 75_percent | 300,000 | 1512.0 | 100.0% | 3.22s | 75.0% | 86.2% | 0.407m | 2.39N |
| 100_percent | 400,000 | 1558.1 | 100.0% | 3.16s | 75.0% | 86.5% | 0.366m | 2.43N |

Capture means the pole stayed within ±12° with low angular velocity for at least 0.5 s.
Final stable means at least 80% of the final 2 s was within ±10° and ±0.50 m.

Training uses SAC with one environment and a replay buffer retained across curriculum stages.
