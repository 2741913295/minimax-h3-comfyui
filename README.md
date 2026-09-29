# MiniMax-H3-ComfyUI

MiniMax H3 的 ComfyUI 本地推理扩展，包含 PDD Acc 加速节点、MiniMax H3 批处理工具，以及多参考、首帧、尾帧、首尾帧和竖屏转横屏视频扩展工作流。

- PDD Acc：为 FL2VA / Ref2VA 应用官方 8 Step PDD Acc 权重。
- 多参考生视频：图片、视频、音频可组合输入。
- 首帧、尾帧、首尾帧生视频：读取本地 Excel 提示词并批量生成。
- 竖屏转横屏：MiniMax H3 Fun ControlNet 2.0 的视频 Inpainting / Outpainting 流程。
- SageAttention：批处理任务默认要求 ComfyUI 使用 `--use-sage-attention` 启动。
- 指标记录：每个输出视频旁保存 `*_metrics.json`，记录推理时间、峰值显存和 H3 分阶段性能数据。

## 目录结构

```text
ComfyUI/
├── custom_nodes/
│   └── ComfyUI-MiniMax-H3-PDD-Acc/
│       ├── nodes.py
│       ├── pdd_acc_core.py
│       ├── example_workflows/
│       ├── tests/
│       ├── convert_pdd_acc.py
│       ├── bake_pdd_trunk.py
│       └── README.md
├── script_examples/
│   ├── minimax_h3_pdd_batch.py
│   └── minimax_h3_fun_outpaint.py
├── data/
│   ├── video-data/
│   │   ├── 2d/
│   │   ├── 3d/
│   │   ├── live_shot/
│   │   └── 对应提示词.xlsx
│   └── H3多参考测试案例/
│       ├── case_qr_08_别墅客厅/
│       ├── case_dd_01_红雨降临/
│       ├── case_qr_07_徐家老宅继位/
│       └── video/
├── models/
│   ├── diffusion_models/
│   ├── text_encoders/
│   ├── vae/
│   ├── pdd_acc/
│   └── model_patches/
└── outputs/
```

## 环境

推荐环境：

| 项目 | 版本 |
|---|---|
| Python | 3.11 |
| PyTorch | CUDA 13.0 版本 |
| CUDA 驱动 | 支持 CUDA 13.0 |
| GPU | 48 GB 显存或更高 |
| ComfyUI | 0.37.0 或更新版本 |

创建环境并安装 ComfyUI 依赖：

```bash
conda create -n comfyui-h3 python=3.11 -y
conda activate comfyui-h3

cd /data/xiawei/project/ComfyUI

python -m pip install torch torchvision \
  --index-url https://download.pytorch.org/whl/cu130

python -m pip install -r requirements.txt \
  -i https://mirrors.ustc.edu.cn/pypi/simple
```

安装 SageAttention：

```bash
/data/xiawei/envs/comfyui-h3/bin/python -m pip install sageattention \
  -i https://mirrors.ustc.edu.cn/pypi/simple
```

安装插件：

```bash
cd /data/xiawei/project/ComfyUI/custom_nodes
git clone https://github.com/Jalen-Brunson/ComfyUI-MiniMax-H3-PDD-Acc
```

本地项目目录已经存在时无需再次克隆：

```text
/data/xiawei/project/ComfyUI/custom_nodes/ComfyUI-MiniMax-H3-PDD-Acc
```

## 模型

### 必需模型

| 功能 | 文件 | ComfyUI 目录 | 本地模型目录 |
|---|---|---|---|
| 首帧、尾帧、首尾帧 | `minimax_h3_fl2va_pruned_int8_convrot.safetensors` | `models/diffusion_models/` | `/data/xiawei/models/MiniMax-H3/diffusion_models/` |
| 多参考、视频扩展 | `minimax_h3_ref2va_pruned_int8_convrot.safetensors` | `models/diffusion_models/` | `/data/xiawei/models/MiniMax-H3/diffusion_models/` |
| 文本编码器 | `qwen3vl_32b_minimax_h3_int8_convrot.safetensors` | `models/text_encoders/` | `/data/xiawei/models/MiniMax-H3/text_encoders/` |
| 视频 VAE | `minimax_h3_video_vae_fp16.safetensors` | `models/vae/` | `/data/xiawei/models/MiniMax-H3/vae/` |
| 音频 VAE | `minimax_h3_audio_vae_fp32.safetensors` | `models/vae/` | `/data/xiawei/models/MiniMax-H3/vae/` |
| FL2VA PDD | `MiniMax-H3-FL2VA-Acc-8Step.safetensors` | `models/pdd_acc/` | `/data/xiawei/models/MiniMax-H3-PDD/` |
| Ref2VA PDD | `MiniMax-H3-Ref2VA-Acc-8Step.safetensors` | `models/pdd_acc/` | `/data/xiawei/models/MiniMax-H3-PDD/` |
| 视频扩展 | `minimax_h3_fun_controlnet_union_2.0_pruned_int8_convrot.safetensors` | `models/model_patches/` | `/data/xiawei/models/MiniMax-H3/model_patches/` |

