+++
title = 'Xps13-9340 在 Arch linux 上的摄像头驱动配置'
date = 2026-05-06T09:33:39+08:00
description = "记录在 Xps13-9340 上配置 ipu6 摄像头(ov02c10) 并成功点亮的过程"
tags = ["玩机","XPS 13-9340","Arch Linux","摄像头","IPU6","Wayland","Shell"]
+++

## 风险提示

作者对 linux 的驱动层并不了解，以下折腾过程多数由 codex 辅助完成，可能并非最佳实践，此文的目的只是为了记录折腾过程，方便自己原样复刻，折腾的时间点是 2026 年 5 月，linux 内核版本为 7.0.3，桌面环境为 KDE Plasma / Wayland 。

## 目标

让 Xps 13-9340 的内置摄像头的 RGB 输出在 Arch linux 下可用( IR / Windows hello 实在没能力折腾了)，并且最终可被识别为 web camera。

## 结果以及路线

可用，不过没做到 web camera 按需启用。

大致路线： `AUR Intel IPU6 + HAL + icamerasrc + v4l2loopback` 

## 最初状态

通过 `ArchInstall` 脚本安装完系统，并安装完 KDE 环境后，主线内核已经可以识别到传感器，但是用户态没有可用画面(黑帧)。

关键现象如下:

1. 传感器成功识别 `OVTI02C1:00 -> ov02c10`
2. `libcamera` 能枚举到相机，但是抓一帧拿到的是纯黑帧，抓取时摄像头隐私灯会亮。
3. 内核错误关键字

```
intel_ipu6_isys.isys intel_ipu6.isys.40: csi2-4 error: Frame sync error
int3472-discrete INT3472:0c: GPIO type 0x02 unknown; the sensor may not work
ov02c10 ... supply dovdd/avdd/dvdd not found, using dummy regulator
```

## 想法

经过我和 gemini 还有 gpt 的查询，最终我认为在主线上折腾希望比较渺茫，查询到的资料中虽然没有直接指向从 aur 这条路完全点亮的，但是多少有点进度，所以决定从 aur 入手去折腾。

## 需要成功安装与配置的包

1. `intel-ipu6-dkms-git`
2. `intel-ipu6-camera-bin`
3. `intel-ipu6-camera-hal-git`
4. `icamerasrc-git`
5. `v4l2loopback-dkms`
6. `v4l2-relayd`

本质上就是

```bash
yay -S intel-ipu6-dkms-git intel-ipu6-camera-bin intel-ipu6-camera-hal-git icamerasrc-git v4l2loopback-dkms v4l2-relayd
```

当然，只是安装成功才是第一步


## 安装

1. `intel-ipu6-dkms-git` 没声明前置依赖，补上 `linux-headers` 等依赖
   
```bash
sudo pacman -S linux-headers
sudo dkms autoinstall
```

补上再安装后， `dkms status` 应该能看到类似输出

```
ipu6-drivers/r263.51fe72485, 7.0.3-arch1-2, x86_64: installed (Original modules exist)
```

2. `intel-ipu6-camera-hal-git` 在 GCC 16 下编译失败

需要更改  `~/.cache/yay/intel-ipu6-camera-hal-git/PKGBUILD` 的 `build()` 方法：

- 每次先删旧 build 目录
- 给 C / C++ 都加针对 unused warning 的 `Wno-error`

```bash
build() {
    rm -rf "$_pkgname/build"
    cmake -B "$_pkgname/build" -S "$_pkgname"       \
        -DCMAKE_BUILD_TYPE=Release      \
        -DCMAKE_INSTALL_PREFIX="/usr"   \
        -DCMAKE_INSTALL_LIBDIR="lib"    \
        -DBUILD_CAMHAL_ADAPTOR=ON       \
        -DBUILD_CAMHAL_PLUGIN=ON        \
        -DIPU_VERSIONS="ipu6;ipu6ep;ipu6epmtl" \
        -DUSE_PG_LITE_PIPE=ON \
        -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
        -DCMAKE_C_FLAGS="-Wno-error=unused-but-set-variable -Wno-error=unused-variable" \
        -DCMAKE_CXX_FLAGS="-Wno-error=unused-but-set-variable -Wno-error=unused-variable"
    cmake --build "$_pkgname/build"
}
```

然后不要直接再 `yay -S` 刷新 PKGBUILD，直接在缓存目录里构建：

```bash
cd ~/.cache/yay/intel-ipu6-camera-hal-git
makepkg -si
```

## 配置

**警告: 操作配置前先备份，防止出问题，如果你是自上而下查看本文，可以先往下看，不要忙着操作，看完后再开始操作，此处的配置只保证你能点亮一次，如果完全按照这样配置，后面还有其他坑等着你。**

1. 让 HAL profile 正确命中传感器。
   
现状：

HAL 没命中 `OV02C10` 反而退回 `AR0234_TGL_10bits.aiqb` ，关键报错:

```
CameraId=0 failed to open libcamhal device
CamHAL[ERR] MediaControl init failed
```

查找配置发现系统里其实已有目标传感器的配置文件 

- `/etc/camera/ipu6epmtl/libcamhal_profile.xml`
- `/etc/camera/ipu6epmtl/sensors/ov02c10-uf.xml`
- `/etc/camera/ipu6epmtl/OV02C10_*.aiqb`
- `/etc/camera/ipu6epmtl/gcss/graph_settings_OV02C10_*.xml`


