# Centos Caffe 部署

## 前置条件

- Centos 7
- OpenCV 3.3.1
- Cmake-3.15.1
- Python 2.7.5
- virtualenv 虚拟环境

## 编译 OpenCV

### 安装依赖包

```shell
sudo yum -y install epel-release \
            git wget unzip gcc gcc-c++ cmake cmake3 \
            qt5-qtbase-devel \
            python python-devel python-pip \
            gtk2-devel \
            libpng-devel \
            jasper-devel \
            openexr-devel \
            libwebp-devel \
            libjpeg-turbo-devel \
            freeglut-devel mesa-libGL mesa-libGL-devel \
            libtiff-devel \
            libdc1394-devel \
            tbb-devel eigen3-devel \
            libv4l-devel \
            gstreamer-plugins-base-devel
```

### 安装 ffmpeg

```shell
yum install -y epel-release
rpm -v --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro
rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm
yum install -y ffmpeg ffmpeg-devel
ffmpeg -version
```

### 创建虚拟环境

```shell
pip install virtualenv virtualenvwrapper -i https://pypi.doubanio.com/simple/

# 这里注意使用Python2和Python3
echo "export WORKON_HOME=/root/.virtualenvs" >> ~/.bashrc
echo "export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python" >> ~/.bashrc
echo "source /usr/bin/virtualenvwrapper.sh" >> ~/.bashrc
source ~/.bashrc
mkvirtualenv OpenCV-3.3.1-py2 -p python

# 下次进入的时候使用
workon OpenCV-3.3.1-py2
# 阿里云服务器默认选用阿里源，可取消 -i 参数
pip install cmake -i https://pypi.doubanio.com/simple/
pip install numpy==1.11.0 scipy==0.17.0 matplotlib==2.1.1 scikit-image==0.13.1  ipython==5.5 scikit-learn dlib -i https://pypi.doubanio.com/simple/
```

### OpenCV 编译安装

OpenCV 安装版本选择是根据 anaconda 安装 Caffe 时，依赖安装的是 OpenCV-3.3.1。

```shell
# 下载对应版本的OpenCV，并解压

mkdir /opt/opencv_3rdparty
cd /opt/opencv_3rdparty

# 下载编译OpenCV时需要的依赖

sed -i 's/https:\/\/raw.githubusercontent.com\/opencv\/opencv_3rdparty\/\${IPPICV_COMMIT}\/ippicv\//file:\/\/\/opt\/opencv_3rdparty\//g' ./opencv-3.3.1/3rdparty/ippicv/ippicv.cmake

export ENV_OPENCV_PY=/root/.virtualenvs/OpenCV-3.3.1-py2
cd opencv-3.3.1 && mkdir build && cd build
# CMAKE_INSTALL_PREFIX 安装路径，这个一般指定Python模块安装路径的lib上层。当然还可以指定别的路径，注意加载环境变量。在虚拟环境中，直接是家目录。该包生成的Python模块是标准的，可以直接指定Python包安装路径。如果安装错误也没关系，可以使用软链接方式，具体方式请自行网上查找。
cmake -D CMAKE_BUILD_TYPE=Release \
      -D WITH_FFMPEG=ON \
      -D CMAKE_INSTALL_PREFIX=$ENV_OPENCV_PY \
      -D PYTHON_EXECUTABLE=$ENV_OPENCV_PY/bin/python \
      -D PYTHON_INCLUDE_DIR=$ENV_OPENCV_PY/include/python2.7 \
      -D PYTHON_LIBRARY=$ENV_OPENCV_PY/lib/python2.7/config/libpython2.7.so \
      -D PYTHON_NUMPY_INCLUDE_DIRS=$ENV_OPENCV_PY/lib/python2.7/site-packages/numpy/core/include/ ..

make -j$(nproc)
make install

# 测试生成的Python 模块
>>import cv2
>>import os.path.dirname(os.__file__)
```

## 编译 Caffe

### 安装依赖

按照官网要求，安装 boost-devel 开发包时会安装依赖包 boost-1.53，该版本过低，暂不安装。

