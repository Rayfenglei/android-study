> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127796497)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.MTK 去掉开机通过长按电源键 + 音量加进入 recovery 工厂模式的核心类](#t1)

[3.MTK 去掉开机通过长按电源键 + 音量加进入 recovery 工厂模式的核心功能分析和实现](#t2)

 [3.1 boot_mode.c 关于处理 recovery 模式的相关功能](#t3)

[3.2 factory.c 关于启动模式进入工厂模式的修改](#t4)

1. 概述
-----

  在进行系统产品开发中，当设备开机启动的时候，可以通过长按电源键和音量加减然后会弹出  
[recovery](https://so.csdn.net/so/search?q=recovery&spm=1001.2101.3001.7020) 工厂模式 快速启动模式 等可以通过音量键上下移动 然后音量键确定进入对应的模式  
因为产品开发需要要求不能通过电源键和音量加减进入 recovery 模式和[工厂模式](https://so.csdn.net/so/search?q=%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F&spm=1001.2101.3001.7020)，所以要求去掉这个功能

2.[MTK](https://so.csdn.net/so/search?q=MTK&spm=1001.2101.3001.7020) 去掉开机通过长按电源键 + 音量加进入 recovery 工厂模式的核心类
----------------------------------------------------------------------------------------------------------

```
vendor/mediatek/proprietary/bootable/bootloader/lk/platform/mt6580/boot_mode.c
vendor/mediatek/proprietary/bootable/bootloader/lk/platform/mt6580/factory.c
```

3.MTK 去掉开机通过长按电源键 + 音量加进入 recovery 工厂模式的核心功能分析和实现
-------------------------------------------------

在 MTK 平台的产品开发中，关于对开机启动过程中 按键操作的相关处理是在 vendor/mediatek/proprietary/bootable/bootloader/lk/platform/  
下芯片类型 下的 boot_mode.c 处理进入 recovery 模式，而在 factory.c 进入处理工厂模式  
接下来看分析代码来处理不让对应模式，改为平常启动模式就可以了

 3.1 boot_mode.c 关于处理 recovery 模式的相关功能
--------------------------------------

```
      #ifdef NO_BOOT_MODE_SEL
/******************************************************************************
******************************************************************************/
void boot_mode_select(void)
{
	dprintf(CRITICAL, "Warning: %s is turned off!\n", __func__);
}
 
#else
 
/******************************************************************************
******************************************************************************/
void boot_mode_select(void)
{
 
	int factory_forbidden = 0;
	//  int forbid_mode;
	/*We put conditions here to filer some cases that can not do key detection*/
	extern int kedump_mini(void) __attribute__((weak));
	if (kedump_mini) {
		if (kedump_mini()) {
			mrdump_check();
			if (g_boot_mode == FASTBOOT)
				return;
#ifdef MTK_PMIC_FULL_RESET
			dprintf(CRITICAL, "kedump:full pmic reset!\n");
			mtk_arch_full_reset();
#else
			dprintf(CRITICAL, "kedump:sw reset!\n");
			mtk_arch_reset(1);
#endif
			return;
		}
	}
	if (meta_detection()) {
		return;
	}
 
#if defined (HAVE_LK_TEXT_MENU)
	/*Check RTC to know if system want to reboot to Fastboot*/
	if (Check_RTC_PDN1_bit13()) {
		dprintf(CRITICAL, "[FASTBOOT] reboot to boot loader\n");
		g_boot_mode = FASTBOOT;
		Set_Clr_RTC_PDN1_bit13(false);
		return;
	}
	/*If forbidden mode is factory, cacel the factory key detection*/
	if (g_boot_arg->sec_limit.magic_num == 0x4C4C4C4C) {
		if (g_boot_arg->sec_limit.forbid_mode == F_FACTORY_MODE) {
			//Forbid to enter factory mode
			dprintf(CRITICAL, "%s Forbidden\n",MODULE_NAME);
			factory_forbidden=1;
		}
	}
	//  forbid_mode = g_boot_arg->boot_mode &= 0x000000FF;
	/*If boot reason is power key + volumn down, then
	 disable factory mode dectection*/
	if (mtk_detect_pmic_just_rst()) {
		factory_forbidden=1;
	}
	/*Check RTC to know if system want to reboot to Recovery*/
#ifndef CFG_MTK_WDT_COMMON
	if (Check_RTC_Recovery_Mode()) {
#else
	if (Check_RTC_Recovery_Mode() ||
	    (mtk_wdt_restart_reason() == MTK_WDT_NONRST2_BOOT_RECOVERY)) {
#endif
		g_boot_mode = RECOVERY_BOOT;
		return;
	}
	/*If MISC Write has not completed  in recovery mode
	 before system reboot, go to recovery mode to
	finish remain tasks*/
	if (recovery_check_command_trigger()) {
		return;
	}
	ulong begin = get_timer(0);
 
	/*we put key dectection here to detect key which is pressed*/
	dprintf(INFO, "eng build\n");
	while (get_timer(begin)<50) {
 
 
		if (!factory_forbidden) {
			if (mtk_detect_key(MT65XX_FACTORY_KEY)) {
				dprintf(CRITICAL, "%s Detect key\n",MODULE_NAME);
				dprintf(CRITICAL, "%s Enable factory mode\n",MODULE_NAME);
				g_boot_mode = NORMAL_BOOT;
				//video_printf("%s : detect factory mode !\n",MODULE_NAME);
				return;
			}
		}
 
		if (mtk_detect_key(MT65XX_BOOT_MENU_KEY)) {
			dprintf(CRITICAL, "\n%s Check  boot menu\n",MODULE_NAME);
			dprintf(CRITICAL, "%s Wait 50ms for special keys\n",MODULE_NAME);
			//mtk_wdt_disable();
			/*************************/
			//mt65xx_backlight_on();
			/*************************/
			//boot_menu_select();
			//mtk_wdt_init();
            g_boot_mode = NORMAL_BOOT;
			return;
		}
#ifdef MT65XX_RECOVERY_KEY
		if (mtk_detect_key(MT65XX_RECOVERY_KEY)) {
			dprintf(CRITICAL, "%s Detect cal key\n",MODULE_NAME);
			dprintf(CRITICAL, "%s Enable recovery mode\n",MODULE_NAME);
			g_boot_mode = NORMAL_BOOT;
			//video_printf("%s : detect recovery mode !\n",MODULE_NAME);
			return;
		}
#endif
	}
#else
 
	/*We put conditions here to filer some cases that can not do key detection*/
 
	/*Check RTC to know if system want to reboot to Fastboot*/
#ifdef MTK_FASTBOOT_SUPPORT
	if (Check_RTC_PDN1_bit13()) {
		dprintf(INFO,"[FASTBOOT] reboot to boot loader\n");
		g_boot_mode = FASTBOOT;
		Set_Clr_RTC_PDN1_bit13(false);
		return TRUE;
	}
#endif
 
	/*If forbidden mode is factory, cacel the factory key detection*/
	if (g_boot_arg->sec_limit.magic_num == 0x4C4C4C4C) {
		if (g_boot_arg->sec_limit.forbid_mode == F_FACTORY_MODE) {
			//Forbid to enter factory mode
			dprintf(INFO, "%s Forbidden\n",MODULE_NAME);
			factory_forbidden=1;
		}
	}
//    forbid_mode = g_boot_arg->boot_mode &= 0x000000FF;
	/*If boot reason is power key + volumn down, then
	 disable factory mode dectection*/
	if (mtk_detect_pmic_just_rst()) {
		factory_forbidden=1;
	}
	/*Check RTC to know if system want to reboot to Recovery*/
#ifndef CFG_MTK_WDT_COMMON
	if (Check_RTC_Recovery_Mode()) {
#else
	if (Check_RTC_Recovery_Mode() ||
	    (mtk_wdt_restart_reason() == MTK_WDT_NONRST2_BOOT_RECOVERY)) {
#endif
		g_boot_mode = RECOVERY_BOOT;
		return TRUE;
	}
	/*If MISC Write has not completed  in recovery mode
	 and interrupted by system reboot, go to recovery mode to
	finish remain tasks*/
	if (recovery_check_command_trigger()) {
		return TRUE;
	}
	ulong begin = get_timer(0);
	/*we put key dectection here to detect key which is pressed*/
	while (get_timer(begin)<50) {
#ifdef MTK_FASTBOOT_SUPPORT
		if (mtk_detect_key(MT_CAMERA_KEY)) {
			dprintf(INFO,"[FASTBOOT]Key Detect\n");
			g_boot_mode = FASTBOOT;
			return TRUE;
		}
#endif
		if (!factory_forbidden) {
			if (mtk_detect_key(MT65XX_FACTORY_KEY)) {
				dprintf(INFO, "%s Detect key\n",MODULE_NAME);
				dprintf(INFO, "%s Enable factory mode\n",MODULE_NAME);
				g_boot_mode = FACTORY_BOOT;
				//video_printf("%s : detect factory mode !\n",MODULE_NAME);
				return TRUE;
			}
		}
#ifdef MT65XX_RECOVERY_KEY
		if (mtk_detect_key(MT65XX_RECOVERY_KEY)) {
			dprintf(INFO, "%s Detect cal key\n",MODULE_NAME);
			dprintf(INFO, "%s Enable recovery mode\n",MODULE_NAME);
			g_boot_mode = RECOVERY_BOOT;
			//video_printf("%s : detect recovery mode !\n",MODULE_NAME);
			return TRUE;
		}
#endif
	}
 
#endif
 
#ifdef MTK_KERNEL_POWER_OFF_CHARGING
	if (kernel_power_off_charging_detection()) {
		dprintf(CRITICAL, " < Kernel Power Off Charging Detection Ok> \n");
		return;
	} else {
		dprintf(CRITICAL, "< Kernel Enter Normal Boot > \n");
	}
#endif
}
#endif  // NO_BOOT_MODE_SEL
```

在 boot_mode.c 中关于处理启动模式选择的方法 boot_mode_select(void) 中处理是否启动工厂模式  
if (!factory_forbidden) {  
            if (mtk_detect_key(MT65XX_FACTORY_KEY)) {  
                dprintf(CRITICAL, "%s Detect key\n",MODULE_NAME);  
                dprintf(CRITICAL, "%s Enable factory mode\n",MODULE_NAME);  
-                               g_boot_mode = FACTORY_BOOT;  
+                               g_boot_mode = NORMAL_BOOT;  
                //video_printf("%s : detect factory mode !\n",MODULE_NAME);  
                return;  
            }  
        }  
所以把 g_boot_mode = NORMAL_BOOT; 改成 NORMAL_BOOT 表示正常启动即可  
而在  
if (mtk_detect_key(MT65XX_BOOT_MENU_KEY)) {  
            dprintf(CRITICAL, "\n%s Check  boot menu\n",MODULE_NAME);  
            dprintf(CRITICAL, "%s Wait 50ms for special keys\n",MODULE_NAME);  
            //mtk_wdt_disable();  
            /*************************/  
            //mt65xx_backlight_on();  
            /*************************/  
            //boot_menu_select();  
            //mtk_wdt_init();  
            g_boot_mode = NORMAL_BOOT;  
            return;  
        }  
在处理启动模式菜单选择的时候，注释掉处理选择启动模式的相关方法  
在处理 recovery 模式的代码中  
#ifdef MT65XX_RECOVERY_KEY  
        if (mtk_detect_key(MT65XX_RECOVERY_KEY)) {  
            dprintf(CRITICAL, "%s Detect cal key\n",MODULE_NAME);  
            dprintf(CRITICAL, "%s Enable recovery mode\n",MODULE_NAME);  
-                       g_boot_mode = RECOVERY_BOOT;  
+                       g_boot_mode = NORMAL_BOOT;  
            //video_printf("%s : detect recovery mode !\n",MODULE_NAME);  
            return;  
        }  
#endif  
同样也是把 g_boot_mode = NORMAL_BOOT; 改成 NORMAL_BOOT 表示正常启动即可

3.2 factory.c 关于启动模式进入工厂模式的修改
-----------------------------

```
  BOOL factory_check_key_trigger(void)
{
#if 1
	//wait
	ulong begin = get_timer(0);
	dprintf(CRITICAL, "\n%s Check factory boot\n",MODULE_NAME);
	dprintf(CRITICAL, "%s Wait 50ms for special keys\n",MODULE_NAME);
 
	/* If the boot reason is RESET and volume down is uesd as reset key, than we will NOT enter factory mode. */
	//this function is marked because we use volume up to reset phone on ROME
	/*
	if(mtk_detect_pmic_just_rst())
	{
	  return false;
	}
	*/
 
	while (get_timer(begin)<50) {
		if (mtk_detect_key(MT65XX_FACTORY_KEY)) {
			dprintf(CRITICAL, "%s Detect key\n",MODULE_NAME);
			dprintf(CRITICAL, "%s Enable factory mode\n",MODULE_NAME);
			g_boot_mode = NORMAL_BOOT;
			//video_printf("%s : detect factory mode !\n",MODULE_NAME);
			return TRUE;
		}
	}
#endif
	return FALSE;
}
```

在 factory.c 中处理 factory_check_key_trigger 进入工厂模式的方法中在  
if (mtk_detect_key(MT65XX_FACTORY_KEY)) {  
            dprintf(CRITICAL, "%s Detect key\n",MODULE_NAME);  
            dprintf(CRITICAL, "%s Enable factory mode\n",MODULE_NAME);  
-                       g_boot_mode = FACTORY_BOOT;  
+                       g_boot_mode = NORMAL_BOOT;  
            //video_printf("%s : detect factory mode !\n",MODULE_NAME);  
            return TRUE;  
        }  
同样也是把 g_boot_mode = NORMAL_BOOT; 改成 NORMAL_BOOT 表示正常启动即可