解决方案

编辑 `/etc/camera/ipu6epmtl/libcamhal_profile.xml`

改成只保留：

```xml
<availableSensors value="ov02c10-uf-4"/>
```
<details>
<summary>点击展开编辑前后文件对比</summary>

编辑前

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- Copyright (C) 2024-2025 Intel Corporation.

     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     See the License for the specific language governing permissions and
     limitations under the License.
-->

<CameraSettings>
    <Common>
        <version value="1.0"/>
        <platform value="IPU6"/>
        <!-- The value format of availableSensors is "sensor name"-wf/uf-"CSI port ID". -->
        <availableSensors value="ov05c10-uf-0,ov05c10-uf-4,ov08x40-uf-0,ov08x40-uf-4,
                                 ov13b10-uf-0,ov13b10-wf-4,ov5675-uf-4,ov01a1s-uf-0,
                                 ov01a10-uf-0,ov01a10-uf-4,
                                 ov02c10-uf-0,ov02c10-uf-1,ov02c10-uf-4,
                                 ov02e10-uf-1,ov02e10-uf-4,
                                 hm2170-uf-0,hm2170-uf-1,hm2170-uf-4,
                                 hm2172-uf-1,hm2172-uf-4,hi556-uf-1,
                                 imx390-1-0,imx390-2-0,imx390-3-0,imx390-4-0,
                                 imx390-5-4,imx390-6-4,ar0234-1-0,ar0234-2-4,
                                 isx031-1-0,isx031-2-0,isx031-3-4,isx031-4-4,
                                 lt6911uxc,lt6911uxe-1-0,lt6911uxe-2-4,
                                 external_source,ar0234_usb"/>
    </Common>
</CameraSettings>
```

编辑后

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- Copyright (C) 2024-2025 Intel Corporation.

     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     See the License for the specific language governing permissions and
     limitations under the License.
-->

<CameraSettings>
    <Common>
        <version value="1.0"/>
        <platform value="IPU6"/>
        <!-- The value format of availableSensors is "sensor name"-wf/uf-"CSI port ID". -->
        <availableSensors value="ov02c10-uf-4"/>
    </Common>
</CameraSettings>
```

</details>


2. 编辑本机专用 sensor profile

默认 `ov02c10-uf.xml` 还是会让 HAL 自动推导 `$I2CBUS` / `$CSI_PORT` / `$CAPTURE_ID`，而它在这台机器上似乎有问题。

处理方式是用本机专用写死版本覆盖

**后文发现这里还会有其他坑，想要原样复刻的读者不必急于按文章复刻，可继续往后阅读，阅读完成后再着手配置**

用 `media-ctl -p` 看 IPU6 的媒体图
找到实际启用的链路：
`ov02c10 ... -> Intel IVSC CSI -> Intel IPU6 CSI2 4 -> Intel IPU6 ISYS Capture 32`

编辑 `/etc/camera/ipu6epmtl/sensors/ov02c10-uf.xml`

- 写死真实 sensor 名字：`ov02c10 17-0036`
- 写死真实 CSI：`Intel IPU6 CSI2 4`
- 写死真实 capture：`Intel IPU6 ISYS Capture 32`
- 只保留一套调参：`OV02C10_1BG203N3_ADL.aiqb`

<details>

<summary>点击展开编辑前后文件对比</summary>

编辑前

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- Copyright (C) 2022-2023 Intel Corporation.

     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     See the License for the specific language governing permissions and
     limitations under the License.
-->

