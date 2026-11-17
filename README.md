# SelfRisingRobot

两轴舵机自起立机器人 **"poco"**。

在 MuJoCo 中用强化学习训练出"倒地后自己爬起来"的动作，再把训练好的策略网络部署到搭载 M5Atom 的实机机器人上运行。

原始说明文章：

https://homemadegarbage.com/rl13

## 目录结构

```
SelfRisingRobot/
├── README.md                      本文件：项目总览与目录结构
├── 3Dmodel/                       实机结构的 3D 打印文件（STL）
│   ├── footP.stl                  足部底座（机身主体）
│   ├── arm1P.stl                  一级臂
│   ├── arm2P.stl                  二级臂
│   └── armhornP.stl               舵机摇臂转接件
├── RL/                            仿真与强化学习部分
│   ├── README.md                  RL 部分的使用说明（日文）
│   ├── robo1.xml                  MuJoCo 模型定义（2 自由度、位置伺服、步长 1 ms）
│   ├── robo1_env.py               Gymnasium 环境：观测 4 维、动作 2 维、控制周期 20 ms
│   ├── assets/                    robo1.xml 引用的 STL 网格
│   │   ├── foot.stl               足部
│   │   ├── arm1.stl               一级臂
│   │   ├── arm2.stl               二级臂
│   │   ├── servo1.stl             舵机 1 外形
│   │   ├── servo2.stl             舵机 2 外形
│   │   └── ATOM.stl               M5Atom 控制板外形
│   ├── robo1_getup_ppo.zip        已训练好的 Stable-Baselines3 PPO 模型
│   ├── play_robo1_policy.py       在 MuJoCo 中回放策略（可视化）
│   ├── eval_robo1_policy.py       从各种倒地姿态出发做评估
│   ├── scripted_getup.py          用 viewer 回放脚本化的参考起立轨迹
│   ├── search_all_getup.py        为每种倒地姿态搜索候选起立轨迹，输出 BEST_SEQ
│   ├── getup_reference.py         各倒地姿态对应的舵机目标角 waypoint 序列（教师数据）
│   ├── pretrain_robo1_from_scripted.py  用参考轨迹生成"观测-动作"对，预训练 PPO 策略
│   ├── train_robo1.py             PPO 继续训练
│   ├── export_policy_header.py    把训练好的模型导出成 Arduino 用的 C 头文件
│   └── requirements.txt           Python 依赖
└── Arduino/                       M5Atom 固件
    ├── README.md                  Arduino 部分的使用说明（日文）
    └── robo03/
        ├── robo03.ino             主程序：IMU 读取 + 卡尔曼滤波 + 策略推理 + Web 控制
        └── policy_network.h       已导出的策略网络（4→64→64→2 的 MLP，纯 C 数组）
```

## 3D 模型

`3Dmodel/` 存放制作实机用的 STL 文件。

- `footP.stl`
- `arm1P.stl`
- `arm2P.stl`
- `armhornP.stl`

## 强化学习（RL/）

`RL/` 存放仿真环境和已训练模型。主要内容：

- MuJoCo 模型
- Gymnasium 环境
- Stable-Baselines3 PPO 训练好的模型
- 回放与评估脚本

观测 4 维（roll、pitch、两个舵机的目标角），动作 2 维（目标角增量，限幅 ±0.08 rad，角度范围 ±1.55 rad）。

常用命令（在 `RL/` 目录下执行）：

```powershell
python -m venv .venv
.\\.venv\\Scripts\\Activate.ps1
pip install -r requirements.txt

python play_robo1_policy.py                 # 回放
python play_robo1_policy.py --pose roll_pos # 指定初始倒地姿态
python eval_robo1_policy.py                 # 评估
```

训练流程：`pretrain_robo1_from_scripted.py`（用教师轨迹预训练）→ `train_robo1.py`（PPO 继续训练）→ `export_policy_header.py`（导出 `policy_network.h`）。

## Arduino（实机固件）

`Arduino/` 存放控制实机的 M5Atom 程序。主要内容：

- `robo03.ino`
- `policy_network.h`

`policy_network.h` 是训练好的策略转成的 C 头文件，用于在 M5Atom 上实时跑起立动作。它由 `robo03.ino` 直接 include，因此必须和 `.ino` 放在同一个 `robo03` 文件夹里。

依赖库（在 Arduino IDE 的库管理器中安装）：

- M5Atom
- Kalman
- ESP32Servo

另需安装 ESP32 开发板环境。

烧写：用 Arduino IDE 打开 `robo03/robo03.ino`，选择 M5Atom 编译上传。

Wi-Fi 控制：启动后 M5Atom 自己开一个热点。

- SSID：`robo1`
- 密码：`password`
- 地址：`http://192.168.42.1`

浏览器打开后可以手动调舵机、启停起立动作、开关自动起立。M5Atom 本体按键也能触发起立；起立过程中再按一次，动作停止并回到手动模式。

注意：

- 舵机 1 接 GPIO 26，舵机 2 接 GPIO 32。
- 舵机偏置和脉宽可在网页上调，会存进 Preferences。
- 想用别的训练结果，直接替换 `policy_network.h` 即可。

## 说明

- 各子文件夹的详细用法见其中的 README（目前为日文）。
- `robo1_getup_ppo.zip` 依赖 `robo1.xml` 和 `robo1_env.py` 的环境定义，模型和环境不能分开改。
- 缺少 `assets/` 里的 STL 文件时 MuJoCo 无法加载模型。
- 请在仓库根目录下执行命令。
