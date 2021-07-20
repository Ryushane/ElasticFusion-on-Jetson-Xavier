# ElasticFusion-on-Jetson-Xavier
Implement ElasticFusion on Nvidia Jetson Xavier

当年做毕设记录的一些配置，虽然现在没在做这个方向了，扔出来记录一下。

## 环境配置

安装依赖

```bash
sudo apt-get install -y cmake-qt-gui git build-essential libusb-1.0-0-dev libudev-dev openjdk-8-jdk freeglut3-dev libglew-dev libsuitesparse-dev libeigen3-dev zlib1g-dev libjpeg-dev
```

nmd,wsm？中间发现包下不下来，我以为是代理出问题了，看了一下是连不上，结果调了半天还是连不上，然后ping了一下国内的网站也ping不通，才发现网口没网，我以为网口出了问题，最后捣鼓半天，去network里设置代理的时候才发现不知道飞行模式为什么开了，卧槽。

然而我弄好了飞行模式发现还是无法科学上网，这就很迷惑了，在我一番尝试之后，我发现是特么校园网搞的鬼，就算我直接连接网线，也得登录校园网的系统才能上其他的网站，最可气的是它可以裸连百度，在我连接bilibili的时候才自动跳转到登录界面，我佛了。

测试`proxychains`

```
proxychains4 curl www.google.com
```

会有输出的

## 依赖项

```
sudo apt-get install -y cmake-qt-gui git build-essential libusb-1.0-0-dev libudev-dev openjdk-8-jdk freeglut3-dev libglew-dev libsuitesparse-dev libeigen3-dev zlib1g-dev libjpeg-dev
```



## OpenNI 2

之前因为无法安装OpenNI 2我以为凉了，但是去issue里面看发现有人编译了arm 64的版本

