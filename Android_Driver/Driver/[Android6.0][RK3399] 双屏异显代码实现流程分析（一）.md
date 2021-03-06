---
title: [Android6.0][RK3399] 双屏异显代码实现流程分析（一）
tags: Rockchip,
grammar_cjkRuby: true
---
Platform: RK3399
OS: Android 6.0
Version: v2016.08

[TOC]

本文分为两部分。
《[RK3399] 双屏异显代码实现流程分析（一）》为分析 RK video 部分标准的代码（base on 2017.2.13 updated）
《[RK3399] 双屏异显代码实现流程分析（二）》为打上双屏异显 patch 后的代码流程分析（eDP + mipi）

## 代码流程
参考 KrisFei 大神总结的 **3288 display 模块加载流程**。
http://blog.csdn.net/kris_fei/article/details/52584903
KrisFei 归纳的代码流程如下：

1. mipi dsi 接口信息初始化
2. fb相关信息读取
3. timing参数初始化
4. mipi dsi controller初始化
5. lcdc控制器注册

## 代码详解
在 **RK3399** 上代码没有太大的变化。下面为 display 部分的标准流程。

在 make menuconfig 配置的时候
  Location:                                                                           
  |     -> Device Drivers                                                              
  |       -> Graphics support                                                      
  |         -> Rockchip Misc Video driver                                           
  |           -> LCD Panel Select (<choice> [=y])
  drivers/video/rockchip/screen/Kconfig

  choice 包括 General lcd panel 和 rk mipi dsi lcd
  差别是
  ```
	< # CONFIG_LCD_GENERAL is not set
	< CONFIG_LCD_MIPI=y
	---
	> CONFIG_LCD_GENERAL=y
	> # CONFIG_LCD_MIPI is not set
```
分别对应的驱动文件是
lcd_general.c
lcd_mipi.c

现在的版本（2017.2.13）中，lcd_general.c  还未实现代码。
所以我们从分析默认的 lcd_mipi.c 开始。

### mipi dsi 接口信息初始化
**lcd_mipi.c**
rk_mipi_screen_init    ->
    platform_driver_probe  ->    //name是rk_mipi_screen
        rk_mipi_screen_probe  ->
            rk_mipi_screen_init_dt    
			//读取mipi信息（包括 screen_init）,dsi_lane,dsi_hs_clk,mipi_dsi_num, power, rst, gpio, 屏幕的 timing 信息（包括 sceen on cmds, cmd_type, cmd_delay, cmd_debug）

### fb相关信息读取
rk_fb_init ->    rk_fb.c
    platform_driver_register ->    //name: "rockchip,rk-fb"
        rk_fb_probe ->    //获取disp-mode, u-boot-logo-on等参数。
            rockchip_ion_client_create    //创建ion client。

/* ION与PMEM类似，管理一或多个内存池，其中有一些会在boot time的时候预先分配，以备给特殊的硬件使用（GPU，显示控制器等）。它通过ION heaps来管理这些pool。它可以被userspace的process之间或者内核中的模块之间进行内存共享。*/

### timing 参数初始化
//不管是那种接口类型的lcd,lcd的时序参数都是要读取的。
rk_screen_init ->    rk_screen.c
    platform_driver_register ->    //name: "rk-screen"
        rk_screen_probe ->
            rk_fb_prase_timing_dt ->  rk_fb.c  //读取来的配置存在结构体 rk_screen 变量中
                of_get_display_timing    //获取时序参数，dts中可以配置多组，这里会循环读取。
                display_timings_get    //根据当前native-mode来选取当前使用哪组时序参数。
                rk_fb_video_mode_from_timing    //把 timing转换到fb video mode中去供后续使用。

### mipi dsi controller 初始化
//如果是另外的接口那就调用相应的接口控制器驱动来初始化.
rk32_mipi_dsi_init ->    rk32_mipi_dsi.c
    platform_driver_register ->    //name: "rk32-mipi"
        rk32_mipi_dsi_probe ->    //初始化struct dsi结构,包括clock, dsi ops, rk_screen 传递过来的参数,
            rk_fb_get_prmry_screen -> rk_screen.c   //获取在之前 rk_screen_probe() 中初始化的rk_screen变量.
				//rk_mipi_dsi_probe -> //这个在 3399 代码中没有了
					register_dsi_ops     //dsi->ops给dsi_ops
				//dsi_probe_current_chip    //检测dsi chip是否存在，这个在 3399 的代码中没有了
				rk_fb_trsm_ops_register        //注册trsm_mipi_ops为trsm_dsi_ops

这里 3288 中的 rk_mipi_dsi_probe 在 3399 中被删掉了
直接在 rk_fb_get_prmry_screen 中 register_dsi_ops，也省略掉了 dsi_probe_current_chip


### lcdc控制器注册：
rk3368_lcdc_module_init ->    rk3368_lcdc.c
    platform_driver_register -> //.name = "rk3368-lcdc",
        rk3368_lcdc_probe ->
		    of_property_read_u32(np, "rockchip,prop", &prop);//判断屏幕是 primary 还是 extend，如果是 extend 会延后 register
            rk3368_lcdc_parse_dt    //读取lcdc控制器的参数
            dev_drv->ops = &lcdc_drv_ops;    //lcdc对应ops
            devm_request_irq    //lcdc对应irq是rk3368_lcdc_isr()
            rk_fb_register    -> //对应ops是lcdc_drv_ops
				rk_fb->lcdc_dev_drv[i] = dev_drv; //根据 RK30_MAX_LCDC_SUPPORT，循环注册两组 lcdc_dev_drv
                init_lcdc_device_driver -> //初始化 lcdc_device_driver
                    init_lcdc_win    //一个lcdc能支持4层win.
                    rk_disp_pwr_ctr_parse_dt    //解析lcdc power ctrl相关内容。
                    rk_fb_set_prmry_screen
                    rk_fb_trsm_ops_get    //根据不同的屏幕类型选择对应的ops.
                framebuffer_alloc    //系统根据 win 的多少来创建相应数量的 fb
				fb_videomode_to_var //将 fb_videomode 转化为 fb_var_screeninfo
				dsp_mode == ONE_VOP_DUAL_MIPI_VER_SCAN
				//判断双屏同显的刷新方式，这里如果是垂直刷新的话
				//设置 fbi->var.xres /= 2;fbi->var.yres *= 2;	fbi->var.xres_virtual /= 2;	fbi->var.yres_virtual *= 2;
                fbi->fbops = &fb_ops;    //fb ops
                rkfb_create_sysfs    //生成到/dev/graphics/fbx/下
                register_framebuffer
                rkfb_create_sysfs    
                //以下 code 只跑一次
                kthread_run    //创建rk_fb_wait_for_vsync_thread
                dev_drv->ops->post_dspbuf    //show logo for primary display device
