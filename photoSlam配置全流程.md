# photo slam配置全流程

项目地址：[HuajianUP/Photo-SLAM： [CVPR 2024\] Photo-SLAM：单目、立体和 RGB-D 相机的实时同步定位和逼真映射 (github.com)](https://github.com/HuajianUP/Photo-SLAM?tab=readme-ov-file)

本文是记录了2023CVPR photo slam系统环境配置，编辑时间为**20240817**。

photo slam是一种带有超基元图的新型 SLAM 框架 ，利用显性几何特征（ORB描述子）进行定位，并学习隐性光度特征来表示观察环境的纹理信息（3D 高 斯 泼 溅），图像处理方面引入了基于高斯金字塔的训练方法，以逐步学习多层次特征，从而提高逼真映射性能。支持单目、立体和 RGB-D 数据，不需要依赖密集的深度信息即可实现高质量的映射。

整个slam系统脱离ros，利用ORB-SLAM3、 3D高斯泼溅和LibTorch 框架 和 CUDA 实现，并且全部由C++编写，可使用 Jet- son AGX Orin 等嵌入式平台实时运行。

文章中提到使用固定学习率和λ = 0.2的情况下，利用随机梯度下降算法对逼真映射进行了优化。理论上越大分辨率的图像需要的层数应该越多，考虑到测试数据集的图像分辨率，高斯金字塔的级别默认设置为 3， 即 n = 2。

在文章中与ORB-SLAM3、BundleFusion、DROID-SLAM、Nice-SLAM、Orbeez-SLAM、ESLAM、CoSLAM、Point-SLAM和Go-SLAM等SLAM系统进行比较。在多个知名的RGB-D数据集上进行了单目和RGB-D传感器类型的测试，包括Replica数据集和TUM RGB-D数据集。对于立体视觉测试，使用了EuRoC MAV数据集。还用ZED 2立体相机收集了室外场景进行额外评估。

性能评估方面，使用绝对轨迹误差（ATE）及其均方根误差（RMSE）和标准差来估计定位的准确性。采用PSNR、SSIM和LPIPS等指标来分析光度映射的性能，并报告了跟踪FPS、渲染FPS和GPU内存使用情况。

文章中提到他们的部署环境有：NVIDIA RTX 4090 GPU、Intel Core i9-13900K CPU和64GB RAM的桌面计算机；NVIDIA RTX 3080ti 16 GB laptop、Intel Core i9-12900HX和32 GB RAM的笔记本电脑；Jetson AGX Orin开发者套件。说实话大组就是有钱，本人的部署环境是NVIDIA RTX 4070ti 12 GB、Intel Core i5-13600KF和32 GB RAM的台式电脑（一看就是臭打游戏的）。

整个环境配置流程还是很麻烦的，下面是官网给出的一些测试通过的环境。本人选择使用Ubuntu 20.04 LTS作为系统环境，所有要求版本全部按照官网要求配置（之前使用之前的环境直接配给我配麻了，按照官方给的环境配一遍成功）。

<table>
    <thead>
        <tr>
            <th>依赖项</th>
            <th colspan=3>测试环境</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>操作系统</td>
            <td>Ubuntu 20.04 LTS</td>
            <td>Ubuntu 22.04 LTS</td>
            <td>Jetpack 5.1.2</td>
        </tr>
        <tr>
            <td>gcc</td>
            <td>10.5.0</td>
            <td>11.4.0</td>
            <td>9.4.0</td>
        </tr>
        <tr>
            <td>cmake</td>
            <td>3.27.5</td>
            <td>3.22.1</td>
            <td>3.26.4</td>
        </tr>
        <tr>
            <td><a href="https://developer.nvidia.com/cuda-toolkit-archive">CUDA</a> </td>
            <td>11.8</td>
            <td>11.8</td>
            <td>11.4</td>
        </tr>
        <tr>
            <td><a href="https://developer.nvidia.com/rdp/cudnn-archive">cuDNN</a> </td>
            <td>8.9.3</td>
            <td>8.7.0</td>
            <td>8.6.0</td>
        </tr>
        <tr>
            <td><a href="https://opencv.org/releases">OpenCV</a> (使用 opencv_contrib 和 CUDA 构建)</td>
            <td>4.7.0</td>
            <td>4.8.0</td>
            <td>4.7.0</td>
        </tr>
        <tr>
            <td><a href="https://pytorch.org/get-started/locally">LibTorch</a> <2.1.2 </td>
            <td colspan=2>cxx11-abi-shared-with-deps-2.0.1+cu118</td>
            <td>2.0.0+nv23.05-cp38-linux_aarch64</td>
        </tr>
        <tr>
            <td colspan=4>(可选) <a href="https://github.com/IntelRealSense/librealsense">Intel® RealSense™ SDK 2.0</a> </td>
        </tr>
        <tr>
            <td colspan=4>(备注) 我们使用的 Jetson AGX Orin 开发套件配备了 64GB 内存，电源模式设置为 MAXN。</td>
        </tr>
    </tbody>
</table>
这里是在从新装的20.04 系统上配置的，首先需要一些基本环境。

## 小鱼一键换源，挂梯子，装一些基本工具

```bash
wget http://fishros.com/install -O fishros && . fishros
```

## 基本依赖

```bash
sudo apt install libeigen3-dev libboost-all-dev libjsoncpp-dev libopengl-dev mesa-utils libglfw3-dev libglm-dev
```

## gcc11.40

在Ubuntu系统上安装GCC 11.4.0，可以通过以下步骤完成：

1. **更新系统的包列表：**

   ```
   sudo apt update
   ```

2. **添加GCC官方PPA源：** 为了安装指定版本的GCC，我们需要添加适合的PPA源。可以使用`ppa:ubuntu-toolchain-r/test`，它包含了多个GCC版本。

   ```
   sudo add-apt-repository ppa:ubuntu-toolchain-r/test
   sudo apt update
   ```

3. **安装GCC 11和G++ 11：** 现在我们可以安装GCC 11和G++ 11：

   ```
   sudo apt install gcc-11 g++-11
   ```

4. **验证安装版本：** 安装完成后，可以检查已安装的GCC版本：

   ```
   gcc-11 --version
   ```

   输出应类似于：

   ```
   gcc (Ubuntu 11.4.0-1ubuntu1~20.04) 11.4.0
   ```

5. **设置默认版本（可选）：** 如果系统中安装了多个GCC版本，并希望将GCC 11设置为默认版本，可以执行以下命令：

   ```
   sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 100
   sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-11 100
   ```

   如果需要在多个版本之间切换，可以使用：

   ```
   sudo update-alternatives --config gcc
   sudo update-alternatives --config g++
   ```

## cmake3.22.1

1. **更新系统包列表：**

   ```
   sudo apt update
   ```

2. **安装必要的依赖项：**

   ```
   sudo apt install build-essential libssl-dev
   ```

3. **下载CMake 3.22.1的源代码：**

   ```
   wget https://github.com/Kitware/CMake/releases/download/v3.22.1/cmake-3.22.1.tar.gz
   ```

4. **解压下载的压缩包：**

   ```
   tar -zxvf cmake-3.22.1.tar.gz
   cd cmake-3.22.1
   ```

5. **编译和安装CMake：**

   ```
   ./bootstrap
   make
   sudo make install
   ```

   这个过程可能需要一些时间，具体取决于您的系统性能。

6. **验证CMake版本：** 安装完成后，可以检查CMake的版本以确保安装正确：

   ```
   cmake --version
   ```

   您应该看到类似以下的输出：

   ```
   cmake version 3.22.1
   ```

## cuda11.8

1. 执行[安装前操作](https://docs.nvidia.com/cuda/archive/11.8.0/cuda-installation-guide-linux/index.html#pre-installation-actions)。

- 确认系统具有支持 CUDA 的 GPU。
- 确认系统运行的是受支持的 Linux 版本。
- 确认系统已安装 gcc。
- 确认系统已安装正确的内核头文件和开发包。
- 下载英伟达™（NVIDIA®）CUDA 工具包。
- 处理相互冲突的安装方法。

2. **删除过期签名密钥：**

```
sudo apt-key del 7fa2af80
```

CUDA 代码库的新 GPG 公钥是 [3bf863cc](https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/3bf863cc.pub) 。 必须使用 cuda-keyring 软件包或手动在系统上注册；apt-key 命令已过时，不推荐使用。

选择安装方法可以本地 repo 或网络 repo，国内的同学建议还是本地安装吧。

3. Ubuntu 的本地版本库安装

**在文件系统上安装本地版本库：**

这里我们应该先去[官网](https://developer.nvidia.com/cuda-toolkit-archive)上下载好cuda11.8的安装包，选好版本按照指示就可以下载：

![1724031064366](/media/asus/Samsung USB/ubuntu/桌面/photoSlam配置全流程.assets/1724031064366.jpg)

下面是给出的本地安装步骤，已经注释了每步干啥，比较耗时的部分是wget的安装包，建议提前下载好。

```bash
# 下载并设置 CUDA 的优先级，确保使用 NVIDIA 提供的软件包而不是系统默认的版本
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/cuda-ubuntu1804.pin
sudo mv cuda-ubuntu1804.pin /etc/apt/preferences.d/cuda-repository-pin-600

# 下载 CUDA 11.8 的本地安装包
wget https://developer.download.nvidia.com/compute/cuda/11.8.0/local_installers/cuda-repo-ubuntu1804-11-8-local_11.8.0-520.61.05-1_amd64.deb

# 安装下载的 CUDA 11.8 本地安装包
sudo dpkg -i cuda-repo-ubuntu1804-11-8-local_11.8.0-520.61.05-1_amd64.deb

# 注册永久公开 GPG 密钥，以确保软件包的完整性和真实性
sudo cp /var/cuda-repo-ubuntu1804-11-8-local/cuda-*-keyring.gpg /usr/share/keyrings/

# 更新软件包索引，获取 CUDA 相关的软件包信息
sudo apt-get update 

# 安装 CUDA 工具包及相关组件
sudo apt-get -y install cuda

```

4. Ubuntu 的网络版本库安装

同样去[官网](https://developer.nvidia.com/cuda-toolkit-archive)上下载好cuda11.8的安装包，选好版本按照指示，选择network就行。

![1724031502118](/media/asus/Samsung USB/ubuntu/桌面/photoSlam配置全流程.assets/1724031502118.jpg)

```bash
# 下载并安装 CUDA keyring 包，用于验证来自 NVIDIA 的软件包
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/cuda-keyring_1.0-1_all.deb

# 安装 keyring 包，确保 CUDA 软件包的安全性
sudo dpkg -i cuda-keyring_1.0-1_all.deb

# 更新软件包索引，以便 apt 能获取最新的软件包信息
sudo apt-get update

# 安装 CUDA 工具包及相关组件
sudo apt-get -y install cuda

```

5. 重启系统

完成安装后，重启系统以确保 CUDA 驱动和库加载正确

```bash
sudo reboot
```

6. 安装后操作

重启后，验证 CUDA 是否正确安装并配置：

检查 CUDA 版本：

```bash
nvcc --version
```

显示下面的内容就证明安装成功：

```bash
asus@ASUS:~$ nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2022 NVIDIA Corporation
Built on Wed_Sep_21_10:33:58_PDT_2022
Cuda compilation tools, release 11.8, V11.8.89
Build cuda_11.8.r11.8/compiler.31833905_0
```

7. 可选：安装 NVIDIA GDS（GPUDirect Storage），**对于photoslam不需要**。
    如果需要 GPUDirect Storage 支持，可以安装 nvidia-gds：

```bash
sudo apt-get install nvidia-gds
```

8. **设置环境变量：** 确保`/usr/local/cuda-11.8/bin`和`/usr/local/cuda-11.8/lib64`被正确添加到`PATH`和`LD_LIBRARY_PATH`中。

编辑 ~/.bashrc 文件：

```bash
sudo nano ~/.bashrc
```

在文件末尾添加以下内容：

```bash
export PATH=/usr/local/cuda-11.8/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-11.8/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
```

保存并关闭文件（按 Ctrl + X 然后按 Y 和 Enter）。

使更改生效：

```bash
source ~/.bashrc
```

9. 验证环境变量
通过以下命令验证环境变量是否设置正确：

```bash
echo $PATH | grep /usr/local/cuda-11.8/bin
echo $LD_LIBRARY_PATH | grep /usr/local/cuda-11.8/lib64
```

如果这两个命令返回包含 /usr/local/cuda-11.8/bin 和 /usr/local/cuda-11.8/lib64 的路径，说明环境变量设置正确。

## 安装显卡驱动
1. 我的电脑默认是装了显卡驱动，使用下面命令查看是否已安装 NVIDIA 驱动程序以及安装的版本：

```bash
nvidia-smi
```

![2024-08-19 10-01-31 的屏幕截图](/media/asus/Samsung USB/ubuntu/桌面/photoSlam配置全流程.assets/2024-08-19 10-01-31 的屏幕截图.png)

上面显示我的驱动版本是560.28.03 ，最高支持CUDA版本是  12.6。如果返回 NVIDIA 驱动的详细信息，说明系统已安装 NVIDIA 驱动程序。如果返回错误或显示 command not found，说明未安装或安装不正确。

2. 先用下面命令查看设备支持的NVIDIA驱动程序

```bash
ubuntu-drivers devices
```

3. 自动安装推荐的驱动程序

```bash
sudo ubuntu-drivers autoinstall
```

这个命令将自动检测你的 GPU 并安装适合的驱动程序   。

4. 如果你需要安装特定版本的驱动程序，可以使用以下命令：

```bash
sudo apt-get install nvidia-driver-<version>
```

**这里的version和查看设备支持的NVIDIA驱动程序版本对应**

5. 重启系统

驱动程序安装完成后，重启系统以使更改生效。

```bash
sudo reboot
```

6. 验证安装

重启后，再次运行 nvidia-smi 命令，检查驱动程序是否安装成功，并确认 GPU 已被正确识别和使用。如果显示了显卡信息、驱动版本和 CUDA 版本，说明驱动安装成功。

## cuDNN8.7.0
安装的目标是三个文件：

```bash
sudo dpkg -i libcudnn8_<version>-<architecture>.deb
sudo dpkg -i libcudnn8-dev_<version>-<architecture>.deb
sudo dpkg -i libcudnn8-samples_<version>-<architecture>.deb  ## 可选
```

但是英伟达将其整合为一个包，例如：

```bash
cudnn-local-repo-ubuntu2004-8.9.3.82_1.0-1_amd64.deb
```

1. 访问 [NVIDIA cuDNN 下载页面](https://developer.nvidia.com/cudnn)，在[这里](https://developer.nvidia.com/cudnn-archive)找到对应的历史版本，登录或注册 NVIDIA 开发者账号才能正常下载。

在下载页面选择适合您系统的 cuDNN 版本和目标 CUDA 版本，我选择8.9.3的版本。

![1724034114332](/media/asus/Samsung USB/ubuntu/桌面/photoSlam配置全流程.assets/1724034114332.jpg)

或者使用下面命令下载：

```bash
wget https://developer.nvidia.com/downloads/compute/cudnn/secure/8.9.3/local_installers/11.x/cudnn-local-repo-ubuntu2004-8.9.3.28_1.0-1_amd64.deb/
```

2. 然后使用下面命令安装

```bash
sudo dpkg -i cudnn-local-repo-ubuntu2004-8.9.3.82_1.0-1_amd64.deb
```
3. 配置密匙

```bash
sudo cp /var/cudnn-local-repo-ubuntu2004-8.9.3/cudnn-8.9.3-keyring.gpg /usr/share/keyrings/
```
4. 更新并安装

```bash
sudo apt-get update
sudo apt-get -y install cudnn
```

如果下载了其他版本的cudnn，这时候应该安装libcudnn8XXX  ，libcudnn8-devXXX ， libcudnn8-samples要改成/var/cudnn-local-repo-ubuntu2004xxxx/文件夹下对应需要安装的版本。

```bash
sudo apt-get -y install libcudnn8XXX libcudnn8-devXXX libcudnn8XXX-samples
```

5. 验证是否安装成功

```bash
cat /usr/include/cudnn_version.h | grep CUDNN_MAJOR -A 2
```
如果终端显示了下面内容，说明已经安装好了cudnn 8.9.7。

```bash
#define CUDNN_MAJOR 8
#define CUDNN_MINOR 9
#define CUDNN_PATCHLEVEL 7
--
#define CUDNN_VERSION (CUDNN_MAJOR * 1000 + CUDNN_MINOR * 100 + CUDNN_PATCHLEVEL)

/* cannot use constexpr here since this is a C-only file */
```

6. 下面是命令的总结：

```bash
# 下载 cuDNN 8.9.3 的本地安装包（适用于 CUDA 11.x 和 Ubuntu 20.04）
wget https://developer.nvidia.com/downloads/compute/cudnn/secure/8.9.3/local_installers/11.x/cudnn-local-repo-ubuntu2004-8.9.3.28_1.0-1_amd64.deb/

# 安装下载的 cuDNN 本地安装包
sudo dpkg -i cudnn-local-repo-ubuntu2004-8.9.3.28_1.0-1_amd64.deb

# 将 cuDNN 的 GPG 密钥复制到系统的密钥环中，以验证软件包的完整性
sudo cp /var/cudnn-local-repo-ubuntu2004-8.9.3/cudnn-*-keyring.gpg /usr/share/keyrings/

# 更新软件包索引，获取 cuDNN 相关的软件包信息
sudo apt-get update

# 安装 cuDNN 库及其相关组件
sudo apt-get -y install cudnn
```

## OpenCV4.8.0

1. 按照官方说明，在 [OpenCV realeases](https://github.com/opencv/opencv/releases) 和 [opencv_contrib](https://github.com/opencv/opencv_contrib/tags) 中，找到需要安装 [OpenCV 4.7.0](https://github.com/opencv/opencv/archive/refs/tags/4.7.0.tar.gz) 和相应的 [opencv_contrib 4.7.0](https://github.com/opencv/opencv_contrib/archive/refs/tags/4.7.0.tar.gz)将它们下载到同一目录（例如，`~/opencv` ）并解压缩。 然后打开终端并运行：

```bash
cd ~/opencv
cd opencv-4.7.0/
mkdir build
cd build

## 下面是一个cmake编译事例
cmake -DCMAKE_BUILD_TYPE=RELEASE -DWITH_CUDA=ON -DWITH_CUDNN=ON -DOPENCV_DNN_CUDA=ON -DWITH_NVCUVID=ON -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda-11.8（CUDA的目录） -DOPENCV_EXTRA_MODULES_PATH="../../opencv_contrib-4.7.0/modules"（opencv_contrib-4.7.0/modules的目录） -DBUILD_TIFF=ON -DBUILD_ZLIB=ON -DBUILD_JASPER=ON -DBUILD_CCALIB=ON -DBUILD_JPEG=ON -DWITH_FFMPEG=ON ..
## 下面是我的编译命令
cmake -DCMAKE_BUILD_TYPE=RELEASE -DWITH_CUDA=ON -DWITH_CUDNN=ON -DOPENCV_DNN_CUDA=ON -DWITH_NVCUVID=ON -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda-11.8 -DOPENCV_EXTRA_MODULES_PATH="/home/asus/opencv_contrib-4.8.0/modules
 -DBUILD_TIFF=ON -DBUILD_ZLIB=ON -DBUILD_JASPER=ON -DBUILD_CCALIB=ON -DBUILD_JPEG=ON -DWITH_FFMPEG=ON ..

## 花点时间检查 cmake 输出，看看是否有 OpenCV 需要但未安装在设备上的软件包，缺啥补啥

make -j8  #使用make -j$(nproc)自动选择适合自己cpu体质的核心数
## 官方说OpenCV的编译可能会卡在99%，我卡在最后的100%，这可能都是由于最后的链接过程造成的。我们只需等待一段时间，直到它完成并无错误退出即可。
```

2. 将 OpenCV 安装到系统路径：

```
sudo make install
```

默认情况下，使用 cmake 安装软件时，如果没有指定 CMAKE_INSTALL_PREFIX，安装目录通常为 /usr/local。具体来说：

- 可执行文件（如 bin 目录）通常安装在 /usr/local/bin。

- 库文件（如 lib 目录）通常安装在 /usr/local/lib。
- 头文件（如 include 目录）通常安装在 /usr/local/include。
- CMake 配置文件 通常安装在 /usr/local/lib/cmake 或 /usr/local/share/cmake 下。

对于 OpenCV，默认安装路径通常是：/usr/local/lib/cmake/opencv4

如果希望通过在 `cmake` 命令中添加 `-DCMAKE_INSTALL_PREFIX=/your_preferred_path` 选项来将 OpenCV 安装到自定义路径，请记住通过添加额外的 cmake 选项来帮助 Photo-SLAM 找到 OpenCV。 

比如我在上面编译OenCV时指定了命令-DCMAKE_INSTALL_PREFIX=/asus/local。那么 OpenCV 的所有相关文件会安装到 /asus/local 目录下。例如：

- 可执行文件 会安装到 /asus/local/bin。
- 库文件 会安装到 /asus/local/lib。
- 头文件 会安装到 /asus/local/include。
- CMake 配置文件 会安装到 /asus/local/lib/cmake/opencv4。

因此，在配置其他项目（如 Photo-SLAM）时，可以通过指定 -DOpenCV_DIR=/asus/local/lib/cmake/opencv4 来让 CMake 正确找到 OpenCV 的配置文件

否则，您也可以在 `CMakeLists.txt`, `ORB-SLAM3/CMakeLists.txt` 和 `ORB-SLAM3/Thirdparty/DBoW2/CMakeLists.txt` 中添加以下行，就像为 LibTorch 所做的那样。

```bash
set(OpenCV_DIR /your_preferred_path/lib/cmake/opencv4)
```

## LibTorch  2.1.2

***支持的 LibTorch 版本最高为 2.1.2***，如果使用的版本高于此版本，可能会出现兼容性问题。

1. 首先下载 [libtorch](https://download.pytorch.org/libtorch/cu118)和解压缩，关于如何选择可以参考下面内容：

- CUDA 版本匹配：使用的是 CUDA 11.8，因此选择文件名中包含 cu118 的版本。

- C++ ABI 兼容性：

  cxx11-abi 表示使用了 C++11 ABI。如果不知道该选择哪个，通常选择带有 cxx11-abi 的版本，这个是较新的 ABI 方式。
  shared 和 static：shared 表示共享库（动态库），static 表示静态库。一般来说，选择 shared 版本即可。
  是否包含依赖项：

- with-deps 表示该版本包含了所有的依赖项，通常建议选择这个版本以避免缺少依赖的问题。
  without-deps 表示不包含依赖项，适用于系统已经安装了这些依赖项的情况。
  推荐的选择
  考虑到以上几点，可以选择如下版本：libtorch-cxx11-abi-shared-with-deps-2.0.1%2Bcu118.zip
  使用 C++11 ABI 的动态库版本，并且包含所有依赖项，适用于 CUDA 11.8。这个版本通常是最兼容和最容易使用的这也是官方给出的。

2. 解压缩LibTorch：下载完成后，将LibTorch压缩包解压缩到您选择的目录中。

终端命令代码如下：

```bash
wget https://download.pytorch.org/libtorch/nightly/cpu/libtorch-cxx11-abi-shared-with-deps-2.0.1%2Bcu118.zip
unzip libtorch-cu118.zip -d ./the_path_to_where_you_extracted_LibTorch #请将 /path/to/your/libtorch 替换为你想要解压的目标目录。
rm libtorch-cu118.zip
```

3. 配置 CMake 项目
   在使用 LibTorch 的 CMake 项目中，需要确保 CMake 能找到 LibTorch。通常，在调用find_package(Torch REQUIRED)之前，需要在 CMakeLists.txt 文件中指定 Torch_DIR，指向解压后的 libtorch 目录。

```bash
# In CMakeLists.txt
set(Torch_DIR ./the_path_to_where_you_extracted_LibTorch/libtorch/share/cmake/Torch) #请将./the_path_to_where_you_extracted_LibTorch替换为您实际解压缩LibTorch的路径。
```

或者在项目构建时指定LibTorch的目录，即修改`build.sh`在cmake命令中加上下面的内容：

```bash
cmake .. -DTorch_DIR=the_path_to_where_you_extracted_LibTorch/libtorch/share/cmake/Torch -DOpenCV_DIR=/home/rapidlab/libs/opencv/lib/cmake/opencv4
# 请将 ./the_path_to_where_you_extracted_LibTorch 替换为您实际解压 LibTorch 的路径。
```

4. 构建项目：CMake配置完成后，可以去其他项目（photoslam）中编译。
5. 命令总结：

```bash
# 下载 LibTorch 包（夜间构建，适用于 CUDA 11.8 和 C++11 ABI，包含依赖项）
wget https://download.pytorch.org/libtorch/nightly/cpu/libtorch-cxx11-abi-shared-with-deps-2.0.1%2Bcu118.zip -O libtorch-cu118.zip

# 将 LibTorch 压缩包解压到指定目录
unzip libtorch-cu118.zip -d ./the_path_to_where_you_extracted_LibTorch 
# 请将 ./the_path_to_where_you_extracted_LibTorch 替换为你想要解压的目标目录。

# 删除下载的压缩包以节省空间
rm libtorch-cu118.zip

# 在 CMakeLists.txt 中设置 Torch_DIR 变量，指向解压后的 LibTorch 目录
# In CMakeLists.txt
set(Torch_DIR ./the_path_to_where_you_extracted_LibTorch/libtorch/share/cmake/Torch)
# 请将 ./the_path_to_where_you_extracted_LibTorch 替换为您实际解压 LibTorch 的路径。
```

## 安装Photo-SLAM

1. 首先，克隆Photo-SLAM的代码库并编译：

```
git clone https://github.com/HuajianUP/Photo-SLAM.git
cd Photo-SLAM/
chmod +x ./build.sh
./build.sh
```

构建过程中遇到了一些问题：

- 首先是`build.sh`脚本出错，千万别再windows中修改保存提交了文件，否则会出现换行符还有不可见字符的问题，**切记切记**。 
- 编译时显示CUDA_ARCHITECTURES参数未被设置：

```bash
CMake Error in /home/asus/Photo-SLAM/build/CMakeFiles/CMakeTmp/CMakeLists.txt:
  CUDA_ARCHITECTURES is empty for target "cmTC_6707d".


CMake Error in /home/asus/Photo-SLAM/build/CMakeFiles/CMakeTmp/CMakeLists.txt:
  CUDA_ARCHITECTURES is empty for target "cmTC_6707d".


CMake Error at /usr/local/share/cmake-3.22/Modules/CMakeDetermineCompilerABI.cmake:49 (try_compile):
  Failed to generate test project build system.
Call Stack (most recent call first):
  /usr/local/share/cmake-3.22/Modules/CMakeTestCUDACompiler.cmake:19 (CMAKE_DETERMINE_COMPILER_ABI)
  CMakeLists.txt:2 (project)
```

CMake无法为CUDA目标生成合适的构建系统。这是CMake在配置CUDA时常见的问题，尤其是在配置文件未指定适当的CUDA架构时。

要解决此问题，可以采取以下步骤：

手动设置CUDA架构：
在CMake构建文件（通常是CMakeLists.txt）中手动指定适当的CUDA_ARCHITECTURES。你可以在文件的顶部添加以下行：

```cmake
set(CUDA_ARCHITECTURES 70)
```

这里的701表示支持Compute Capability7.0的架构。你需要根据你的GPU架构调整这个值。如果你不确定自己的GPU架构，可以参考[NVIDIA官方文档](https://developer.nvidia.com/cuda-gpus)获取计算能力（Compute Capability）信息。根据文档，NVIDIA GeForce RTX 4070 Ti 的计算能力是 8.9，理论上`set(CUDA_ARCHITECTURES "89")`就可以了，但是 现在nvcc 编译器版本不支持 compute_89 的架构。

![1724045775311](/media/asus/Samsung USB/ubuntu/桌面/photoSlam配置全流程.assets/1724045775311.jpg)

或者在编译时手动设置通用架构：

```bash
cmake -S .. -B . -DCMAKE_CUDA_ARCHITECTURES=70
# -S ..：指定源代码目录。.. 表示上一级目录，也就是 CMakeLists.txt 所在的目录。这意味着 CMake 会从上一级目录中读取 CMakeLists.txt 文件。

# -B .：指定构建目录。. 表示当前目录，也就是你在终端中运行该命令的目录。CMake 会在这个目录中生成所有的构建文件，包括 Makefile、缓存文件等。
```

或者将命令添加进CMakeLists.txt文件开头中

```cmake
cmake_minimum_required(VERSION 3.20 FATAL_ERROR)
project(photo-slam LANGUAGES CXX CUDA)

# 显式指定 CUDA 编译器的路径
set(CMAKE_CUDA_COMPILER /usr/local/cuda-11.8/bin/nvcc)  #此行显式设置了 CMake 使用的 CUDA 编译器路径，确保在所有构建环境中都使用正确的 nvcc

# 设置通用的 CUDA 架构
set(CMAKE_CUDA_ARCHITECTURES 70)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

```

![2024-08-19 13-45-25 的屏幕截图](/media/asus/Samsung USB/ubuntu/桌面/photoSlam配置全流程.assets/2024-08-19 13-45-25 的屏幕截图.png)

显示上面内容就证明彻底编译好了，**庆祝一下！！**

2. 在基准数据集上的Photo-SLAM示例

本文中提到的基准数据集包括：Replica（NICE-SLAM版本）、TUM RGB-D、EuRoC。

| 数据集                       | 概述                                                         | 内容                                                         | 用途                                                         |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Replica（NICE-SLAM版本）** | 一个高质量的室内 3D 扫描数据集，适用于 SLAM 系统评估和开发。 | 包括多个高分辨率室内场景，提供真实感的纹理和几何结构，逐帧 RGB-D 数据。 | 用于评估和开发 SLAM 系统，尤其适合处理复杂几何和高分辨率场景。 |
| **TUM RGB-D**                | 由德国慕尼黑工业大学创建的 RGB-D 数据集，广泛用于视觉 SLAM 评估。 | 包含各种室内环境的 RGB-D 图像序列，包括彩色图像、深度图像和摄像机位姿数据。 | 用于 SLAM、视觉里程计、3D 重建、相机跟踪和物体识别等领域。   |
| **EuRoC**                    | 欧洲机器人项目提供的视觉惯性 SLAM 数据集，适用于视觉和惯性融合的研究。 | 包括 MAV 拍摄的室内场景，同步的立体摄像头图像、IMU 数据和地面真实轨迹。 | 用于开发和测试视觉与惯性融合的 SLAM 系统，特别适用于动态场景和 MAV 应用。 |

作者很贴心的将数据集下载整理为脚本，国内的同学有能力还是提前看脚本中的网站提前自己下好（这里是下载zip格式的，不要下载成rosbag）：

```bash
# 1. 下载并解压 Replica 数据集（NICE-SLAM 版本）
wget https://cvg-data.inf.ethz.ch/nice-slam/data/Replica.zip
unzip Replica.zip
# -------------------------------------------------------------------------------------------------
# 2. 下载并解压 TUM RGB-D 数据集
# Freiburg1 - Desk
wget https://vision.in.tum.de/rgbd/dataset/freiburg1/rgbd_dataset_freiburg1_desk.tgz
tar -xvzf rgbd_dataset_freiburg1_desk.tgz

# Freiburg2 - XYZ
wget https://vision.in.tum.de/rgbd/dataset/freiburg2/rgbd_dataset_freiburg2_xyz.tgz
tar -xvzf rgbd_dataset_freiburg2_xyz.tgz

# Freiburg3 - Long Office Household
wget https://vision.in.tum.de/rgbd/dataset/freiburg3/rgbd_dataset_freiburg3_long_office_household.tgz
tar -xvzf rgbd_dataset_freiburg3_long_office_household.tgz
# --------------------------------------------------------------------------------------------------
# 3. 下载并解压 EuRoC 数据集
# Machine Hall - MH_01_easy
wget http://robotics.ethz.ch/~asl-datasets/ijrr_euroc_mav_dataset/machine_hall/MH_01_easy/MH_01_easy.zip
unzip MH_01_easy.zip

# Machine Hall - MH_02_easy
wget http://robotics.ethz.ch/~asl-datasets/ijrr_euroc_mav_dataset/machine_hall/MH_02_easy/MH_02_easy.zip
unzip MH_02_easy.zip

# Vicon Room 1 - V1_01_easy
wget http://robotics.ethz.ch/~asl-datasets/ijrr_euroc_mav_dataset/vicon_room1/V1_01_easy/V1_01_easy.zip
unzip V1_01_easy.zip

# Vicon Room 2 - V2_01_easy
wget http://robotics.ethz.ch/~asl-datasets/ijrr_euroc_mav_dataset/vicon_room2/V2_01_easy/V2_01_easy.zip
unzip V2_01_easy.zip

```

- （可选）使用脚本下载数据集：

  ```
  cd scripts
  chmod +x ./*.sh
  ./download_replica.sh
  ./download_tum.sh
  ./download_euroc.sh
  ```

- 测试时，可以使用以下命令运行系统。对于Replica数据集，请在指定`PATH_TO_Replica`和`PATH_TO_SAVE_RESULTS`路径后执行命令。如果单纯为评估效果，可以通过添加`no_viewer`参数来禁用可视化界面：

  ```bash
  ./bin/replica_rgbd \ # 可执行文件，用于运行 Replica RGB-D SLAM 程序
      ./ORB-SLAM3/Vocabulary/ORBvoc.txt \ # ORB-SLAM3 使用的字典文件路径，用于特征匹配
      ./cfg/ORB_SLAM3/RGB-D/Replica/office0.yaml \ # ORB-SLAM3 的配置文件，包含 RGB-D 模式下的参数设置
      ./cfg/gaussian_mapper/RGB-D/Replica/replica_rgbd.yaml \ # 高斯映射器的配置文件，设置映射过程的相关参数
      PATH_TO_Replica/office0 \ # Replica 数据集的路径，包含用于处理的场景数据（如 office0 场景）
      PATH_TO_SAVE_RESULTS \ # 结果保存路径，指定程序运行后生成的输出文件的保存位置
      # no_viewer # （可选）禁用可视化窗口运行，仅在后台处理并保存结果

下面是我的命令：

```bash
./bin/replica_rgbd \
./ORB-SLAM3/Vocabulary/ORBvoc.txt \
./cfg/ORB_SLAM3/RGB-D/Replica/office0.yaml \
./cfg/gaussian_mapper/RGB-D/Replica/replica_rgbd.yaml \
/media/asus/E/数据集/Replica/office0 \
/home/asus/result/Replica_0
```

运行可得到下面效果：

![2024-08-19 14-10-44 的屏幕截图](/media/asus/Samsung USB/ubuntu/桌面/photoSlam配置全流程.assets/2024-08-19 14-10-44 的屏幕截图.png)

对于其他数据集，可以参考脚本在Photo-SLAM/scripts文件夹下的.sh文件。比如euro数据集：


```bash
# 脚本中给出的
../bin/euroc_stereo \
    ../ORB-SLAM3/Vocabulary/ORBvoc.txt \
    ../cfg/ORB_SLAM3/Stereo/EuRoC/EuRoC.yaml \
    ../cfg/gaussian_mapper/Stereo/EuRoC/EuRoC.yaml \
    /home/rapidlab/dataset/VSLAM/EuRoC/MH_01_easy \
    ../cfg/ORB_SLAM3/Stereo/EuRoC/EuRoC_TimeStamps/MH01.txt \
    ../results/euroc_stereo_$i/MH_01_easy \
    no_viewer
# 根据我的情况自定义的
./bin/euroc_stereo \
    ./ORB-SLAM3/Vocabulary/ORBvoc.txt \
    ./cfg/ORB_SLAM3/Stereo/EuRoC/EuRoC.yaml \
    ./cfg/gaussian_mapper/Stereo/EuRoC/EuRoC.yaml \
    /media/asus/E/数据集/Euroc数据集/MH_01_easy \
    ./cfg/ORB_SLAM3/Stereo/EuRoC/EuRoC_TimeStamps/MH01.txt \
    /home/asus/result/MH01_EASY \
    #no_viewer
```

![2024-08-19 14-20-01 的屏幕截图](/media/asus/Samsung USB/ubuntu/桌面/photoSlam配置全流程.assets/2024-08-19 14-20-01 的屏幕截图.png)

说实话，个人感觉跑euro数据集效果不是很好。

其他的数据集类似。总结一下要跑photoslam系统需要生么内容。

ORB相关的内容：词典，相机参数，时间戳（可选）？；高斯绘图相关：相机参数；数据输入路径；结果输出路径

作者还提供了在论文中提到的所有基准数据集上进行实验的脚本。 将每个序列运行五次，以降低系统非确定性的影响。 需要更改 scripts/*.sh 中的数据集根目录行，然后运行：

```bash
cd scripts
chmod +x ./*.sh
./replica_mono.sh
./replica_rgbd.sh
./tum_mono.sh
./tum_rgbd.sh
./euroc_stereo.sh
```

3. 其他脚本

## **Photo-SLAM评估系统**

克隆Photo-SLAM的评估代码库：

```bash
git clone https://github.com/HuajianUP/Photo-SLAM-eval.git
```


要使用此工具包，需要确保在每个数据集上的结果都以正确的格式存储。如果使用我们提供的 ./xxx.sh 脚本进行实验，结果将存储在以下结构中：

```bash
results
├── replica_mono_0
│   ├── office0
│   ├── ....
│   └── room2
├── replica_rgbd_0
│   ├── office0
│   ├── ....
│   └── room2
│
└── [replica/tum/euroc]_[mono/stereo/rgbd]_num  ....
    ├── scene_1
    ├── ....
    └── scene_n
```

安装所需的 Python 包：

```bash
pip install evo numpy scipy scikit-image lpips pillow tqdm plyfile
```

（可选）安装用于渲染的子模块:

```bash
# 如果您已安装原始的 GS 子模块，可以跳过这些步骤。

pip install submodules/simple-knn/ 
pip install submodules/diff-gaussian-rasterization/
```

转换 Replica GT 摄像机位姿文件为适用于 EVO 包的位姿文件格式

```bash
python ./Photo-SLAM-eval/shapeReplicaGT.py --replica_dataset_path PATH_TO_REPLICA_DATASET
```

将 TUM 的 camera.yaml 复制到相应的数据集路径
由于 TUM 数据集的某些序列中的图像存在畸变，我们需要在评估前对地面真实图像进行去畸变处理。此外，文件 camera.yaml 在 run.py 中作为一个指示器使用。

```bash
cp ./Photo-SLAM-eval/TUM/fr1/camera.yaml PATH_TO_TUM_DATASET/rgbd_dataset_freiburg1_desk
cp ./Photo-SLAM-eval/TUM/fr2/camera.yaml PATH_TO_TUM_DATASET/rgbd_dataset_freiburg2_xyz
```

要获取所有指标，可以运行

```bash
python ./Photo-SLAM-eval/onekey.py --dataset_center_path PATH_TO_ALL_DATASET --result_main_folder RESULTS_PATH
```

最终，应该会获得两个文件：RESULTS_PATH/log.txt 和 RESULTS_PATH/log.csv。

## 使用实景相机的Photo-SLAM示例

这里我没有Intel RealSense D455，没有进行测试，放出官网给的教程：

**我们提供了使用Intel RealSense D455的示例代码，位于`examples/realsense_rgbd.cpp`。运行示例请参考`scripts/realsense_d455.sh`脚本。**