<CameraSettings>
    <Sensor name="ov02c10-uf" description="OV02C10 sensor.">
        <MediaCtlConfig id="0" ConfigMode="AUTO" outputWidth="1928" outputHeight="1092" format="V4L2_PIX_FMT_SGRBG10"><!-- RAW10 BE capture -->
            <format name="ov02c10 $I2CBUS" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI-2 $CSI_PORT" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI2 BE SOC 0" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>

            <link srcName="ov02c10 $I2CBUS" srcPad="0" sinkName="Intel IPU6 CSI-2 $CSI_PORT" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI-2 $CSI_PORT" srcPad="1" sinkName="Intel IPU6 CSI2 BE SOC 0" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI2 BE SOC 0" srcPad="1" sinkName="Intel IPU6 BE SOC capture 0" sinkPad="0" enable="true"/>

            <videonode name="ov02c10 $I2CBUS" videoNodeType="VIDEO_PIXEL_ARRAY"/>
            <videonode name="Intel IPU6 CSI-2 $CSI_PORT" videoNodeType="VIDEO_ISYS_RECEIVER"/>
            <videonode name="Intel IPU6 BE SOC capture 0" videoNodeType="VIDEO_GENERIC"/>
        </MediaCtlConfig>
        <MediaCtlConfig id="0" mediaCfg="1" ConfigMode="AUTO" outputWidth="1928" outputHeight="1092" format="V4L2_PIX_FMT_SGRBG10"><!-- RAW10 BE capture -->
            <format name="ov02c10 $I2CBUS" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI2 $CSI_PORT" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>

            <link srcName="ov02c10 $I2CBUS" srcPad="0" sinkName="Intel IPU6 CSI2 $CSI_PORT" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI2 $CSI_PORT" srcPad="1" sinkName="Intel IPU6 ISYS Capture $CAPTURE_ID" sinkPad="0" enable="true"/>

            <videonode name="ov02c10 $I2CBUS" videoNodeType="VIDEO_PIXEL_ARRAY"/>
            <videonode name="Intel IPU6 CSI2 $CSI_PORT" videoNodeType="VIDEO_ISYS_RECEIVER"/>
            <videonode name="Intel IPU6 ISYS Capture $CAPTURE_ID" videoNodeType="VIDEO_GENERIC"/>
        </MediaCtlConfig>

        <StaticMetadata>
            <!-- format,widthxheight,field(none:0,alternate:7),mcId -->
            <supportedStreamConfig value="V4L2_PIX_FMT_NV12,1920x1080,0,0,
                                          V4L2_PIX_FMT_NV12,1280x720,0,0,
                                          V4L2_PIX_FMT_NV12,640x480,0,0,
                                          V4L2_PIX_FMT_NV12,640x360,0,0"/>

            <supportedFeatures value="MANUAL_EXPOSURE,
                                      MANUAL_WHITE_BALANCE,
                                      IMAGE_ENHANCEMENT,
                                      NOISE_REDUCTION,
                                      PER_FRAME_CONTROL,
                                      SCENE_MODE"/>
            <supportedAeExposureTimeRange value="AUTO,10,1000000"/> <!--scene_mode,min_exposure_time,max_exposure_time -->
            <supportedAeGainRange value="AUTO,0,60"/> <!--scene_mode,min_gain,max_gain -->
            <fpsRange value="15,15,15,30,30,30"/>
            <evRange value="-6,6"/>
            <evStep value="1,3"/>
            <supportedAeMode value="AUTO,MANUAL"/>
            <supportedVideoStabilizationModes value="OFF"/>
            <supportedSceneMode value="NORMAL"/>
            <supportedAntibandingMode value="AUTO,50Hz,60Hz,OFF"/>
            <supportedAwbMode value="AUTO,INCANDESCENT,FLUORESCENT,DAYLIGHT,FULL_OVERCAST,PARTLY_OVERCAST,SUNSET,VIDEO_CONFERENCE,MANUAL_CCT_RANGE,MANUAL_WHITE_POINT,MANUAL_GAIN,MANUAL_COLOR_TRANSFORM"/>
            <supportedAfMode value="OFF"/>
        </StaticMetadata>

        <supportedTuningConfig value="NORMAL,VIDEO,OV02C10_1BG203N3_ADL,STILL_CAPTURE,VIDEO,OV02C10_1BG203N3_ADL" />

        <!-- The lard tags configuration. Every tag should be 4-characters. -->
        <!-- <TuningMode, cmc tag, aiq tag, isp tag, others tag>  -->
        <lardTags value="VIDEO,DFLT,DFLT,DFLT,DFLT"/>

        <supportedISysSizes value="1928x1092"/> <!-- ascending order request -->
        <supportedISysFormat value="V4L2_PIX_FMT_SGRBG10"/>
        <enableAIQ value="true"/>
        <iSysRawFormat value="V4L2_PIX_FMT_SGRBG10"/>
        <pSysFormat value="V4L2_PIX_FMT_NV12"/>
        <initialSkipFrame value="0"/>
        <exposureLag value="2"/>
        <gainLag value="2"/>
        <ltmGainLag value="1"/>
        <yuvColorRangeMode value="full"/> <!-- there are 2 yuv color range mode, like full, reduced. -->
        <graphSettingsFile value="graph_settings_OV02C10_1BG203N3_ADL.xml"/>
        <graphSettingsType value="dispersed"/>
        <enablePSysProcessor value="true"/>
        <dvsType value="IMG_TRANS"/>
        <testPatternMap value="Off,0,ColorBars,2"/>
        <enableAiqd value = "true"/>
        <useCrlModule value = "false"/>
        <maxRequestsInflight value="5"/>
        <supportPrivacy value="0"/> <!-- privacy mode, 0: off, 1: CVF privacy 2: AE-based privacy -->
    </Sensor>
    <Sensor name="ov02c10-uf" description="OV02C10 sensor.">
        <MediaCtlConfig id="0" ConfigMode="AUTO" outputWidth="1928" outputHeight="1092" format="V4L2_PIX_FMT_SGRBG10"><!-- RAW10 BE capture -->
            <format name="ov02c10 $I2CBUS" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI-2 $CSI_PORT" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI2 BE SOC 0" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>

            <link srcName="ov02c10 $I2CBUS" srcPad="0" sinkName="Intel IPU6 CSI-2 $CSI_PORT" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI-2 $CSI_PORT" srcPad="1" sinkName="Intel IPU6 CSI2 BE SOC 0" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI2 BE SOC 0" srcPad="1" sinkName="Intel IPU6 BE SOC capture 0" sinkPad="0" enable="true"/>

            <videonode name="ov02c10 $I2CBUS" videoNodeType="VIDEO_PIXEL_ARRAY"/>
            <videonode name="Intel IPU6 CSI-2 $CSI_PORT" videoNodeType="VIDEO_ISYS_RECEIVER"/>
            <videonode name="Intel IPU6 BE SOC capture 0" videoNodeType="VIDEO_GENERIC"/>
        </MediaCtlConfig>
        <MediaCtlConfig id="0" mediaCfg="1" ConfigMode="AUTO" outputWidth="1928" outputHeight="1092" format="V4L2_PIX_FMT_SGRBG10"><!-- RAW10 BE capture -->
            <format name="ov02c10 $I2CBUS" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI2 $CSI_PORT" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>

            <link srcName="ov02c10 $I2CBUS" srcPad="0" sinkName="Intel IPU6 CSI2 $CSI_PORT" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI2 $CSI_PORT" srcPad="1" sinkName="Intel IPU6 ISYS Capture $CAPTURE_ID" sinkPad="0" enable="true"/>

            <videonode name="ov02c10 $I2CBUS" videoNodeType="VIDEO_PIXEL_ARRAY"/>
            <videonode name="Intel IPU6 CSI2 $CSI_PORT" videoNodeType="VIDEO_ISYS_RECEIVER"/>
            <videonode name="Intel IPU6 ISYS Capture $CAPTURE_ID" videoNodeType="VIDEO_GENERIC"/>
        </MediaCtlConfig>

        <StaticMetadata>
            <!-- format,widthxheight,field(none:0,alternate:7),mcId -->
            <supportedStreamConfig value="V4L2_PIX_FMT_NV12,1920x1080,0,0,
                                          V4L2_PIX_FMT_NV12,1280x720,0,0,
                                          V4L2_PIX_FMT_NV12,640x480,0,0,
                                          V4L2_PIX_FMT_NV12,640x360,0,0"/>

            <supportedFeatures value="MANUAL_EXPOSURE,
                                      MANUAL_WHITE_BALANCE,
                                      IMAGE_ENHANCEMENT,
                                      NOISE_REDUCTION,
                                      PER_FRAME_CONTROL,
                                      SCENE_MODE"/>
            <supportedAeExposureTimeRange value="AUTO,10,1000000"/> <!--scene_mode,min_exposure_time,max_exposure_time -->
            <supportedAeGainRange value="AUTO,0,60"/> <!--scene_mode,min_gain,max_gain -->
            <fpsRange value="15,15,15,30,30,30"/>
            <evRange value="-6,6"/>
            <evStep value="1,3"/>
            <supportedAeMode value="AUTO,MANUAL"/>
            <supportedVideoStabilizationModes value="OFF"/>
            <supportedSceneMode value="NORMAL"/>
            <supportedAntibandingMode value="AUTO,50Hz,60Hz,OFF"/>
            <supportedAwbMode value="AUTO,INCANDESCENT,FLUORESCENT,DAYLIGHT,FULL_OVERCAST,PARTLY_OVERCAST,SUNSET,VIDEO_CONFERENCE,MANUAL_CCT_RANGE,MANUAL_WHITE_POINT,MANUAL_GAIN,MANUAL_COLOR_TRANSFORM"/>
            <supportedAfMode value="OFF"/>
        </StaticMetadata>

        <supportedTuningConfig value="NORMAL,VIDEO,OV02C10_1SG204N3_ADL,STILL_CAPTURE,VIDEO,OV02C10_1SG204N3_ADL" />

        <!-- The lard tags configuration. Every tag should be 4-characters. -->
        <!-- <TuningMode, cmc tag, aiq tag, isp tag, others tag>  -->
        <lardTags value="VIDEO,DFLT,DFLT,DFLT,DFLT"/>

        <supportedISysSizes value="1928x1092"/> <!-- ascending order request -->
        <supportedISysFormat value="V4L2_PIX_FMT_SGRBG10"/>
        <enableAIQ value="true"/>
        <iSysRawFormat value="V4L2_PIX_FMT_SGRBG10"/>
        <pSysFormat value="V4L2_PIX_FMT_NV12"/>
        <initialSkipFrame value="0"/>
        <exposureLag value="2"/>
        <gainLag value="2"/>
        <ltmGainLag value="1"/>
        <yuvColorRangeMode value="full"/> <!-- there are 2 yuv color range mode, like full, reduced. -->
        <graphSettingsFile value="graph_settings_OV02C10_1SG204N3_ADL.xml"/>
        <graphSettingsType value="dispersed"/>
        <enablePSysProcessor value="true"/>
        <dvsType value="IMG_TRANS"/>
        <testPatternMap value="Off,0,ColorBars,2"/>
        <enableAiqd value = "true"/>
        <useCrlModule value = "false"/>
        <maxRequestsInflight value="5"/>
        <supportPrivacy value="0"/> <!-- privacy mode, 0: off, 1: CVF privacy 2: AE-based privacy -->
    </Sensor>
    <Sensor name="ov02c10-uf" description="OV02C10 sensor.">
        <MediaCtlConfig id="0" ConfigMode="AUTO" outputWidth="1928" outputHeight="1092" format="V4L2_PIX_FMT_SGRBG10"><!-- RAW10 BE capture -->
            <format name="ov02c10 $I2CBUS" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI-2 $CSI_PORT" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI2 BE SOC 0" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>

            <link srcName="ov02c10 $I2CBUS" srcPad="0" sinkName="Intel IPU6 CSI-2 $CSI_PORT" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI-2 $CSI_PORT" srcPad="1" sinkName="Intel IPU6 CSI2 BE SOC 0" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI2 BE SOC 0" srcPad="1" sinkName="Intel IPU6 BE SOC capture 0" sinkPad="0" enable="true"/>

            <videonode name="ov02c10 $I2CBUS" videoNodeType="VIDEO_PIXEL_ARRAY"/>
            <videonode name="Intel IPU6 CSI-2 $CSI_PORT" videoNodeType="VIDEO_ISYS_RECEIVER"/>
            <videonode name="Intel IPU6 BE SOC capture 0" videoNodeType="VIDEO_GENERIC"/>
        </MediaCtlConfig>
        <MediaCtlConfig id="0" mediaCfg="1" ConfigMode="AUTO" outputWidth="1928" outputHeight="1092" format="V4L2_PIX_FMT_SGRBG10"><!-- RAW10 BE capture -->
            <format name="ov02c10 $I2CBUS" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI2 $CSI_PORT" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>

            <link srcName="ov02c10 $I2CBUS" srcPad="0" sinkName="Intel IPU6 CSI2 $CSI_PORT" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI2 $CSI_PORT" srcPad="1" sinkName="Intel IPU6 ISYS Capture $CAPTURE_ID" sinkPad="0" enable="true"/>

            <videonode name="ov02c10 $I2CBUS" videoNodeType="VIDEO_PIXEL_ARRAY"/>
            <videonode name="Intel IPU6 CSI2 $CSI_PORT" videoNodeType="VIDEO_ISYS_RECEIVER"/>
            <videonode name="Intel IPU6 ISYS Capture $CAPTURE_ID" videoNodeType="VIDEO_GENERIC"/>
        </MediaCtlConfig>

        <StaticMetadata>
            <!-- format,widthxheight,field(none:0,alternate:7),mcId -->
            <supportedStreamConfig value="V4L2_PIX_FMT_NV12,1920x1080,0,0,
                                          V4L2_PIX_FMT_NV12,1280x720,0,0,
                                          V4L2_PIX_FMT_NV12,640x480,0,0,
                                          V4L2_PIX_FMT_NV12,640x360,0,0"/>

            <supportedFeatures value="MANUAL_EXPOSURE,
                                      MANUAL_WHITE_BALANCE,
                                      IMAGE_ENHANCEMENT,
                                      NOISE_REDUCTION,
                                      PER_FRAME_CONTROL,
                                      SCENE_MODE"/>
            <supportedAeExposureTimeRange value="AUTO,10,1000000"/> <!--scene_mode,min_exposure_time,max_exposure_time -->
            <supportedAeGainRange value="AUTO,0,60"/> <!--scene_mode,min_gain,max_gain -->
            <fpsRange value="15,15,15,30,30,30"/>
            <evRange value="-6,6"/>
            <evStep value="1,3"/>
            <supportedAeMode value="AUTO,MANUAL"/>
            <supportedVideoStabilizationModes value="OFF"/>
            <supportedSceneMode value="NORMAL"/>
            <supportedAntibandingMode value="AUTO,50Hz,60Hz,OFF"/>
            <supportedAwbMode value="AUTO,INCANDESCENT,FLUORESCENT,DAYLIGHT,FULL_OVERCAST,PARTLY_OVERCAST,SUNSET,VIDEO_CONFERENCE,MANUAL_CCT_RANGE,MANUAL_WHITE_POINT,MANUAL_GAIN,MANUAL_COLOR_TRANSFORM"/>
            <supportedAfMode value="OFF"/>
        </StaticMetadata>

        <supportedTuningConfig value="NORMAL,VIDEO,OV02C10_CIFME14_ADL,STILL_CAPTURE,VIDEO,OV02C10_CIFME14_ADL" />

        <!-- The lard tags configuration. Every tag should be 4-characters. -->
        <!-- <TuningMode, cmc tag, aiq tag, isp tag, others tag>  -->
        <lardTags value="VIDEO,DFLT,DFLT,DFLT,DFLT"/>

        <supportedISysSizes value="1928x1092"/> <!-- ascending order request -->
        <supportedISysFormat value="V4L2_PIX_FMT_SGRBG10"/>
        <enableAIQ value="true"/>
        <iSysRawFormat value="V4L2_PIX_FMT_SGRBG10"/>
        <pSysFormat value="V4L2_PIX_FMT_NV12"/>
        <initialSkipFrame value="0"/>
        <exposureLag value="2"/>
        <gainLag value="2"/>
        <ltmGainLag value="1"/>
        <yuvColorRangeMode value="full"/> <!-- there are 2 yuv color range mode, like full, reduced. -->
        <graphSettingsFile value="graph_settings_OV02C10_CIFME14_ADL.xml"/>
        <graphSettingsType value="dispersed"/>
        <enablePSysProcessor value="true"/>
        <dvsType value="IMG_TRANS"/>
        <testPatternMap value="Off,0,ColorBars,2"/>
        <enableAiqd value = "true"/>
        <useCrlModule value = "false"/>
        <maxRequestsInflight value="5"/>
        <supportPrivacy value="0"/> <!-- privacy mode, 0: off, 1: CVF privacy 2: AE-based privacy -->
    </Sensor>
