# MMPose 自动化脚本使用指南

## 脚本说明

本项目包含三个自动化PowerShell脚本，用于简化MMPose的环境配置和使用流程。

### 1. `setup_environment.ps1` - 环境初始化脚本

**功能**: 一键完成conda环境创建、依赖安装和MMPose包安装。

**使用方法**:
```powershell
.\setup_environment.ps1
```

**执行步骤**:
1. 创建名为`mmpose`的conda环境（Python 3.8）
2. 安装`requirements.txt`中的所有依赖
3. 安装openmim、mmengine、mmcv和mmdet
4. 安装mmpose包

**注意**: 首次使用项目时运行一次即可。

---

### 2. `run_pipeline.ps1` - 完整分析流程脚本

**功能**: 自动化执行从视频输入到3D骨架可视化的完整分析流程。

**使用模式**:

#### 模式A: 单个视频处理
```powershell
.\run_pipeline.ps1 -VideoPath "data/prof/Curry_1.mp4" -VideoName "Curry_1"
```

#### 模式B: 批量视频处理
```powershell
.\run_pipeline.ps1 -BatchMode -VideoNames "Curry_1","Curry_2"
```

#### 可选参数:
- `-Device`: 指定计算设备，默认为`cpu`，可选`cuda`（需要GPU支持）
  ```powershell
  .\run_pipeline.ps1 -BatchMode -VideoNames "Curry_1","Curry_2" -Device cuda
  ```

**执行流程**:
1. **3D姿态估计**: 使用RTMDet和RTMPose进行2D检测，再通过VideoPose3D提升到3D
2. **JSON转NPY**: 将预测结果从JSON格式转换为NumPy数组
3. **投篮阶段划分**: 自动识别投篮动作的不同阶段
4. **建立标准模板**: 基于多个样本建立标准动作模板
5. **3D可视化**: 生成带有阶段标注的3D骨架视频

**输出位置**:
- `vis_results/`: 可视化视频和中间结果
- `data/prof_3d/`: 3D姿态数据(.npy)、阶段划分结果(.json)、标准模板(.json)

---

### 3. `quick_start.ps1` - 快速启动脚本

**功能**: 快速切换到项目目录并激活conda环境。

**使用方法**:
```powershell
.\quick_start.ps1
```

**适用场景**: 每次打开新终端准备运行项目时使用。

---

## 完整使用流程

### 首次使用

1. **克隆或下载项目到本地**
2. **确保已安装Anaconda或Miniconda**
3. **运行环境初始化**:
   ```powershell
   .\setup_environment.ps1
   ```
   等待约10-20分钟完成所有依赖安装

4. **准备视频数据**:
   - 将待分析的视频放入`data/prof/`目录
   - 确保已下载所需的模型权重文件到`weights/`目录

5. **运行分析**:
   ```powershell
   # 批量处理默认视频
   .\run_pipeline.ps1 -BatchMode
   
   # 或处理自定义视频
   .\run_pipeline.ps1 -VideoPath "data/prof/MyVideo.mp4" -VideoName "MyVideo"
   ```

6. **查看结果**:
   - 在`vis_results/`目录查看可视化视频
   - 在`data/prof_3d/`目录查看数据文件

### 日常使用

每次打开新终端后:
```powershell
.\quick_start.ps1
```

然后根据需要运行分析命令。

---

## 手动命令参考

如果需要单独运行某个步骤，可以使用以下命令:

### 1. 3D姿态估计
```powershell
python demo/body3d_pose_lifter_demo.py `
    demo/mmdetection_cfg/rtmdet_m_640-8xb32_coco-person.py `
    weights/rtmdet_m_8xb32-100e_coco-obj365-person-235e8209.pth `
    configs/body_2d_keypoint/rtmpose/body8/rtmpose-m_8xb256-420e_body8-256x192.py `
    weights/rtmpose-m_simcc-body7_pt-body7_420e-256x192-e48f03d0_20230504.pth `
    configs/body_3d_keypoint/video_pose_lift/h36m/video-pose-lift_tcn-27frm-supv_8xb128-160e_h36m.py `
    weights/videopose_h36m_27frames_fullconv_supervised-fe8fbba9_20210527.pth `
    --input data/prof/Curry_1.mp4 `
    --output-root vis_results `
    --save-predictions `
    --device cpu
```

### 2. JSON转NPY
```powershell
python tools/parse_prof_json_to_npy.py --json-dir vis_results --names Curry_1 Curry_2 --out-dir data/prof_3d
```

### 3. 投篮阶段划分
```powershell
python tools/shooting_phase_split.py --npy-path data/prof_3d/Curry_1.npy
```

### 4. 建立标准模板
```powershell
python tools/build_standard_template.py --prof-dir data/prof_3d --names Curry_1 Curry_2 --out data/prof_3d/standard_template.json
```

### 5. 生成3D骨架视频（不带阶段）
```powershell
python tools/visualize_prof_3d.py --npy-path data/prof_3d/Curry_1.npy --out-dir vis_results
```

### 6. 生成3D骨架视频（带阶段）
```powershell
python tools/visualize_prof_3d.py --npy-path data/prof_3d/Curry_1.npy --phases-json data/prof_3d/Curry_1_phases.json --out-dir vis_results
```

---

## 常见问题

### 1. PowerShell执行策略限制

如果运行脚本时遇到权限错误，需要修改执行策略:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### 2. Conda命令未找到

确保Anaconda或Miniconda已正确安装并添加到系统PATH。

### 3. GPU加速

如果有NVIDIA GPU且已安装CUDA:
```powershell
.\run_pipeline.ps1 -BatchMode -Device cuda
```

### 4. 视频文件路径

确保视频路径正确，使用相对路径或绝对路径均可:
```powershell
# 相对路径
.\run_pipeline.ps1 -VideoPath "data/prof/MyVideo.mp4" -VideoName "MyVideo"

# 绝对路径
.\run_pipeline.ps1 -VideoPath "D:\Videos\MyVideo.mp4" -VideoName "MyVideo"
```

---

## 项目结构

```
mmpose/
├── setup_environment.ps1      # 环境初始化脚本
├── run_pipeline.ps1           # 完整流程自动化脚本
├── quick_start.ps1            # 快速启动脚本
├── README_AUTOMATION.md       # 本文档
├── data/
│   ├── prof/                  # 输入视频目录
│   └── prof_3d/               # 3D数据输出目录
├── vis_results/               # 可视化结果输出目录
├── weights/                   # 模型权重文件
├── configs/                   # 配置文件
├── demo/                      # 演示脚本
└── tools/                     # 工具脚本
```

---

## 技术支持

如有问题，请查阅:
- [MMPose官方文档](https://mmpose.readthedocs.io/)
- [GitHub Issues](https://github.com/open-mmlab/mmpose/issues)