### ModelScope 下载命令

```bash
modelscope download --model Comfy-Org/MiniMax-H3 \
  diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors \
  --local_dir /data/xiawei/models/MiniMax-H3

modelscope download --model Comfy-Org/MiniMax-H3 \
  diffusion_models/minimax_h3_ref2va_pruned_int8_convrot.safetensors \
  --local_dir /data/xiawei/models/MiniMax-H3

modelscope download --model Comfy-Org/MiniMax-H3 \
  text_encoders/qwen3vl_32b_minimax_h3_int8_convrot.safetensors \
  --local_dir /data/xiawei/models/MiniMax-H3

modelscope download --model Comfy-Org/MiniMax-H3 \
  vae/minimax_h3_video_vae_fp16.safetensors \
  --local_dir /data/xiawei/models/MiniMax-H3

modelscope download --model Comfy-Org/MiniMax-H3 \
  vae/minimax_h3_audio_vae_fp32.safetensors \
  --local_dir /data/xiawei/models/MiniMax-H3

modelscope download --model PAI/MiniMax-H3-Acc-LoRAs \
  MiniMax-H3-FL2VA-Acc-8Step.safetensors \
  --local_dir /data/xiawei/models/MiniMax-H3-PDD

modelscope download --model PAI/MiniMax-H3-Acc-LoRAs \
  MiniMax-H3-Ref2VA-Acc-8Step.safetensors \
  --local_dir /data/xiawei/models/MiniMax-H3-PDD

modelscope download --model Comfy-Org/MiniMax-H3 \
  model_patches/minimax_h3_fun_controlnet_union_2.0_pruned_int8_convrot.safetensors \
  --local_dir /data/xiawei/models/MiniMax-H3
```

### 软链接

```bash
cd /data/xiawei/project/ComfyUI

mkdir -p models/diffusion_models models/text_encoders models/vae
mkdir -p models/pdd_acc models/model_patches

ln -sfn /data/xiawei/models/MiniMax-H3/diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors \
  models/diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors

ln -sfn /data/xiawei/models/MiniMax-H3/diffusion_models/minimax_h3_ref2va_pruned_int8_convrot.safetensors \
  models/diffusion_models/minimax_h3_ref2va_pruned_int8_convrot.safetensors

ln -sfn /data/xiawei/models/MiniMax-H3/text_encoders/qwen3vl_32b_minimax_h3_int8_convrot.safetensors \
  models/text_encoders/qwen3vl_32b_minimax_h3_int8_convrot.safetensors

ln -sfn /data/xiawei/models/MiniMax-H3/vae/minimax_h3_video_vae_fp16.safetensors \
  models/vae/minimax_h3_video_vae_fp16.safetensors

ln -sfn /data/xiawei/models/MiniMax-H3/vae/minimax_h3_audio_vae_fp32.safetensors \
  models/vae/minimax_h3_audio_vae_fp32.safetensors

ln -sfn /data/xiawei/models/MiniMax-H3-PDD/MiniMax-H3-FL2VA-Acc-8Step.safetensors \
  models/pdd_acc/MiniMax-H3-FL2VA-Acc-8Step.safetensors

ln -sfn /data/xiawei/models/MiniMax-H3-PDD/MiniMax-H3-Ref2VA-Acc-8Step.safetensors \
  models/pdd_acc/MiniMax-H3-Ref2VA-Acc-8Step.safetensors

ln -sfn /data/xiawei/models/MiniMax-H3/model_patches/minimax_h3_fun_controlnet_union_2.0_pruned_int8_convrot.safetensors \
  models/model_patches/minimax_h3_fun_controlnet_union_2.0_pruned_int8_convrot.safetensors
```

## 节点

### MiniMax H3 PDD Acc LoRA (Apply)

节点名：`MiniMaxH3PDDAccApply`

输入：