</CameraSettings>
```

编辑后

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<CameraSettings>
    <Sensor name="ov02c10-uf" description="OV02C10 sensor for Dell XPS 13 9340.">
        <MediaCtlConfig id="0" mediaCfg="1" ConfigMode="AUTO" outputWidth="1928" outputHeight="1092" format="V4L2_PIX_FMT_SGRBG10">
            <format name="ov02c10 17-0036" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>
            <format name="Intel IPU6 CSI2 4" pad="0" width="1928" height="1092" format="V4L2_MBUS_FMT_SGRBG10_1X10"/>

            <link srcName="ov02c10 17-0036" srcPad="0" sinkName="Intel IPU6 CSI2 4" sinkPad="0" enable="true"/>
            <link srcName="Intel IPU6 CSI2 4" srcPad="1" sinkName="Intel IPU6 ISYS Capture 32" sinkPad="0" enable="true"/>

            <videonode name="ov02c10 17-0036" videoNodeType="VIDEO_PIXEL_ARRAY"/>
            <videonode name="Intel IPU6 CSI2 4" videoNodeType="VIDEO_ISYS_RECEIVER"/>
            <videonode name="Intel IPU6 ISYS Capture 32" videoNodeType="VIDEO_GENERIC"/>
        </MediaCtlConfig>

        <StaticMetadata>
            <supportedStreamConfig value="V4L2_PIX_FMT_NV12,1920x1080,0,0,
                                          V4L2_PIX_FMT_NV12,1280x720,0,0,
                                          V4L2_PIX_FMT_NV12,640x480,0,0,
                                          V4L2_PIX_FMT_NV12,640x360,0,0"/>

            <supportedFeatures value="MANUAL_EXPOSURE,
                                      MANUAL_WHITE_BALANCE,
                                      IMAGE_ENHANCEMENT,
                                      NOISE_REDUCTION,
                                      PER_FRAME_CONTROL,
                                      SCENE_MODE"/>
            <supportedAeExposureTimeRange value="AUTO,10,1000000"/>
            <supportedAeGainRange value="AUTO,0,60"/>
            <fpsRange value="15,15,15,30,30,30"/>
            <evRange value="-6,6"/>
            <evStep value="1,3"/>
            <supportedAeMode value="AUTO,MANUAL"/>
            <supportedVideoStabilizationModes value="OFF"/>
            <supportedSceneMode value="NORMAL"/>
            <supportedAntibandingMode value="AUTO,50Hz,60Hz,OFF"/>
            <supportedAwbMode value="AUTO,INCANDESCENT,FLUORESCENT,DAYLIGHT,FULL_OVERCAST,PARTLY_OVERCAST,SUNSET,VIDEO_CONFERENCE,MANUAL_CCT_RANGE,MANUAL_WHITE_POINT,MANUAL_GAIN,MANUAL_COLOR_TRANSFORM"/>
            <supportedAfMode value="OFF"/>
        </StaticMetadata>

        <supportedTuningConfig value="NORMAL,VIDEO,OV02C10_1BG203N3_ADL,STILL_CAPTURE,VIDEO,OV02C10_1BG203N3_ADL" />
        <lardTags value="VIDEO,DFLT,DFLT,DFLT,DFLT"/>
        <supportedISysSizes value="1928x1092"/>
        <supportedISysFormat value="V4L2_PIX_FMT_SGRBG10"/>
        <enableAIQ value="true"/>
        <iSysRawFormat value="V4L2_PIX_FMT_SGRBG10"/>
        <pSysFormat value="V4L2_PIX_FMT_NV12"/>
        <initialSkipFrame value="0"/>
        <exposureLag value="2"/>
        <gainLag value="2"/>
        <ltmGainLag value="1"/>
        <yuvColorRangeMode value="full"/>
        <graphSettingsFile value="graph_settings_OV02C10_1BG203N3_ADL.xml"/>
        <graphSettingsType value="dispersed"/>
        <enablePSysProcessor value="true"/>
        <dvsType value="IMG_TRANS"/>
        <testPatternMap value="Off,0,ColorBars,2"/>
        <enableAiqd value = "true"/>
        <useCrlModule value = "false"/>
        <maxRequestsInflight value="5"/>
        <supportPrivacy value="0"/>
    </Sensor>
</CameraSettings>

```

