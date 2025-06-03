# SailOpt – Wingsail Optimal-Angle Prediction

利用逐小时气象（ERA5）+ 船舶 AIS 轨迹，训练深度学习模型（LSTM / GRU / TCN / Informer）预测 **翼形风帆** 的最优攻角与桅杆俯仰角，实现节油与减排。
## 📁 目录结构

```plaintext
sail_opt/
├── data/
│   ├── aligned_weather_ship_sail_2022.csv         # 对齐后示例 CSV
│   └── test_aligned_weather_ship_sail.csv         # 48h 测试样本
├── configs/
│   └── default.yaml                                # 超参与路径配置
├── src/
│   ├── dataset.py                                  # 数据集与滑窗处理
│   ├── model.py                                    # LSTM/GRU/TCN/Informer 等模型
│   ├── train.py                                    # 训练脚本
│   └── utils.py                                    # 通用工具函数
├── run_train.py                                    # 训练入口
├── inference.py                                    # 推理入口
├── requirements.txt                                # 依赖列表
└── README.md                                       # 本文件
```
---

## 1. 数据集概要

### 1.1 物理来源  
| 模块         | 数据源                           | 作用                                   |
|--------------|----------------------------------|----------------------------------------|
| **气象场**   | Copernicus ERA5 (0.25° × 1 h)    | u10 / v10 风速、温度、气压、波浪等      |
| **船舶轨迹** | AIS Vessel Tracks (1 min → 1 h) | 纬经度、地速 SOG、航向 COG、船艏 Heading |
| **气动曲线** | CFD / 风洞 CL–CD                | 生成软标签 *opt_wing_angle*、*opt_mast_pitch* |

### 1.2 表结构（18 列）  

| #  | 字段                | 单位      | 描述                                |
|----|---------------------|-----------|-------------------------------------|
| 1  | `timestamp`         | —         | UTC 整点时间戳 (`YYYY-MM-DDTHH:00:00Z`) |
| 2  | `latitude` / `longitude` | °     | 船舶位置 (WGS-84)                   |
| 3  | `u10`, `v10`        | m/s       | 10 m 东西 / 南北风分量               |
| 4  | `wind_gust`         | m/s       | 10 m 阵风极值                       |
| 5  | `t2m`, `d2m`        | K         | 2 m 气温 / 露点                     |
| 6  | `msl`               | Pa        | 海平面气压                          |
| 7  | `tp`                | mm h⁻¹    | 上一小时总降水                      |
| 8  | `ssrd`              | W m⁻²     | 下行短波辐射                        |
| 9  | `swh`, `wave_dir`   | m / °     | 显著波高、平均波向                  |
| 10 | `sog`, `cog`, `heading` | kn / ° | 船速、航向、船艏指向                |
| 11 | `opt_wing_angle`    | °         | CFD 软标签：理论最优攻角            |
| 12 | `opt_mast_pitch`    | °         | 理论最优桅杆俯仰角 (`0.2 × angle`)  |

> **维度**  
> - **输入特征**：13 列  
> - **输出标签**：2 列  
> - 每行 = 某船、某整点的“天气 + 船况 + 最优帆姿”

### 1.3 典型张量  
- **序列窗口**：`seq_len = 6` → 过去 6 h  
- **张量形状**：`(batch, seq_len, 13)`  
- **标签**：`y = [opt_wing_angle, opt_mast_pitch]` 对应窗口最后一行  
- **损失函数**：`MSELoss()`（°²）

---

## 2. 快速上手

```bash
# 创建并激活虚拟环境
python -m venv venv && source venv/bin/activate

# 安装依赖
pip install -r requirements.txt

# 训练（默认使用 LSTM）
python run_train.py

# 覆盖超参示例
python run_train.py --model_name gru --learning_rate 5e-4 --batch_size 128
# 训练完成后，最优模型保存在 checkpoints/best.pt


```
## 3. 模型切换

| `--model_name` | 结构                     | 典型超参                                      |
|----------------|--------------------------|-----------------------------------------------|
| `lstm` (默认)  | 双层 LSTM + MLP          | `hidden_size=128`                             |
| `gru`          | 双层 GRU + MLP           | `hidden_size=128`                             |
| `tcn`          | 4× 残差 TCN              | `hidden_size=64`                              |
| `informer`     | 轻量 Transformer Encoder | `hidden_size=128`, `seq_len ≥ 24`             |

---