| 输入 | 说明 |
|---|---|
| `model` | MiniMax H3 UNET |
| `pdd_file` | `models/pdd_acc/` 中的 PDD Acc 文件 |
| `nfe` | PDD 推理步数，支持 `8`、`6`、`4` |
| `lora_strength` | Trunk LoRA 强度，默认 `1.0` |
| `head_strength` | PDD Head 强度，默认 `1.0` |
| `on_off_grid` | 非训练采样网格处理方式，默认 `error` |
| `partition` | 自定义 PDD 分块，例如 `8,8,4,4,4,4` |
| `enabled` | 是否启用 PDD |
| `bypass_sigmas` | `enabled=false` 时使用的基础模型采样计划 |
| `partition_check` | FL2VA / Ref2VA 主干匹配检查 |

输出：

| 输出 | 说明 |
|---|---|
| `MODEL` | 已应用 PDD Acc 的模型 |
| `SIGMAS` | 与 PDD 分块匹配的采样 sigma |
| `info` | 模型主干、LoRA 模块、分块和检查结果 |

PDD 设置：

| 目标步数 | 分块 |
|---|---|
| 8 | `4,4,4,4,4,4,4,4` |
| 6 | `8,8,4,4,4,4` |
| 4 | `8,8,8,8` |

PDD 工作流必须使用：

| 项目 | 值 |
|---|---|
| 采样器 | `euler` |
| 指导 | `CFG 1.0` / `BasicGuider` |
| 视频 Sigma Shift | `12.0` |
| 音频 Sigma Shift | `3.0` |
| SIGMAS | PDD Apply 节点或 PDD Scheduler 节点输出 |

FL2VA 必须搭配 `MiniMax-H3-FL2VA-Acc-8Step.safetensors`；Ref2VA 必须搭配 `MiniMax-H3-Ref2VA-Acc-8Step.safetensors`。

### MiniMax H3 PDD Acc Scheduler

节点名：`MiniMaxH3PDDAccScheduler`

用于部分去噪、两阶段采样和自定义工作流。输入 `nfe`、`denoise`、`partition`，输出符合 PDD 训练网格的 `SIGMAS`。

### MiniMax H3 PDD Acc Warmup Scheduler (2-phase)

节点名：`MiniMaxH3PDDAccWarmupScheduler`

第一阶段运行基础模型，第二阶段运行 PDD 模型。适用于人物一致性、结构稳定性或参考图约束较强的任务。

### MiniMax H3 AV Latent Upscale By

节点名：`MiniMaxH3AVLatentUpscaleBy`

对 MiniMax H3 的视频 latent 进行空间放大，音频 latent 保持不变。用于低分辨率先生成，再部分去噪细化到高分辨率。

## 示例工作流

| 文件 | 功能 |
|---|---|
| `example_workflows/pdd_acc_t2v_basic.json` | PDD 8 Step 文生视频和音频 |
| `example_workflows/pdd_acc_t2v_warmup_split.json` | 基础模型 Warmup + PDD 后段采样 |
| `example_workflows/pdd_acc_t2v_latent_upscale.json` | 两阶段 latent 放大和细化 |
| `example_workflows/pdd_video_upscale_long.json` | 长视频分窗 latent 放大 |

在 ComfyUI 页面直接拖入 JSON 文件即可加载工作流。

## 启动 ComfyUI

GPU 7、端口 8187：

```bash
cd /data/xiawei/project/ComfyUI

/data/xiawei/envs/comfyui-h3/bin/python -u main.py \
  --listen 127.0.0.1 \
  --port 8187 \
  --cuda-device 7 \
  --input-directory /data/xiawei/project/ComfyUI/data \
  --output-directory /data/xiawei/project/ComfyUI/outputs \
  --use-sage-attention \
  --database-url sqlite:////data/xiawei/project/ComfyUI/user_sage.db \
  2>&1 | tee logs/comfyui_gpu7_sage.log
```

`--cuda-device 7` 后，ComfyUI 内部显示为 `cuda:0` 属于正常现象：进程只看得到物理 7 号卡。

## 批处理工具

批处理入口：

```text
/data/xiawei/project/ComfyUI/script_examples/minimax_h3_pdd_batch.py
```

通用参数：

| 参数 | 说明 |
|---|---|
| `--server` | ComfyUI 服务地址 |
| `--gpu-index` | 用于 `nvidia-smi` 监控的物理 GPU 编号 |
| `--durations` | `4 15`，生成 4 秒和 15 秒 |
| `--output-variant` | 输出子目录名称 |
| `--force` | 已存在同名输出时重新生成 |
| `--ref-video-vae-scale` | 多参考视频在 VAE 编码前的缩放比例，范围 `0.25-1.0` |
| `--no-warmup` | 跳过模型热身 |
| `--allow-sdpa` | 允许未启用 SageAttention 的对照运行 |