</details>


3. 修复 `PSYS` 权限问题

上面的两个配置改完以后，`icamerasrc` 不再直接报 `MediaControl init failed`，但是依然起不来，会继续报一个新的错误：

```text
Failed to open PSYS, error: Permission denied
```

这时候问题已经不是传感器 profile 没命中了，而是普通用户没有权限访问 `ipu-psys` 这个设备节点。进一步确认问题所在。

- `/dev/ipu-psys0` 的权限是 `root:video 660`
- 当前用户不在 `video` 组
- `/dev/media0` 这种节点有 `uaccess`
- 但是 `/dev/ipu-psys0` 默认没有被自动补 ACL

加一条 `udev` 规则，给普通用户权限。

新建：

```bash
/etc/udev/rules.d/71-intel-ipu6-psys-uaccess.rules
```

内容：

```
SUBSYSTEM=="intel-ipu6-psys", TAG+="uaccess"
```

然后重新加载规则：

```bash
sudo udevadm control --reload
sudo udevadm trigger --action=add --sysname-match=ipu-psys0
```

配置到这一步以后，普通用户执行：

```bash
GST_DEBUG=2 gst-launch-1.0 -e icamerasrc num-buffers=1 ! fakesink
```

应该可以不需要 `sudo` 直接成功。

做到上面这一步以后，严格来说相机已经不是完全不能用的状态了。

