# yolov3_tiny + movidius + Ros melodic18.04 + upcore + realsense


# 一.安装环境

## 1.安装realsense
[参考realsense](https://github.com/IntelRealSense/realsense-ros)
#### Step1.
`sudo apt-get install ros-melodic-realsense2-camera`

## 2.安装Openvino
[参考openvino](https://software.intel.com/en-us/articles/get-started-with-neural-compute-stick)
#### Step1.下载openvino安装包
最好直接下载我使用的版本[2020.4.287](https://pan.baidu.com/s/1X1k8_Hwbyhu7Na1Nx0-WXg) 提取码：jhbo 
#### Step2.解压安装包
`tar xvf l_openvino_toolkit_p_2020.4.287.tgz`
#### Step3.进入安装包
`cd l_openvino_toolkit_p_2020.4.287`
#### Step4.安装依赖
`sudo -E ./install_openvino_dependencies.sh`
#### Step5.运行安装GUI(建议添加sudo，添加sudo命令安装到根目录下，不加sudo命令默认安装到home目录下)
`sudo ./install_GUI.sh`
#### Step6.安装完成之后添加source到bashrc下
```
echo "source /opt/intel/openvino/bin/setupvars.sh" >> ~/.bashrc
source ~/.bashrc
```

## 3.安装movidius(NCS2)依赖
`cd ~/intel/openvino/install_dependencies`  
`sudo ./install_NCS_udev_rules.sh`

## 4.配置Model Optimizer的依赖（此项较久upcore需要20分钟）
[参考OpenvinoToolkit](https://docs.openvinotoolkit.org/2019_R2/_docs_install_guides_installing_openvino_linux.html#install-external-dependencies)    
`cd /opt/intel/openvino/deployment_tools/model_optimizer/install_prerequisites`
`sudo ./install_prerequisites.sh`

## 5.检查安装情况
#### Step1.命令行进入python3
`python3`
#### Step2.运行以下命令看是否有报错
```
import cv2
import openvino
import tensorflow
```
#### Step3.报错(如果import没有报错则可以跳过Step3)
如果`import tensorflow`报错core dumped：
![image.png](https://i.loli.net/2020/10/31/d8ALNvUqgIHP1ci.png)
这是因为upcore的CPU版本太老，不支持tensorflow1.6.0以后的AVX指令集

#### Step3.解决方案
重装tensorflow 1.5.0版本，安装完成后重新运行Step2的指令，确保能够成功import cv2 openvino tensorflow    
```
sudo pip3 uninstall tensorflow
sudo pip3 install tensorflow==1.5.0
```



# 二.测试识别demo(参考)[https://ptitdeveloper.com/blog/openvino-toi-uu-hoa-hieu-suat-model-darknet-yolov3/]
>  分为两部分，第一部分演示如何运行示例代码，第二部分演示如何实现自己的weights文件部署
## 1.运行示例代码(包含转换好的xml文件可以直接运行测试)

#### Step1.下载示例代码
`git clone https://github.com/GitZzw/demo.git`

#### Step2.修改openvino安装路径下的yolo_v3_tiny.json文件
将两个.json文件中的classes改为1(demo中只有一个识别目标)
两个文件路径分别为
`/opt/intel/openvino/deployment_tools/model_optimizer/extensions/front/tf/yolo_v3_tiny.json` 和 `/opt/intel/openvino_2020.4.287/deployment_tools/model_optimizer/extensions/front/tf/yolo_v3_tiny.json`

#### Step3.运行demo代码
图像测试
`python3 object_detection_demo_yolov3_async.py -i demo.jpg -m /demo_weights/frozen_darknet_yolov3_model.xml -d MYRIAD`
视频测试
`python3 object_detection_demo_yolov3_async.py -i demo.mp4 -m /demo_weights/frozen_darknet_yolov3_model.xml -d MYRIAD`

## 2.自己的weights文件转换实现
#### Step1.下载转换工具
 `git clone https://github.com/GitZzw/demo.git`

#### #### Step2.修改openvino安装路径下的yolo_v3_tiny.json文件
将两个.json文件中的classes改为你训练时识别目标数量，否则会报错
> error:cannot reshape array of size 18900 into shape (1,18,30,30)

两个文件路径分别为
`/opt/intel/openvino/deployment_tools/model_optimizer/extensions/front/tf/yolo_v3_tiny.json` 和 `/opt/intel/openvino_2020.4.287/deployment_tools/model_optimizer/extensions/front/tf/yolo_v3_tiny.json`

#### Step3.将你的weights文件和voc.names文件放到demo/tensorflow-yolo-v3/文件夹下

#### Step4.将weigths文件转换为tensorflow模型
打开convert_weights_pb.py文件可以修改参数，确认路径名称和size大小(默认480)
> 注:size大小应该修改为与训练weights的cfg文件的width和heights一致

```
cd tensorflow-yolo-v3
python3 convert_weights_pb.py

```
执行将会在文件目录下生成frozen_darknet_yolov3_model.pb文件

#### Step5.将tensorflow模型转换为IR模型
记得将.pb路径设为自己的
```
python3 /opt/intel/openvino/deployment_tools/model_optimizer/mo_tf.py --input_model ~/demo/tensorflow-yolo-v3/frozen_darknet_yolov3_model.pb --tensorflow_use_custom_operations_config /opt/intel/openvino/deployment_tools/model_optimizer/extensions/front/tf/yolo_v3_tiny.json --data_type FP16 --batch 1 --reverse_input_channels
```
执行将会在目录下生成xml，bin，mapping三个文件

#### Step6.运行object_detection_demo_yolov3_async.py即可