批处理默认检查 ComfyUI 是否由 `--use-sage-attention` 启动。未启用 SageAttention 时，添加 `--allow-sdpa` 才能运行。

### 多参考生视频

输入目录：

```text
data/H3多参考测试案例/
├── case_qr_08_别墅客厅/
├── case_dd_01_红雨降临/
└── case_qr_07_徐家老宅继位/
```

运行 `case_qr_08_别墅客厅`，生成 768p 4 秒和 15 秒视频：

```bash
cd /data/xiawei/project/ComfyUI

/data/xiawei/envs/comfyui-h3/bin/python -u \
  script_examples/minimax_h3_pdd_batch.py multi-reference \
  --server http://127.0.0.1:8187 \
  --gpu-index 7 \
  --cases case_qr_08_别墅客厅 \
  --durations 4 15 \
  --ref-video-vae-scale 0.5 \
  --output-variant sage \
  2>&1 | tee logs/multi_reference_gpu7_sage.log
```

`--ref-video-vae-scale 0.5` 仅缩小多参考输入视频的 VAE 编码尺寸，文本编码器仍使用原始视频信息。设置为 `1.0` 时关闭视频编码缩放。

### 首帧生视频

输入目录和提示词：

```text
data/video-data/
├── 2d/
├── 3d/
├── live_shot/
└── 对应提示词.xlsx
```

每个子目录取第一个案例：

```bash
cd /data/xiawei/project/ComfyUI

/data/xiawei/envs/comfyui-h3/bin/python -u \
  script_examples/minimax_h3_pdd_batch.py first-frame \
  --server http://127.0.0.1:8187 \
  --gpu-index 7 \
  --per-category 1 \
  --durations 4 15 \
  --output-variant sage \
  2>&1 | tee logs/first_frame_gpu7_sage.log
```

全部案例：

```bash
/data/xiawei/envs/comfyui-h3/bin/python -u \
  script_examples/minimax_h3_pdd_batch.py first-frame \
  --server http://127.0.0.1:8187 \
  --gpu-index 7 \
  --per-category 0 \
  --durations 4 15 \
  --output-variant sage \
  2>&1 | tee logs/first_frame_all_gpu7_sage.log
```

### 尾帧生视频

每个类别按 Excel 顺序取前两个案例：

```bash
cd /data/xiawei/project/ComfyUI

/data/xiawei/envs/comfyui-h3/bin/python -u \
  script_examples/minimax_h3_pdd_batch.py last-frame \
  --server http://127.0.0.1:8187 \
  --gpu-index 7 \
  --per-category 2 \
  --durations 4 15 \
  --output-variant sage \
  2>&1 | tee logs/last_frame_gpu7_sage.log
```

### 首尾帧生视频

2D 图片 `1:2`、`2:3`、`7:8` 分别作为首帧和尾帧。提示词使用每组首帧图片在 Excel 中的提示词。

```bash
cd /data/xiawei/project/ComfyUI

/data/xiawei/envs/comfyui-h3/bin/python -u \
  script_examples/minimax_h3_pdd_batch.py first-last-frame \
  --server http://127.0.0.1:8187 \
  --gpu-index 7 \
  --pairs 1:2 2:3 7:8 \
  --durations 4 15 \
  --output-variant sage \
  2>&1 | tee logs/first_last_frame_gpu7_sage.log
```

### 竖屏转横屏视频扩展

输入目录：

```text
data/H3多参考测试案例/video/
├── video_1.mp4
├── video_2.mp4
└── text.txt
```

视频扩展使用 MiniMax H3 Ref2VA、Fun ControlNet Union 2.0 和 H3 原生 latent noise mask。

处理流程：

```text
竖屏视频
  → 等比例缩放到目标高度
  → 1344x768 黑色画布中央放置原视频
  → 左右区域 mask=1，中间区域 mask=0
  → Fun ControlNet Inpainting / Outpainting
  → 生成左右场景细节
  → 还原中央源视频和原始音频
```

运行：

