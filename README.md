# HW3-Task1: 2DGS + AIGC 3D 资产生成与场景融合

## 项目简介

本项目为计算机视觉期末作业任务一，基于 2D Gaussian Splatting(2DGS)、threestudio、Magic123 实现三类 3D 资产构建、开源场景重建、多模型融合渲染全流程：

- **物体 A**：多视角图像 / 视频 + COLMAP + 2DGS 真实物体 3D 重建
- **物体 B**：文本 Prompt + threestudio 文生 3D 虚拟物体
- **物体 C**：单张图像 + Magic123 图生 3D 模型
- **背景场景**：基于开源数据集使用 2DGS 重建环境
- **多资产融合、空间摆放与多视角漫游视频渲染**

---

## 一、环境准备

### 1. 基础环境依赖

推荐使用 Conda 管理虚拟环境，Python 版本建议 3.9 / 3.10，适配主流 3D 重建与 AIGC 框架。


 1. 创建并激活虚拟环境
conda create -n cv_hw3_2dgs python=3.10 -y
conda activate cv_hw3_2dgs

 2. 基础工具依赖
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
pip install numpy opencv-python pillow tqdm scipy matplotlib
pip install plyfile trimesh open3d imageio ffmpeg-python

 3. 数据集/位姿解算依赖（COLMAP配套）
pip install pycolmap h5py
2. 第三方框架克隆与部署
本任务依赖 2DGS、threestudio 两大核心仓库，依次执行克隆、环境配置：

### 2.1 克隆 2DGS 仓库（真实场景 / 物体重建）
 克隆2DGS源码
git clone https://github.com/hustvl/2DGS.git

cd 2DGS

安装2DGS专属依赖 & 编译扩展
pip install -r requirements.txt

python setup.py install

cd ..
### 2.2 克隆 threestudio 仓库（文生 3D 模型）
git clone https://github.com/threestudio-project/threestudio.git

cd threestudio

pip install -r requirements.txt

cd ..

### 二、数据准备
1. 物体 A 数据（多视角重建）
使用手机拍摄环绕视频 / 多角度照片，存放至 ./data/obj_A/raw.mp4

2. 物体 B 数据（文生 3D）
无需额外图像数据，仅准备文本 Prompt，在配置文件中填写即可。

3. 物体 C 数据（单图生 3D）
拍摄单张物体照片，去除背景得到纯前景图
图片存放至 ./data/obj_C/input.png

4. 背景场景数据
下载 Mip-NeRF 360 开源数据集（garden / bicycle / counter 等），解压至 ./data/scene_background/

### 三、模型训练 & 重建命令
3.1 物体 A：2DGS 多视角重建（真实物体）
进入 2DGS 目录，执行训练重建：

bash
cd 2DGS
 2DGS 训练重建（基于COLMAP位姿数据，需先对raw运行colmap获得位姿数据）
python train.py \
    -s ../data/obj_A/colmap \
    --model_path ../output/obj_a_2dgs \
    --iterations 30000

 完成训练后，导出点云/高斯模型
python export_ply.py \
    -s ../data/obj_A/colmap \
    --model_path ../output/obj_a_2dgs
-s：COLMAP 数据集路径

--model_path：模型输出保存路径

### 3.2 物体 B：threestudio 文本生成 3D 模型
进入 threestudio 目录，执行文生 3D 训练：

bash
cd threestudio

基于SDS Loss + 2D扩散模型 文生3D
python launch.py \
    --config configs/text-guided/gs-text.yaml \
    prompt "a cute wooden table, high detail, realistic texture" \
    system.output_dir ../output/obj_b_threestudio

训练完成后导出Mesh
python export.py \
    --config configs/text-guided/gs-text.yaml \
    system.output_dir ../output/obj_b_threestudio
prompt：自定义生成文本提示词

system.output_dir：生成模型保存路径

### 3.3 物体 C：Magic123 单图转 3D 模型
bash
cd Magic123

单图3D重建训练
python run.py \
    --input_img ../data/obj_C/input.jpg \
    --output_dir ../output/obj_c_magic123
    
### 3.4 背景场景：2DGS 场景重建
bash
cd 2DGS

### 开源场景2DGS重建,需要提前将文件存放至scene_background文件夹中,由于文件过大上传不了需要自行下载
python train.py \
    -s ../data/scene_background \
    --model_path ../output/background_scene_2dgs \
    --iterations 40000
