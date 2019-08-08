## Ubuntu Caffe OpenCV 安装部署

### 前置条件

- Ubuntu 16.04
- OpenCV 3.3.1
- Cmake-3.15.1
- Python 2.7.12

### 编译 OpenCV

#### 安装依赖包

安装 Python 以及 OpenCV 安装所需的其他库和包

```shell
sudo apt-get install cmake
sudo apt-get install build-essential \
             libgtk2.0-dev \
             libavcodec-dev \
             libavformat-dev \
             libjpeg.dev \
             libtiff4.dev \
             libswscale-dev \
             libjasper-dev
```

#### 开始安装

```shell
# 这里的包版本是按照项目Python包的具体版本安装的，不是必须这样的。只需要满足条件即可。
pip install numpy==1.11.0 scipy==0.17.0 matplotlib==2.1.1 scikit-image==0.13.1  ipython==5.5 scikit-learn dlib

# 修改编译安装下载依赖
mkdir /opt/opencv_3rdparty
cd /opt/opencv_3rdparty
wget https://raw.githubusercontent.com/opencv/opencv_3rdparty/ippicv/master_20180723/ippicv/ippicv_2019_lnx_intel64_general_20180723.tgz

wget https://github.com/opencv/opencv/archive/opencv-3.3.1.zip
unzip opencv-3.3.1.zip
cd opencv-3.3.1
sed -i 's/https:\/\/raw.githubusercontent.com\/opencv\/opencv_3rdparty\/\${IPPICV_COMMIT}\/ippicv\//file:\/\/\/opt\/opencv_3rdparty\//g' ./opencv-3.3.1/3rdparty/ippicv/ippicv.cmake

mkdir build
cd build

# 注意修改对应路径地址 python 路径，有可能是虚拟环境的
cmake -D CMAKE_BUILD_TYPE=Release \
      -D WITH_FFMPEG=ON \
      -D CMAKE_INSTALL_PREFIX=/usr/local \
      -D OPENCV_PYTHON2_INSTALL_PATH=/usr/local/lib/python2.7/dist-packages/ \
      -D PYTHON_EXECUTABLE=/usr/bin/python \
      -D PYTHON_INCLUDE_DIR=/usr/include/python2.7 \
      -D PYTHON_INCLUDE_DIR2=/usr/include/x86_64-linux-gnu/python2.7 \
      -D PYTHON_LIBRARY=/usr/lib/x86_64-linux-gnu/libpython2.7.so \
      -D PYTHON_INCLUDE_DIRS=/usr/local/lib/python2.7/dist-packages/numpy/core/include/ ..

make -j$(nproc)
make install
```

#### 测试 OpenCV Python 模块

```python
import cv2
print cv2.__version__
3.3.1
```

### 编译 Caffe

#### 通用依赖

```shell
sudo apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        cmake \
        git \
        wget \
        libatlas-base-dev \
        libboost-all-dev \
        libgflags-dev \
        libgoogle-glog-dev \
        libhdf5-serial-dev \
        libleveldb-dev \
        liblmdb-dev \
        libopencv-dev \
        libprotobuf-dev \
        libsnappy-dev \
        protobuf-compiler \
        python-dev \
        python-numpy \
        python-pip \
        python-setuptools \
        python-scipy
```

#### 编译

```shell
# 下载源码 解压

# 安装编译依赖的Python，在caffe 源码包Python目录下，尽量安装项目依赖的包的具体版本安装，只要满足python目录下包版本需求即可。

cd python && for req in $(cat requirements.txt); do pip install $req; done && cd ..
mkdir build && cd build

# 仅CPU加速
cmake -D CPU_ONLY=1 ..
make -j$(nproc)

# 设置Caffe Python包环境变量

# Caffe 源码包路径视情况而定
echo "export CAFFE_ROOT=/opt/download/caffe-1.0" >> ~/.bashrc
echo "export PYCAFFE_ROOT=$CAFFE_ROOT/python" >> ~/.bashrc
echo "export PYTHONPATH=$PYCAFFE_ROOT:$PYTHONPATH" >> ~/.bashrc
echo "export PATH=$CAFFE_ROOT/build/tools:$PYCAFFE_ROOT:$PATH" >> ~/.bashrc
echo "$CAFFE_ROOT/build/lib" >> /etc/ld.so.conf.d/caffe.conf && ldconfig

source ~/.bashrc

# 测试Caffe 包是否可用
>>import caffe
>>print caffe.__version__
1.0.0
```
