# eaglePCB
#元器件
    - symbol 符号 用于原理图
    - package 封装 用于PCB
    - device 元件 用于表明symbol与package的关系

## 元器件保存在元器件库中 `.lbr`
# 工程
    - 原理图 .sch
    - PCB .brd 

# 如何生成gerber文件 aka RS-274x
    1. 新建工程
    2. 在工程下新建原理图
    3. 将元器件放入原理图
    3. 添加电气连接
    4. 使用原理图生成PCB
    4. 摆放元器件
    4. 布线
    5. 根据PCB，利用CAM处理器生成gerber文件
    6. 发送gerber文件到PCB厂打样

    failed
    