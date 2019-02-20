# Centos Caffe 部署

## 前置条件

- Centos 7
- OpenCV 3.3.1
- Cmake-3.15.1
- Python 2.7.5

## 编译 OpenCV

### 安装依赖包

```shell
sudo yum -y install epel-release \
            git wget unzip gcc gcc-c++ cmake3 \
            qt5-qtbase-devel \
            python python-devel python-pip \
            python-devel numpy \
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
mkvirtualenv OpenCV-3.3.1-py2 -p python
workon OpenCV-3.3.1-py2
# 阿里云服务器默认选用阿里源，可取消 -i 参数
pip install cmake -i https://pypi.doubanio.com/simple/
pip install numpy==1.11.0 scipy==0.17.0 matplotlib==2.1.1 scikit-image==0.13.1  ipython==5.5 scikit-learn dlib -i https://pypi.doubanio.com/simple/
```

### OpenCV 编译安装

```shell
# 下载对应版本的OpenCV，并解压
export ENV_OPENCV_PY=/root/.virtualenvs/OpenCV-3.3.1-py2
cd opencv-3.3.1 && mkdir build && cd build
# CMAKE_INSTALL_PREFIX 安装路径，这个一般指定Python模块安装路径的lib上层。在虚拟环境中，直接是家目录。如果安装错误也不要紧，可以使用软链接方式，具体方式请自行网上查找。
cmake -D CMAKE_BUILD_TYPE=Release \
      -D WITH_FFMPEG=ON \
      -D CMAKE_INSTALL_PREFIX=$ENV_OPENCV_PY \
      -D PYTHON_EXECUTABLE=$ENV_OPENCV_PY/bin/python \
      -D PYTHON_INCLUDE_DIR=$ENV_OPENCV_PY/include/python2.7 \
      -D PYTHON_LIBRARY=$ENV_OPENCV_PY/lib/python2.7/config/libpython2.7.so \
      -D PYTHON_NUMPY_INCLUDE_DIRS=$ENV_OPENCV_PY/lib/python2.7/site-packages/numpy/core/include/ ..

make -j$(nproc)
make install
```

## 编译Caffe

### 安装依赖

```shell
yum install -y protobuf-devel leveldb-devel snappy-devel hdf5-devel

```