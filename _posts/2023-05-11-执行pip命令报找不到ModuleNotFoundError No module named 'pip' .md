---
layout: post                          # 表明是博文  
title: "ModuleNotFoundError: No module named 'pip'"           # 博文的标题  
date: 2023-05-11                 # 博文的发表日期，此日期决定主页上博文的先后顺序  
author: "秦"                       # 博文的作者  
catalog: True                         # 开启catalog，将在博文侧边展示博文的结构  
tags:
    - Python
---
# 执行pip命令报找不到ModuleNotFoundError: No module named 'pip' 

环境信息：python3.10

解决办法：

 1. 首先我们需要确定有没有安装pip,执行`python -m ensurepip`

 2. 如果返回`ModuleNotFoundError: No module named ensurepip`说明没装pip，下一步进行安装

 3. `curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py # 下载安装脚本`

 4. `python get-pip.py`

 5. 然后再进行测试，`pip list`

 6. 如果报`ModuleNotFoundError: No module named 'pip'` 错误

 7. 进行下面操作：

 8. 找到python3.x安装目录，打开python3xx._pth文件，内容应该默认是如下所示：

    ```shell
    python310.zip
    .
    
    # Uncomment to run site.main() automatically
    #import site
    ```



 9. 第三行位置，添加`Lib\site-packages`如下所示：

     ```shell
     python310.zip
     .
     Lib\site-packages
     # Uncomment to run site.main() automatically
     #import site
     ```

10.运行pip list 测试，完成