`icamerasrc` 能顺利起流，可以抓到一帧有效数据，不再是之前主线路线下那种纯黑空帧，说明 `Intel HAL + icamerasrc` 这条链路本身已经没什么问题了。

4. 桥接成普通 web camera

为了让浏览器等东西可用，我们需要把 `icamerasrc` 的输出桥接成一个普通的 `/dev/video*`。

这时候就需要：

- `v4l2loopback-dkms`
- `v4l2-relayd`

实际用下来，后者并不是必须非得先上，`gst-launch` 直接桥也可以。

先起一个 loopback 设备

找一个空闲的编号来启动，我在这里选用了 `60`

```bash
sudo modprobe -r v4l2loopback
sudo modprobe v4l2loopback video_nr=60 card_label="Intel-IPU6-Camera" exclusive_caps=1
```

确认成功的话，可以看到：

```bash
cat /sys/class/video4linux/video60/name
```

输出类似：

```text
Intel-IPU6-Camera
```

接下来运行下面的命令，开始最终的尝试。

```bash
gst-launch-1.0 icamerasrc \
  ! video/x-raw,format=NV12,width=1280,height=720,framerate=30/1 \
  ! videoconvert \
  ! video/x-raw,format=YUY2,width=1280,height=720,framerate=30/1 \
  ! v4l2sink device=/dev/video60 sync=false
```

