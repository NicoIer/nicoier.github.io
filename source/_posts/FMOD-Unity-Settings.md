---
title: FMOD-Unity-Settings
date: 2025-07-10 18:16:15
tags: [Unity, FMOD, 音频]
---

# FMOD Unity Settings
[官方文档](https://fmod.com/docs/2.03/unity/settings.html)

FMOD作为一个跨平台的音频引擎，在深度使用中经常会遇到一些手机上出现，电脑上无法复现的问题。
这里记录一下一些比较重要的配置，可以解决一些问题。

![FMOD Settings.png](../img/FMOD%20Settings.png)


## Platform Specific / Android
-  OutputMode 这个选项是选择不同平台下使用的底层音频输出API 
    - Auto 这个选项会根据平台自动选择
    - No Sound 这个选项会禁用音频输出
    - Wav Writer 这个选项会将音频输出写入WAV文件
    - Java AudioTrack 这个选项会使用Android的AudioTrack API进行音频输出
    - OpenSL ES 这个选项会使用Android的OpenSL ES API进行音频输出
    - AAudio 这个选项会使用Android的AAudio API进行音频输出



- Virtual Channel Count  & Real Channel Count  这两个选项是设置Android平台下的虚拟通道数和实际通道数。虚拟通道数是指FMOD内部使用的通道数，实际通道数是指Android系统实际使用的通道数。一般情况下，虚拟通道数应该大于等于实际通道数。Virtual Channel Count 是指可以播放的通道数，一旦超过了实际通道数，最安静的通道将被虚拟化，即不可听见。Real Channel Count 是指实际可听见的通道数。降低这个计数将减少 FMOD 混音器的 CPU 使用率。但是请注意，降低通道数可能会导致音频质量下降，或者在某些情况下，音频可能会被截断或丢失。
f Channels that will be audible. Lowering this count will reduce the FMOD mixer's CPU usage. See the Virtual Voices white paper for more information.