### 编译安装 Boost-1.58 （Caffe 编译需要依赖这个 C++库，版本 1.55 以上）

由于 Centos yum 安装是 Boost-1.53 版本，不符合要求。需要编译安装。

```shell
yum -y install gcc-c++ python-devel bzip2-devel zlib-devel
wget https://nchc.dl.sourceforge.net/project/boost/boost/1.58.0/boost_1_58_0.zip
unzip boost_1_58_0.zip
cd boost_1_58_0 && ./bootstrap.sh --prefix=/usr/local/boost
./b2
cd tools/build && ./bootstrap.sh
./b2 install --prefix=/usr/local/boost
```

```shell
yum install -y protobuf-devel leveldb-devel snappy-devel hdf5-devel

pip install protobuf -i https://pypi.doubanio.com/simple/

# glog 请注意，glog不能使用最新的gflags版本（2.1）进行编译，因此在解决之前，您需要先使用glog进行构建。
wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/google-glog/glog-0.3.3.tar.gz
tar zxvf glog-0.3.3.tar.gz
cd glog-0.3.3
./configure
make -j$(nproc) && make install
# gflags
wget https://github.com/schuhschuh/gflags/archive/master.zip
unzip master.zip
cd gflags-master
mkdir build && cd build
export CXXFLAGS="-fPIC" && cmake .. && make VERBOSE=1
make -j$(nproc) && make install
# lmdb
git clone https://github.com/LMDB/lmdb
cd lmdb/libraries/liblmdb
make -j$(nproc) && make install

# 为获得更好的CPU 性能
yum install -y atlas-devel

# python 包构建
yum install -y python-devel

# 使用cmake 编译
git clone https://github.com/BVLC/caffe.git
cd caffe
# 修改编译配置
cp Makefile.config.example Makefile.config
# CPU_ONLY=1 仅仅 CPU 模式
# BLAS := atlas  CPU更好性能
# BLAS_INCLUDE := /usr/include/atlas
# BLAS_LIB := /usr/lib64/atlas
# Python库相关路径配置
# PYTHON_INCLUDE，PYTHON_LIB

mkdir build && cd build

# 目前只是使用cmake编译，make编译尚未验证。cmake编译时需要atlas相关的动态库，不然编译时会报错找不到Atlas_CBLAS_LIBRARY,Atlas_BLAS_LIBRARY,Atlas_LAPACK_LIBRARY 库。cmake 编译的Makefile在Caffe源码目录下$CAFFE_ROOT/cmake/Modules/FindAtlas.cmake
ln -sv /usr/lib64/atlas/libsatlas.so.3.10 /usr/lib64/atlas/libcblas.so
ln -sv /usr/lib64/atlas/libsatlas.so.3.10 /usr/lib64/atlas/libatlas.so
ln -sv /usr/lib64/atlas/libsatlas.so.3.10 /usr/lib64/atlas/liblapack.so

cmake ..
make -j$(nproc) all

# 这里编译完成之后，会有相关的设置环境变量提示，注意保存并添加环境变量。
make -j$(nproc) install
make -j$(nproc) runtest

# 设置环境变量到~/.bashrc，注意是虚拟环境的家目录。
export CAFFE_ROOT=/opt/caffe-1.0
export PYCAFFE_ROOT=$CAFFE_ROOT/build/install/python
export PYTHONPATH=$PYCAFFE_ROOT:$PYTHONPATH
export PATH=$CAFFE_ROOT/build/tools:$PYCAFFE_ROOT:$CAFFE_ROOT/build/install/lib:/opt/boost_1_58_0/stage/lib:$WORKON_HOME/OpenCV-3.3.1-py2/lib:/usr/local/lib:/usr/lib64/atlas:$CAFFE_ROOT/build/install/python/caffe/_caffe.so:$PATH

source ~/.bashrc

echo /opt/caffe-1.0/build/lib > /etc/ld.so.conf.d/caffe.conf
ldconfig

# 测试caffe
>>import caffe
>>import os
>>print os.path.dirname(caffe.__file__)
```