[Arm 64版本](https://github.com/mikeh9/OpenNI2-TX1)

编译步骤参考下面的教程，[来源](https://www.jetsonhacks.com/2014/08/28/building-openni2-structure-sensor/)

[视频教程](https://www.youtube.com/watch?v=Bn9WqbYtNBw&feature=emb_logo)

**由于上面那个老哥上传的Arm 64版本已经把OpenNI2 build要修改的文件更改完了，我们直接做libfreenect build之后的东西就可以了。**

---

> ```bash
> cd libfreenect
> mkdir build
> cd build
> cmake ..
> make
> sudo make install
> # Build the OpenNI2 driver
> cmake .. -DBUILD_OPENNI2_DRIVER=ON
> make
> # Copy libfreenect to the OpenNI2 driver directory
> Repository=../../Bin/Arm-Release/OpenNI2/Drivers
> cp -L lib/OpenNI2-FreenectDriver/libFreenectDriver* ${Repository}
> # You may have to set permissions to be able to use the Sensor
> sudo usermod -a -G video ubuntu
> # Switch back to OpenNI2 directory here
> # Now copy the libraries and include files to /usr
> sudo cp -r Include /usr/include/openni2
> sudo cp -r Bin/Arm-Release/OpenNI2 /usr/lib/
> sudo cp Bin/Arm-Release/libOpenNI2.* /usr/lib/
> # Create a package config file
> # this will enable ubuntu to find the location of the drivers, libraries and include files.
> sudo gedit /usr/lib/pkgconfig/libopenni2.pc
> 
> and fill it with this:
> 
> prefix=/usr
> exec_prefix=${prefix}
> libdir=${exec_prefix}/lib
> includedir=${prefix}/include/openni2
> 
> Name: OpenNI2
> Description: A general purpose driver for all OpenNI cameras.
> Version: 2.2.0.0
> Cflags: -I${includedir}
> Libs: -L${libdir} -lOpenNI2 -L${libdir}/OpenNI2/Drivers -lDummyDevice -lOniFile -lPS1080.so
> 
> # To make sure it is correctly found, run
> pkg-config --modversion libopenni2
> 
> # which should give the same version as defined in the file above (2.2.0.0)
> 
> # Linux has used the udev system to handle devices such as USB connected peripherals.
> 
> cd Packaging/Linux
> sudo cp primesense-usb.rules /etc/udev/rules.d/557-primesense-usb.rules
> ```

---

- 在上面cmake一步就遇到了问题，因为之前编译的路径不同有缓存，两者不匹配了。解决办法把build文件夹删了重新建一个，或者`make clean`也可以。
- 一路下来没什么问题，不过NIViewer我运行不了，本来的项目了就没有，不管了，我就当可以用吧。（后来证明其实没问题）

## 安装Pangolin 

Pangolin是一个面向OpenGL显示、交互的轻度开发库。

### 必要依赖

- 下载源码

```bash
git clone https://github.com/stevenlovegrove/Pangolin.git
```

- 安装OpenGl

  ```bash
  sudo apt install libgl1-mesa-dev
  ```

- Glew

  ```bash
  sudo apt install libglew-dev
  ```

- CMake

  ```bash
  sudo apt install cmake
  ```

### 推荐依赖

- Python2 / Python3 （自带了）

- Wayland

  ```bash
  sudo apt install pkg-config
  sudo apt install libegl1-mesa-dev libwayland-dev libxkbcommon-dev wayland-protocols
  ```

出了个问题，就是`libxkbcommon-dev`安装的时候需要`libxkbcommon0`的版本为0.5.0，但是已经安装了0.8.0。解决办法就是把`libxkbcommon0`先卸载了...再安装`libxkbcommon-dev`。（如果遇到了就卸载一下，没遇到更好）

### Building

```bash
git clone https://github.com/stevenlovegrove/Pangolin.git
cd Pangolin
mkdir build
cd build
cmake ..
cmake --build .
```

## Build ElasticFusion

这个作者写了个脚本，但是我用不了，把前面装驱动还有Pangolin的都删了，开始build ElasticFusion。这个比还调整过文件目录，搞的自己写的脚本都用不了。

主要编译下面三个部分

---

### Core的编译：

```bash
cd ElasticFusion
cd Core
cd src
mkdir build
cd build
cmake ..
make -j8
sudo make install
sudo ldconfig
```

这里需要注意的是，作者可能调整过目录，所以需要先进入`src`文件夹进行编译，然后再把编译好的`build`文件夹复制到`Core`目录下（因为GPUTest的编译是依赖于`Core/build`的）。

这里编译又遇到了问题出现了 c++错误`unrecognized command line option '-msse2' '-msse3'`

在`CMakeList.txt`里面把涉及到这两个命令的`msse`参数删掉就顺利编译通过了，因为`msse`是针对x86平台的，用来启动`Streaming SIMD Extensions (SSE)`，简单来说就是嘤特尔搞的运算加速（一种运算实施在几个数据上）。Arm不支持，删了就行了。

### GPUTest的编译：

编译Core成功后，再编译GPUTest就很容易了。

```bash
cd ElasticFusion
cd GPUTest
cd src
mkdir build
cd build
cmake ..
make -j8
sudo make install
sudo ldconfig
```

我编译的时候发现`make install`会报`Nothing to make`的问题，不过没管它，直接`sudo ldconfig`就可以了

### GUI的编译：

```bash
cd ElasticFusion
cd GUI
cd src
mkdir build
cd build
cmake ..
make -j8
sudo make install
sudo ldconfig
```

---

手动build又遇到问题了`Unable to find SuiteSparse`

还好有大哥提了个[issue](https://github.com/mp3guy/ElasticFusion/issues/154)

把`ElasticFusion/Core/src/FindSuiteSparse.cmake`里面的内容用附件中的文件内容替换

[FindSuiteSparse.cmake](https://ryushane.top/usr/uploads/2019/11/1135865494.txt)

---

###  ElasticFusion的数据集测试

 最后进入ElasticFusio/GUI/build/中，使用ElasticFusion进行运行数据集：

```
./ElasticFusion -l fr1_xyz.klg
```

跑通了，真是艰难啊...

![elastic](https://cdn.jsdelivr.net/gh/Ryushane/PicGo_Pictures/img/ElasticFusion.png)

---

## Issues

### invalid texture reference

使用官方的数据集进行测试的时候发现了`invalid texture reference`的错误，作者说是`core build`中CMake的参数问题。因为Xavier的架构不在此列，直接把`Core/src/CMakeList.txt`里面这行删除，重新编译。

```cmake
set(CUDA_ARCH_BIN "30 35 50 52 61" CACHE STRING "Specify 'real' GPU arch to build binaries for, BIN(PTX) format is supported. Example: 1.3 2.1(1.3) or 13 21(13)")
```

根本不是这个问题，在[墨釉](https://www.twblogs.net/a/5c547855bd9eee06ef365316/zh-cn)的博客中找到了答案，就是上面在每次`make`后要执行`sudo make install`与`sudo ldconfig`来找到动态链接库，否则测试数据集的时候会出错。

重建之后的结果。

![final](https://cdn.jsdelivr.net/gh/Ryushane/PicGo_Pictures/img/reconstruction.png)

[1]: https://ryushane.top/usr/uploads/2019/11/1135865494.txt	"FindSuiteSparse.cmake"