```bash
cd /data/xiawei/project/ComfyUI

/data/xiawei/envs/comfyui-h3/bin/python -u \
  script_examples/minimax_h3_pdd_batch.py vertical-video-to-horizontal \
  --server http://127.0.0.1:8187 \
  --gpu-index 7 \
  --videos video_1 video_2 \
  --output-variant sage-anchor-cutfix \
  2>&1 | tee logs/outpaint_gpu7_sage_anchor_cutfix.log
```

视频扩展会生成以下调试文件：

```text
outputs/H3多参考测试案例/竖屏转横屏-FunControlNet-Outpaint/
└── <video_id>/1344x768/<variant>/
    ├── debug_source_canvas.mp4
    ├── debug_mask.mp4
    ├── debug_masked_source.mp4
    ├── debug_preprocess.json
    ├── video_outpaint_*.mp4
    └── video_outpaint_*_metrics.json
```

`debug_source_canvas.mp4` 为中央原视频加左右黑色区域；`debug_mask.mp4` 为中央黑色、左右白色；`debug_masked_source.mp4` 为仅保留中央原视频的画布。

## 输出与指标

输出目录按任务、案例、规格和实验版本划分：

```text
outputs/
├── H3多参考测试案例/
│   └── case_qr_08_别墅客厅/
│       └── 768p_15s/
│           └── sage/
│               ├── multi_ref_*.mp4
│               └── multi_ref_*_metrics.json
├── 首帧生视频/
├── 尾帧生视频/
├── 首尾帧生视频/
└── H3多参考测试案例/
    └── 竖屏转横屏-FunControlNet-Outpaint/
```

每个 `*_metrics.json` 包含：

| 字段 | 内容 |
|---|---|
| `execution_seconds` | ComfyUI 正式执行时长，不含模型首次加载、任务排队和热身 |
| `gpu_peak_memory_mib` | 运行期间采样到的物理 GPU 峰值显存 |
| `gpu_peak_increment_mib` | 相对任务开始时的显存增量 |
| `component_profile` | VAE、文本编码、Transformer、Attention、解码、保存等阶段指标 |
| `attention_backend_calls` | Sage / SDPA 后端调用统计 |
| `output_file` | 生成视频绝对路径 |

终端结束时会输出任务数、成功数、失败数、单任务推理时长、批次总时长和批次峰值显存。

## PDD 文件转换

原始 PDD 文件可转换为 ComfyUI 键名格式：

```bash
cd /data/xiawei/project/ComfyUI/custom_nodes/ComfyUI-MiniMax-H3-PDD-Acc

/data/xiawei/envs/comfyui-h3/bin/python \
  convert_pdd_acc.py \
  /data/xiawei/models/MiniMax-H3-PDD/MiniMax-H3-Ref2VA-Acc-8Step.safetensors \
  /data/xiawei/models/MiniMax-H3-PDD/minimax_h3_ref2va_pdd_acc_8step_comfyui.safetensors
```

## PDD Trunk Bake

```bash
cd /data/xiawei/project/ComfyUI/custom_nodes/ComfyUI-MiniMax-H3-PDD-Acc

/data/xiawei/envs/comfyui-h3/bin/python bake_pdd_trunk.py --check \
  --base /data/xiawei/models/MiniMax-H3/diffusion_models/minimax_h3_ref2va_pruned_int8_convrot.safetensors \
  --pdd /data/xiawei/models/MiniMax-H3-PDD/MiniMax-H3-Ref2VA-Acc-8Step.safetensors

/data/xiawei/envs/comfyui-h3/bin/python bake_pdd_trunk.py \
  --base /data/xiawei/models/MiniMax-H3/diffusion_models/minimax_h3_ref2va_pruned_int8_convrot.safetensors \
  --pdd /data/xiawei/models/MiniMax-H3-PDD/MiniMax-H3-Ref2VA-Acc-8Step.safetensors \
  --out /data/xiawei/models/MiniMax-H3/diffusion_models/minimax_h3_ref2va_pddbaked_int8_convrot.safetensors
```

使用 baked UNET 时，在 `MiniMaxH3PDDAccApply` 中设置 `lora_strength=0.0`。

## 测试

```bash
cd /data/xiawei/project/ComfyUI/custom_nodes/ComfyUI-MiniMax-H3-PDD-Acc

/data/xiawei/envs/comfyui-h3/bin/python tests/test_pdd_acc.py
```

完整文件验证：

```bash
cd /data/xiawei/project/ComfyUI/custom_nodes/ComfyUI-MiniMax-H3-PDD-Acc

PDD_ACC_SLOW=1 /data/xiawei/envs/comfyui-h3/bin/python tests/test_pdd_acc.py
```

## License

Apache-2.0

