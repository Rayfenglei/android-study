> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124807097)

### 1. 概述

在 11.0 12.0 进行 [Recovery](https://so.csdn.net/so/search?q=Recovery&spm=1001.2101.3001.7020) 恢复出厂设置时 发现 真正清理的 字体小了 客户不满意 所以要求改大一点字体 于是 就只能去看  
Recovery 部分的源码 这部分都是 C 语言的代码，字体设置不可能像上层一样，就可以了  
于是就找资源 但是博客 参差不齐 有些就寥寥数语，所以就得从 recovery 部分分析解决问题

### 2. 修改 Recovery 字体图片的大小 (正在清理) 文字大小的核心类

```
build/make/core/Makefile
bootable/recovery/tools/image_generator/ImageGenerator.java

```

### 3. 修改 Recovery 字体图片的大小 (正在清理) 文字大小的核心功能分析和实现

在系统中 Makefile 是在编译中很重要的文件，在 recovery 中关于绘图的一些功能也在这里面有根据不同配置  
设置不同的绘制标准  
所以就从 build/make/core/Makefile 看看能不能找到使用的字体格式

```
resource_dir := $(call include-path-for, recovery)/tools/recovery_l10n/res/
image_generator_jar := $(HOST_OUT_JAVA_LIBRARIES)/RecoveryImageGenerator.jar
zopflipng := $(HOST_OUT_EXECUTABLES)/zopflipng
$(RECOVERY_INSTALLING_TEXT_FILE): PRIVATE_SOURCE_FONTS := $(recovery_noto-fonts_dep) $(recovery_roboto-fonts_dep)
$(RECOVERY_INSTALLING_TEXT_FILE): PRIVATE_RECOVERY_FONT_FILES_DIR := $(call intermediates-dir-for,PACKAGING,recovery_font_files)
$(RECOVERY_INSTALLING_TEXT_FILE): PRIVATE_RESOURCE_DIR := $(resource_dir)
$(RECOVERY_INSTALLING_TEXT_FILE): PRIVATE_IMAGE_GENERATOR_JAR := $(image_generator_jar)
$(RECOVERY_INSTALLING_TEXT_FILE): PRIVATE_ZOPFLIPNG := $(zopflipng)
$(RECOVERY_INSTALLING_TEXT_FILE): PRIVATE_RECOVERY_IMAGE_WIDTH := $(recovery_image_width)
$(RECOVERY_INSTALLING_TEXT_FILE): PRIVATE_RECOVERY_BACKGROUND_TEXT_LIST := \
  recovery_installing \
  recovery_installing_security \
  recovery_erasing \
  recovery_error \
  recovery_no_command
$(RECOVERY_INSTALLING_TEXT_FILE): PRIVATE_RECOVERY_WIPE_DATA_TEXT_LIST := \
  recovery_cancel_wipe_data \
  recovery_factory_data_reset \
  recovery_try_again \
  recovery_wipe_data_menu_header \
  recovery_wipe_data_confirmation
$(RECOVERY_INSTALLING_TEXT_FILE): .KATI_IMPLICIT_OUTPUTS := $(filter-out $(RECOVERY_INSTALLING_TEXT_FILE),$(generated_recovery_text_files))
$(RECOVERY_INSTALLING_TEXT_FILE): $(image_generator_jar) $(resource_dir) $(recovery_noto-fonts_dep) $(recovery_roboto-fonts_dep) $(zopflipng)
	# Prepares the font directory.
	@rm -rf $(PRIVATE_RECOVERY_FONT_FILES_DIR)
	@mkdir -p $(PRIVATE_RECOVERY_FONT_FILES_DIR)
	$(foreach filename,$(PRIVATE_SOURCE_FONTS), cp $(filename) $(PRIVATE_RECOVERY_FONT_FILES_DIR) &&) true
	@rm -rf $(dir $@)
	@mkdir -p $(dir $@)
	$(foreach text_name,$(PRIVATE_RECOVERY_BACKGROUND_TEXT_LIST) $(PRIVATE_RECOVERY_WIPE_DATA_TEXT_LIST), \
	  $(eval output_file := $(dir $@)/$(patsubst recovery_%,%_text.png,$(text_name))) \
	  $(eval center_alignment := $(if $(filter $(text_name),$(PRIVATE_RECOVERY_BACKGROUND_TEXT_LIST)), --center_alignment)) \
	  java -jar $(PRIVATE_IMAGE_GENERATOR_JAR) \
	    --image_width $(PRIVATE_RECOVERY_IMAGE_WIDTH) \
	    --text_name $(text_name) \
	    --font_dir $(PRIVATE_RECOVERY_FONT_FILES_DIR) \
	    --resource_dir $(PRIVATE_RESOURCE_DIR) \
	    --output_file $(output_file) $(center_alignment) && \
	  $(PRIVATE_ZOPFLIPNG) -y --iterations=1 --filters=0 $(output_file) $(output_file) > /dev/null &&) true
else
RECOVERY_INSTALLING_TEXT_FILE :=
RECOVERY_INSTALLING_SECURITY_TEXT_FILE :=
RECOVERY_ERASING_TEXT_FILE :=
RECOVERY_ERROR_TEXT_FILE :=
RECOVERY_NO_COMMAND_TEXT_FILE :=
RECOVERY_CANCEL_WIPE_DATA_TEXT_FILE :=
RECOVERY_FACTORY_DATA_RESET_TEXT_FILE :=
RECOVERY_TRY_AGAIN_TEXT_FILE :=
RECOVERY_WIPE_DATA_CONFIRMATION_TEXT_FILE :=
RECOVERY_WIPE_DATA_MENU_HEADER_TEXT_FILE :=
endif # TARGET_RECOVERY_UI_SCREEN_WIDTH

```

主要是这段代码根据 PRIVATE_IMAGE_GENERATOR_JAR 宏的值来设置 recovery 的绘制图片的标准

```
java -jar $(PRIVATE_IMAGE_GENERATOR_JAR) \
	    --image_width $(PRIVATE_RECOVERY_IMAGE_WIDTH) \
	    --text_name $(text_name) \
	    --font_dir $(PRIVATE_RECOVERY_FONT_FILES_DIR) \
	    --resource_dir $(PRIVATE_RESOURCE_DIR) \
	    --output_file $(output_file) $(center_alignment) && \
	  $(PRIVATE_ZOPFLIPNG) -y --iterations=1 --filters=0 $(output_file) $(output_file) > /dev/null &&) true

PRIVATE_IMAGE_GENERATOR_JAR := $(image_generator_jar)

image_generator_jar := $(HOST_OUT_JAVA_LIBRARIES)/RecoveryImageGenerator.jar

```

这样的绘制功能是在 RecoveryImageGenerator.jar 中设置的参数来完成绘制  
st 下生成的 jar 包路径 out/host/linux-x86/framework/RecoveryImageGenerator.jar

查找 RecoveryImageGenerator 模块在源码中对应的路径

```
pathmod RecoveryImageGenerator

bootable/recovery/tools/image_generator

```

所以就来看 bootable/recovery/tools/image_generator/ImageGenerator.java 的源码就可以了

### 3.2ImageGenerator.java 关于绘制参数的相关源码分析

```
public static void main(String[] args)
            throws NumberFormatException, IOException, FontFormatException,
                    LocalizedStringNotFoundException {
        Options options = createOptions();
        CommandLine cmd;
        try {
            cmd = new GnuParser().parse(options, args);
        } catch (ParseException e) {
            System.err.println(e.getMessage());
            printUsage(options);
            return;
        }

        int imageWidth = Integer.parseUnsignedInt(cmd.getOptionValue("image_width"));

        if (cmd.hasOption("verbose")) {
            LOGGER.setLevel(Level.INFO);
        } else {
            LOGGER.setLevel(Level.WARNING);
        }

        ImageGenerator imageGenerator =
                new ImageGenerator(
                        imageWidth,
                        cmd.getOptionValue("text_name"),
                        DEFAULT_FONT_SIZE,
                        cmd.getOptionValue("font_dir"),
                        cmd.hasOption("center_alignment"));

        Set<String> localesSet = null;
        if (cmd.hasOption("locales")) {
            String[] localesList = cmd.getOptionValue("locales").split(",");
            localesSet = new HashSet<>(Arrays.asList(localesList));
            // Ensures that we have the default locale, all english translations are identical.
            localesSet.add("en-rAU");
        }
        Map<Locale, String> localizedStringMap =
                imageGenerator.readLocalizedStringFromXmls(cmd.getOptionValue("resource_dir"),
                        localesSet);
        imageGenerator.generateImage(localizedStringMap, cmd.getOptionValue("output_file"));
    }

```

在构造 ImageGenerator 中 DEFAULT_FONT_SIZE 就是默认字体的大小

```
private static final float DEFAULT_FONT_SIZE =40;

```

默认是 40px 的  
所以 修改为 60 编译查看果然大小已经改变了