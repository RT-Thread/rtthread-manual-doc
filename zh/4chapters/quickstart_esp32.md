# RT-Thread/ESP32 入门指南

## 下载rtthread-esp分支
** 方法1: **   
	* git clone --recursive https://github.com/BernardXiong/rtthread-esp  
** 方法2: **   
	* git clone https://github.com/BernardXiong/rtthread-esp  
	* git submodule init  
	* git submodule update  

## 下载交叉编译工具链
**Linux (x64):**  
<https://dl.espressif.com/dl/xtensa-esp32-elf-linux64-1.22.0-59.tar.gz>  
**Linux (x32):**  
<https://dl.espressif.com/dl/xtensa-esp32-elf-linux32-1.22.0-59.tar.gz>  
**MacOS:**  
<https://dl.espressif.com/dl/xtensa-esp32-elf-osx-1.22.0-59.tar.gz>  
**Windows:**  
<https://dl.espressif.com/dl/xtensa-esp32-elf-win32-1.22.0-59.zip>

## 依赖软件  
1. scons  
2. python  
3. pyserial

## 编译  
1. 修改根目录下"rtconfig.py"文件内EXEC_PATH   = r'D:\tools\msys32\opt\xtensa-esp32-elf\bin'为xtensa-esp32-elf交叉编译器所在目录   
2. 在命令行下使用scons命令编译  
3. 使用python esp-idf/components/esptool_py/esptool/esptool.py --chip esp32 elf2image --flash_mode "dio" --flash_freq "40m" --flash_size "4MB"  -o rtthread.bin rtthread-esp32.elf命令生成bin文件  
4. 也可直接使用make命令在根目录下编译并生成rtthread.bin文件
