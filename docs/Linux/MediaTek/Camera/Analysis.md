<!--
 * @Author: WangGuanran
 * @Email: wangguanran@vanzotec.com
 * @Date: 2020-03-08 17:21:14
 * @LastEditTime: 2020-03-08 18:35:25
 * @LastEditors: WangGuanran
 * @Description: MTK Camera Analysis
 * @FilePath: /docs/Linux/MediaTek/Camera/Analysis.md
 -->

# 简介

本文档主要介绍MTK平台驱动注册以及调用流程的分析。不同平台可能会存在差异，大体流程应该不会有太大区别，本人未参加MTK培训，所有介绍仅为个人分析结果，仅供参考。
本文档以MT6739平台，Android版本7.1，Kernel版本Kernel-4.4为例介绍。

# Kernel层驱动注册流程分析

MTK平台Camera驱动路径：

* kernel-xxx/drivers/misc/mediatek/imgsensor

Camera驱动文件：

*kernel-xxx/drivers/misc/mediatek/imgsensor/src/mt6739/imgsensor.c

MTK平台驱动是挂接在platform总线上的Driver。

```C
#ifdef CONFIG_OF  
static const struct of_device_id gimgsensor_of_device_id[] = {  
    { .compatible = "mediatek,camera_hw", },  
    {}  
};  
#endif  
  
static struct platform_driver gimgsensor_platform_driver = {  
    .probe      = imgsensor_probe,  
    .remove     = imgsensor_remove,  
    .suspend    = imgsensor_suspend,  
    .resume     = imgsensor_resume,  
    .driver     = {  
        .name   = "image_sensor",  
        .owner  = THIS_MODULE,  
#ifdef CONFIG_OF  
        .of_match_table = gimgsensor_of_device_id,  
#endif  
    }  
};  
```

设备树信息在平台dtsi中配置，保存有Camera相关的Clock配置，pinctrl配置，supply配置：
设备树路径：

* ./arch/arm/boot/dts/mt6739.dtsi(平台设备树信息：compatible,supply,clock)
* ./arch/arm/boot/dts/xxx.dts(项目设备树信息:pinctrl)

下面介绍compatible配对成功后执行probe函数之后的程序流程。

```C
static int imgsensor_probe(struct platform_device *pdev)  
{  
    /* Register char driver */  
    if (imgsensor_driver_register()) {  
        PK_PR_ERR("[CAMERA_HW] register char device failed!\n");  
        return -1;  
    }  
  
    gpimgsensor_hw_platform_device = pdev;  
  
#ifndef CONFIG_FPGA_EARLY_PORTING  
    imgsensor_clk_init(&pgimgsensor->clk);  
#endif  
    imgsensor_hw_init(&pgimgsensor->hw);  
    imgsensor_i2c_create();  
    imgsensor_proc_init();  
#ifdef VANZO_FAKE_MAIN2_CAMERA_SUPPORT  
    imgsensor_sysfs_init();  
#endif  
  
    atomic_set(&pgimgsensor->imgsensor_open_cnt, 0);  
#ifdef CONFIG_MTK_SMI_EXT  
    mmdvfs_register_mmclk_switch_cb(mmsys_clk_change_cb,MMDVFS_CLIENT_ID_ISP);  
#endif  
  
    return 0;  
}  
```

第04行：imgsensor_driver_register（）用来Attatch file operation结构体，创建类（存放在/sysfs中），创建相应的设备节点（从/sysfs下查找对应的类在/dev目录下创建设备节点）；

```C
static const struct file_operations gimgsensor_file_operations = {  
    .owner = THIS_MODULE,  
    .open = imgsensor_open,  
    .release = imgsensor_release,  
    .unlocked_ioctl = imgsensor_ioctl,  
#ifdef CONFIG_COMPAT  
    .compat_ioctl = imgsensor_compat_ioctl  
#endif  
};  
```

第14行：imgsensor_hw_init（）主要有两个作用：

1. regulator、pinctrl、mclk的设备树解析及注册
2. 解析imgsensor_custom_config结构体，获得对应pin脚的驱动方式

imgsensor_hw_init函数的定义位置在：

* src/mt6739/imgsensor_hw.c

imgsensor_custom_config结构体是配置Camera相关PIN脚如AVDD、DOVDD等引脚的驱动方式（需要根据硬件连接确认）。其定义位置在：

* src/mt6739/camera_hw/imgsensor_cfg_table.c

第15行：imgsensor_i2c_create（）是注册Camera的I2C驱动
第16行：imgsensor_proc_init（）用来创建proc文件，其中主要使用driver/camera_info，Camera的I2C通了的话会在保存相应的Camera信息在此节点内。
至此所有的驱动注册、匹配均已完毕，上层可以通过ioctrl来调用执行相应的上电、下电等操作。

# Android层调用流程分析

# Revision history

| 版本 | 修改时间     | 修改人 | 修改备注     |
| :--- | :----------- | :----- | :----------- |
| V1.0 | 2019年8月8日 | 王冠然 | Init Version |