如果没报错，恭喜你

```bash
ffprobe /dev/video60
```

应该能看到类似：

```
  Stream #0:0: Video: rawvideo (YUY2 / 0x32595559), yuyv422, 1280x720, 442368 kb/s, 30 fps, 30 tbr, 1000k tbn, start 1.846525
```

此时，随便打开一个摄像头测试 web 页面，你会发现已经是可用状态了。


## 书接上文

你看这文章这么长，就知道如果不出意外的话，就要出意外了，对吧。

还记得我们最开始在 `ov02c10-uf.xml` 里写死了几个值吗

- `ov02c10 17-0036`
- `Intel IPU6 CSI2 4`
- `Intel IPU6 ISYS Capture 32`

当我想当然的准备把上面那个桥接做一个 systemd，以后按需启动的时候，事情突然变得有趣起来，写完 systemd 之后， start stop，完美运行，重启测测呢？

### 新发现

那么多铺垫过去了，你肯定知道事情不出意外的出意外了吧，的确，重启后用不了了，看一下日志吧。

```text
Failed to find DevName for cameraId: 0, get video node: ov02c10 17-0036, devname: /dev/v4l-subdev1
setup Link ov02c10 17-0036 [-1:0] ==> Intel IPU6 CSI2 4 [...] enable 1 failed.
set MediaCtlConf McLink failed
set up mediaCtl failed
```

怎么还找不到了，那再看看设备是怎么个事吧

```bash
sudo media-ctl -p -d /dev/media0
```

结果发现，`CSI2 4` 和 `Capture 32` 都没变，真正变掉的是传感器 entity 名字：

```text
ov02c10 17-0036  ->  ov02c10 16-0036
```

这下麻烦了，前面就是因为动态找 entity 名称有问题，我们才在 xml 配置中写死我们的传感器的，结果重启了名称变化，我们的 HAL 又炸了。

接下来的问题就从如何点亮变成了：

- 为什么同一台机器同一颗 `ov02c10`，entity 名字里的 bus 会漂移
- 能不能让 `ipu6-camera-hal` 对这种漂移更宽容

### Patch HAL

看标题你也知道了，这次我们要去动 HAL 层了，在动之前，先确认一下本地 AUR 缓存里的源码和上游是不是一致，免得 patch 半天最后发现上游早就改了，看一下最后一个 commit hash，发现是同样的  `9899efa`。

先定位到真正出问题的位置：

```cpp
src/iutils/Utils.cpp
CameraUtils::getDeviceName()
```

话不多说，patch 一下

- 保留原本的精确匹配
- 但当目标是 sensor subdev，而且精确匹配失败时
- 增加一个 fallback
- 只按 sensor 前缀去匹配，忽略后面漂移的 bus/address


```patch
diff --git a/src/iutils/Utils.cpp b/src/iutils/Utils.cpp
index dccd841..76a5ef2 100644
--- a/src/iutils/Utils.cpp
+++ b/src/iutils/Utils.cpp
@@ -584,6 +584,19 @@ void CameraUtils::getDeviceName(const char* entityName, string& deviceNodeName,
     const char* dirPath = "/sys/class/video4linux/";
     if (isSubDev) filePrefix = "v4l-subdev";
 
+    string expectedName = entityName ? entityName : "";
+    string expectedSensorPrefix;
+    if (isSubDev && expectedName.rfind("Intel ", 0) != 0) {
+        string::size_type sep = expectedName.find(' ');
+        if (sep != string::npos && sep + 1 < expectedName.size() &&
+            expectedName.find('-', sep + 1) != string::npos) {
+            // Sensor subdev names often take the form "<sensor> <bus>-<addr>".
+            // Match by sensor prefix as a fallback so media entity bus renumbering
+            // does not break profile lookup.
+            expectedSensorPrefix = expectedName.substr(0, sep);
+        }
+    }
+
     DIR* dp = opendir(dirPath);
     CheckAndLogError((dp == nullptr), VOID_VALUE, "@%s, Fail open : %s", __func__, dirPath);
 
@@ -610,11 +623,21 @@ void CameraUtils::getDeviceName(const char* entityName, string& deviceNodeName,
             } else {
                 len = 0;
             }
-            if (len == (int)strlen(entityName) && memcmp(buf, entityName, len) == 0) {
+            if (len == static_cast<int>(expectedName.size()) &&
+                memcmp(buf, expectedName.c_str(), len) == 0) {
                 deviceNodeName = "/dev/";
                 deviceNodeName += dirp->d_name;
                 break;
             }
+
+            if (!expectedSensorPrefix.empty() && deviceNodeName.empty()) {
+                string actualName(buf, len);
+                if (actualName.rfind(expectedSensorPrefix + " ", 0) == 0 &&
+                    actualName.find('-', expectedSensorPrefix.size() + 1) != string::npos) {
+                    deviceNodeName = "/dev/";
+                    deviceNodeName += dirp->d_name;
+                }
+            }
         }
     }
     closedir(dp);
```

