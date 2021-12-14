<div align="center">



<img src="https://user-images.githubusercontent.com/20336673/145025177-6dd4d49b-65ed-457d-84a0-d9e716d85039.png" height="100"/>
<img src="https://user-images.githubusercontent.com/73862727/145950351-9924c1e8-64b2-43d6-84d4-255607cb585d.png"/>


![pypi](https://img.shields.io/badge/pypi-1.0.0-blue)
![docs](https://img.shields.io/badge/docs-latest-blue)
![license](https://img.shields.io/badge/license-Apache2.0-green)





</div>
 

## 简介

  LegoDNN（[文章](https://dl.acm.org/doi/abs/10.1145/3447993.3483249)）是一个针对模型缩放问题的轻量级、块粒度、可伸缩的解决方案。从原始DNN模型中抽取块，生成稀疏派生块，然后对这些块进行再训练。通过组合这些块，扩大原始模型的伸缩选项。并且在运行时，通过算法对块的选择进行了优化。
  本项目是一个对LegoDNN的基于PyTorch的实现。
 <div align="center" padding="10">
   <img src="https://user-images.githubusercontent.com/73862727/145767343-1cddf0f4-a9a9-48ef-8884-57688883e167.png" height=375/>
 </div>
 
  **主要特性**
- **模块化设计**

  本项目将LegoDNN的抽块、再训练等过程解耦成各个模块，通过组合不同的模块组件，用户可以更便捷的对自己的自定义模型Lego化。
  
- **块的自动化抽取**
    
    本项目实现了通用的块的抽取算法（[文章](https://dl.acm.org/doi/abs/10.1145/3447993.3483249)），对于图像分类、目标检测、语义分割、姿态估计、行为识别、异常检测等类型的模型均可以通过算法，自动找出其中的块用于再训练。

## 项目整体架构
<div align="center" padding="10">
 <img src="https://user-images.githubusercontent.com/73862727/145950613-fb0e96d5-3624-4de0-b371-ba0225bf56b3.png" height=375/>
</div>

**处理流程**主要分为离线阶段和在线阶段。

离线阶段：
- 原始模型通过block extrator抽取出模型中的原始块，然后将这些块通过`decendant block generator`生成稀疏派生块，然后用retrain模块将这些块根据原始数据在原始模型中产生的中间数据进行再训练。最后将原始块以及所有的再生块通过`block profiler`对块进行精度和内存的分析，生成分析文件。

在线阶段：
- 在线阶段首先对离线阶段产生的块进行延迟分析和估计，生成延迟评估文件，然后`scailing optimizer`根据延迟评估文件以及离线阶段生成的块的精度分析文件和内存分析文件在运行时根据算法选择最优的块交给`block swicher`进行切换。


**具体模块说明**
- blockmanager：在本框架中通过blockmanager融合了`block extrator`、`descendant block generator`、`block swicher`的功能，主要负责块的抽取，派生，更换，存储等，本项目已经通过AutoBlockManager实现针对多种模型自动对块的抽取,其算法原理详情见[文章]()。
- **offline**：在离线阶段对块进行再训练以提升其精度，并分析每个块的指标。
  - retrain：属于离线阶段对块的再训练。
  - Profile：属于离线阶段对块的大小、精度等信息进行分析统计。
- **online**：在线阶段主要是负责分析块与边缘设备相关的指标以及在线运行时针对特定的内存、精度限定对块进行热更新以进行优化。
  - LatencyProfile：属于在线阶段对块在边缘设备上进行延迟数据的分析。
  - RuntimeOptimizer：属于在线阶段运行期间根据特定内存大小对块进行热更新。

## 安装

在安装legodnn之前，请确保Pytorch已经成功安装在环境中，可以参考PyTorch的官方[文档](https://pytorch.org/)

```shell
pip install legodnn
```
简单的例子
```python
import os
os.environ["CUDA_VISIBLE_DEVICES"] = "1"
import sys
sys.path.insert(0, '../../')
sys.setrecursionlimit(100000)
import torch
from legodnn import BlockExtractor, BlockTrainer, ServerBlockProfiler, EdgeBlockProfiler, OptimalRuntime
from legodnn.gen_series_legodnn_models import gen_series_legodnn_models
from legodnn.utils.dl.common.env import set_random_seed
set_random_seed(0)
from legodnn.block_detection.model_topology_extraction import topology_extraction
from legodnn.presets.auto_block_manager import AutoBlockManager
from legodnn.presets.common_detection_manager_1204_new import CommonDetectionManager
from legodnn.model_manager.common_model_manager import CommonModelManager

from cv_task.datasets.image_classification.cifar_dataloader import CIFAR10Dataloader, CIFAR100Dataloader
from cv_task.image_classification.cifar.models import resnet18

if __name__ == '__main__':
    cv_task = 'image_classification'
    dataset_name = 'cifar100'
    model_name = 'resnet18'
    # compress_layer_max_ratio = 0.25
    compress_layer_max_ratio = 0.125
    device = 'cuda' 
    model_input_size = (1, 3, 32, 32)
    train_batch_size = 128
    test_batch_size = 128
    block_sparsity = [0.0, 0.3, 0.6, 0.8]
    root_path = os.path.join('results/legodnn', 
                             cv_task, model_name+'_'
                             +dataset_name + '_' 
                             + str(compress_layer_max_ratio).replace('.', '-'))
    compressed_blocks_dir_path = root_path + '/compressed'
    trained_blocks_dir_path = root_path + '/trained'
    descendant_models_dir_path = root_path + '/descendant'
    block_training_max_epoch = 20
    test_sample_num = 100
    
    teacher_model = resnet18(num_classes=100).to(device)
    teacher_model.load_state_dict(
            torch.load('cv_task_model/image_classification/cifar100/resnet18/2021-10-20/22-09-22/resnet18.pth')['net'])

    print('\033[1;36m-------------------------------->    BUILD LEGODNN GRAPH\033[0m')
    model_graph = topology_extraction(teacher_model, model_input_size, device=device)
    model_graph.print_ordered_node()
    # exit(0)
    print('\033[1;36m-------------------------------->    START BLOCK DETECTION\033[0m')
    detection_manager = CommonDetectionManager(model_graph, max_ratio=compress_layer_max_ratio) # resnet18
    detection_manager.detection_all_blocks()
    detection_manager.print_all_blocks()
    # exit(0)
    model_manager = CommonModelManager()
    block_manager = AutoBlockManager(block_sparsity, detection_manager, model_manager)
    
    print('\033[1;36m-------------------------------->    START BLOCK EXTRACTION\033[0m')
    block_extractor = BlockExtractor(teacher_model, block_manager, compressed_blocks_dir_path, model_input_size, device)
    block_extractor.extract_all_blocks()

    print('\033[1;36m-------------------------------->    START BLOCK TRAIN\033[0m')
    train_loader, test_loader = CIFAR100Dataloader()
    block_trainer = BlockTrainer(teacher_model, block_manager, model_manager, compressed_blocks_dir_path,
                                 trained_blocks_dir_path, block_training_max_epoch, train_loader, device=device)
    block_trainer.train_all_blocks()

    server_block_profiler = ServerBlockProfiler(teacher_model, block_manager, model_manager,
                                                trained_blocks_dir_path, test_loader, model_input_size, device)
    server_block_profiler.profile_all_blocks()


    edge_block_profiler = EdgeBlockProfiler(block_manager, model_manager, trained_blocks_dir_path, 
                                            test_sample_num, model_input_size, device)
    edge_block_profiler.profile_all_blocks()

    optimal_runtime = OptimalRuntime(trained_blocks_dir_path, model_input_size,
                                     block_manager, model_manager, device)
    gen_series_legodnn_models(deadline=100, model_size_search_range=[15,50], target_model_num=50, 
                              optimal_runtime=optimal_runtime, 
                              descendant_models_save_path=descendant_models_dir_path, 
                              device=device)

```

## docker

使用镜像

|树莓派4B|Jeston|
|----|----|
|`docker run -it lincbit/legodnn:raspberry4B-1.0`|`docker run -it lincbit/legodnn:jetsontx2-1.0`|


## 开源许可证

该项目采用 [Apache 2.0 开源许可证](LICENSE)。

## 更新日志

**1.0.0**版本已经在 2021.12.20 发布：

  基础功能实现
  
## 基准测试和支持的模型

  **图像分类**
  - [x] [VGG (ICLR'2015)](https://arxiv.org/abs/1409.1556)
  - [x] [InceptionV3 (CVPR'2016)](https://www.cv-foundation.org/openaccess/content_cvpr_2016/html/Szegedy_Rethinking_the_Inception_CVPR_2016_paper.html)
  - [x] [ResNet (CVPR'2016)](https://openaccess.thecvf.com/content_cvpr_2016/html/He_Deep_Residual_Learning_CVPR_2016_paper.html)
  - [x] [CBAM (ECCV'2018)](https://openaccess.thecvf.com/content_ECCV_2018/html/Sanghyun_Woo_Convolutional_Block_Attention_ECCV_2018_paper.html)
 
  **目标检测**
  - [x] [Fast R-CNN (NIPS'2015)](https://ieeexplore.ieee.org/abstract/document/7485869)
  - [x] [YOLOv3 (CVPR'2018)](https://arxiv.org/abs/1804.02767)
  - [x] [CenterNet (ICCV'2019)](https://openaccess.thecvf.com/content_ICCV_2019/html/Duan_CenterNet_Keypoint_Triplets_for_Object_Detection_ICCV_2019_paper.html)
  
  **语义分割**
  - [x] [FCN (CVPR'2015)](https://openaccess.thecvf.com/content_cvpr_2015/html/Long_Fully_Convolutional_Networks_2015_CVPR_paper.html)
  - [X] [SegNet (TPAMI'2017)](https://ieeexplore.ieee.org/abstract/document/7803544)
  - [x] [DeepLab v3 (ArXiv'2017)](https://arxiv.org/abs/1706.05587)
  - [x] [CCNet (ICCV'2021)](https://openaccess.thecvf.com/content_ICCV_2019/html/Huang_CCNet_Criss-Cross_Attention_for_Semantic_Segmentation_ICCV_2019_paper.html)
  
  **姿态估计**
  - [x] [DeepPose (CVPR'2014)](https://openaccess.thecvf.com/content_cvpr_2014/html/Toshev_DeepPose_Human_Pose_2014_CVPR_paper.html)
  - [x] [CPN (CVPR'2018)](https://openaccess.thecvf.com/content_cvpr_2018/html/Chen_Cascaded_Pyramid_Network_CVPR_2018_paper.html)
  - [x] [SimpleBaselines (ECCV'2018)](https://openaccess.thecvf.com/content_ECCV_2018/html/Bin_Xiao_Simple_Baselines_for_ECCV_2018_paper.html)
    
  **行为识别**
  - [x] [Two-STeam CNN (NIPS'2014)](https://arxiv.org/abs/1406.2199)
  - [x] [TSN (ECCV'2016)](https://link.springer.com/chapter/10.1007/978-3-319-46484-8_2)
  - [x] [TRN (ECCV'2018)](https://openaccess.thecvf.com/content_ECCV_2018/html/Bolei_Zhou_Temporal_Relational_Reasoning_ECCV_2018_paper.html)
  
  **异常检测**
  - [x] [GANomaly (ACCV'2018)](https://link.springer.com/chapter/10.1007/978-3-030-20893-6_39)
  - [x] [GPND (NIPS'2018)](https://arxiv.org/abs/1807.02588)
  - [x] [OGNet (CVPR'2020)](https://openaccess.thecvf.com/content_CVPR_2020/html/Zaheer_Old_Is_Gold_Redefining_the_Adversarially_Learned_One-Class_Classifier_Training_CVPR_2020_paper.html)
  - [x] [Self-Training (CVPR'2020)](Self-trainedDeepOrdinalRegressionforEnd-to-EndVideoAnomalyDetection)
  
