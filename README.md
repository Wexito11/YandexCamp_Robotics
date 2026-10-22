# GFS-X RL - reinforcement learning for an autonomous tracked robot

🇷🇺 [Русская версия](README.ru.md) · Original team repository: [DrPepper18/YandexCamp_Robotics](https://github.com/DrPepper18/YandexCamp_Robotics)

Team project from the **Yandex Education × Ural Federal University studcamp «Industrial Robotics»** (July 2026, Yekaterinburg).

The task: a **XiaoR GFS-X** tracked robot (Raspberry Pi 4B) has to find a ball with its camera, drive around obstacles and stop with the ball between its grippers. The policy is trained in a Unity simulation and then transferred to the real robot.

## My part

I was responsible for the reinforcement learning side of the project:

- **Agent and training setup** - PPO + LSTM policy in Unity ML-Agents (PyTorch backend), 15 observations × 4 stacked frames, 3 continuous actions + 1 discrete branch, training on 40 parallel arenas in a headless Linux build.
- **Reward design** - shaped approach reward, alignment bonus, action-rate penalty for smooth motor commands on the real hardware.
- **Finding reward hacking** - watched the trained agent, found strategies that collected reward without solving the task and fixed each one:
  - *blind homing* - approaching the ball using the simulator's privileged distance, even when the camera did not see it → approach reward only when the ball is visible;
  - *stare farming* - staying in front of the ball to farm the alignment bonus;
  - *snowplow* - pushing the ball with the bumper to farm the approach reward → contact penalty, with an exception for the gripper zone.
- **Curriculum learning** - a 7-stage curriculum controlled by one parameter: ball distance grows from 1.2 m to 2.6 m, obstacles go from fixed to random, and the last stage adds vision noise and detection dropouts.
- **Evaluation** - compared checkpoints in TensorBoard and in deterministic inference on the full competition rules (random obstacles, far start, balls near walls). Best checkpoint: stable reward ≈ **+5.2** at 7M steps.
- **Sim-to-real** - exported the model to ONNX, connected it to the robot through ROS2 (`/cmd_vel`) and real YOLO detections over UDP, calibrated detection parameters on site.

## Stack

| | |
|---|---|
| Simulation | Unity 6, ML-Agents 1.1.0 |
| RL | PPO + LSTM (sequence length 64, memory 256), PyTorch |
| Vision | YOLO (UDP bridge from the robot) |
| Robot | XiaoR GFS-X, Raspberry Pi 4B, ROS2 |
| Tools | Python, C#, conda, TensorBoard, Linux |

## Repository structure

- `Assets/Scripts/RobotBrain.cs` - RL agent: observations, actions, rewards, episode logic
- `Assets/Scripts/ObstacleRandomizer.cs` - random obstacles every episode
- `Assets/Scripts/DomainRandomizer.cs` - sim-to-real noise
- `Assets/Scripts/SimulatedYoloCamera.cs`, `RealVision.cs` - simulated and real camera detections
- `Assets/Scripts/TrackController.cs`, `VirtualSensors.cs`, `GripperController.cs` - robot model
- `config.yaml`, `config_curriculum.yaml` - PPO + LSTM training configs
- `README_TRAINING.md` - training guide for the team (in Russian)

## Run

1. Install Unity 6 (version in `ProjectSettings/ProjectVersion.txt`) and open the project.
2. Create the Python environment:
   ```bash
   conda create -n camp python=3.10 -y && conda activate camp
   pip install torch --index-url https://download.pytorch.org/whl/cpu
   pip install grpcio==1.48.2 protobuf==3.20.3 "setuptools<70" mlagents==1.1.0
   ```
3. Train: `mlagents-learn config.yaml --run-id=<new_id>`, then press Play in Unity.
4. Watch a trained model: set Behavior Type to **Inference Only**, choose an `.onnx` from `Assets/ModelsW`, and turn on **Deterministic Inference**.