我把这个 patch 存放在 

```bash
~/.cache/yay/intel-ipu6-camera-hal-git/src/0001-relax-sensor-subdev-name-matching.patch
```

之后修改 

```
~/.cache/yay/intel-ipu6-camera-hal-git/PKGBUILD
```

修改关键点

```bash
pkgver() {
    cd $_pkgname
    printf "r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
}

prepare() {
    cd "$_pkgname"
    patch -Np1 -i "${srcdir}/0001-relax-sensor-subdev-name-matching.patch"
}

build() {
    rm -rf "$_pkgname/build"
    cmake -B "$_pkgname/build" -S "$_pkgname"       \
        -DCMAKE_BUILD_TYPE=Release      \
        -DCMAKE_INSTALL_PREFIX="/usr"   \
        -DCMAKE_INSTALL_LIBDIR="lib"    \
        -DBUILD_CAMHAL_ADAPTOR=ON       \
        -DBUILD_CAMHAL_PLUGIN=ON        \
        -DIPU_VERSIONS="ipu6;ipu6ep;ipu6epmtl" \
        -DUSE_PG_LITE_PIPE=ON \
        -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
        -DCMAKE_C_FLAGS="-Wno-error=unused-but-set-variable -Wno-error=unused-variable" \
        -DCMAKE_CXX_FLAGS="-Wno-error=unused-but-set-variable -Wno-error=unused-variable"
    cmake --build "$_pkgname/build"
}

package() {
    DESTDIR="$pkgdir" cmake --install "$_pkgname/build"
}
```

先不急着桥接，先做一个最小验证：

```bash
GST_DEBUG=2 gst-launch-1.0 -e icamerasrc num-buffers=1 ! fakesink
```

结果是：

- patch 前，它会在 `ov02c10 17-0036` 这里失败
- patch 后，即使这次系统实际枚举出来的是 `ov02c10 16-0036`
- 这个命令也可以顺利跑到 `EOS`

这就说明，这个 patch 生效了。

###  systemd

到这里，其实已经没有什么写出来的意义了，能踩的坑都踩过了，剩下的东西一马平川

目标:

平时不常驻桥接，写一个 systemd 手动控制，需要用的时候启动，停掉以后真实摄像头就不再工作，隐私灯也会跟着灭掉，也算是某种“硬件”级隐私开关了。


最终我拆成了三部分：

1. 一个真正执行桥接的脚本
2. 一个 `systemd service`
3. 一条给虚拟摄像头补 `uaccess` 的 `udev` 规则

首先写他个脚本来桥接并保存到以下地址：

```bash
/usr/local/bin/xps13-camera-bridge.sh
```

内容

```bash
#!/usr/bin/env bash
set -euo pipefail

exec gst-launch-1.0 \
  icamerasrc \
  ! video/x-raw,format=NV12,width=1280,height=720,framerate=30/1 \
  ! videoconvert \
  ! video/x-raw,format=YUY2,width=1280,height=720,framerate=30/1 \
  ! v4l2sink device=/dev/video60 sync=false
```

接下来配置 Systemd unit

```bash
/etc/systemd/system/xps13-camera-bridge.service
```

内容如下：

```ini
[Unit]
Description=Dell XPS 13 9340 IPU6 camera bridge
After=systemd-logind.service

[Service]
Type=simple
ExecStartPre=-/usr/bin/modprobe -r v4l2loopback
ExecStartPre=/usr/bin/modprobe v4l2loopback video_nr=60 card_label=Intel-IPU6-Camera exclusive_caps=1
ExecStartPre=/usr/bin/udevadm settle
ExecStart=/usr/local/bin/xps13-camera-bridge.sh
ExecStopPost=-/usr/bin/modprobe -r v4l2loopback
Restart=no

[Install]
WantedBy=multi-user.target
```


### 验证 systemd 

启动：

```bash
sudo systemctl start xps13-camera-bridge.service
```

看状态：

```bash
sudo systemctl status xps13-camera-bridge.service
```

确认设备：

```bash
cat /sys/class/video4linux/video60/name
ffprobe /dev/video60
```

停止：

```bash
sudo systemctl stop xps13-camera-bridge.service
```

记得观察一下隐私灯符合不符合我们的预期，按理说应该不会有问题。


### 再来看看 xml 呢

到这里其实就有一个新的问题了。

我们既然已经 patch 了 HAL，那 `/etc/camera/ipu6epmtl/sensors/ov02c10-uf.xml` 里原来写死的 sensor entity 名字，其实就不再显得那么合理了。

现在更值得尝试的事情反而变成了：

把 `ov02c10 17-0036` 这类写死值，慢慢恢复成动态形式

反正能用都能用了，就先告一段落吧。


## 后记

避雷 xps，thinkpad 这种假的 linux 支持吧，离了 oem 内核什么都不是，我两年前就尝试过去点亮这个破摄像头，不出意外的失败了。折腾这些玩意真的是心累，什么时候在大陆能买到 framework 就好了。

骂归骂，又水了一个 [RR](https://github.com/intel/ipu6-camera-hal/pull/173) ，如果上游收了的话，可能以后折腾这个的人能少踩一点坑吧。
