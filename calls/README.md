# Voice and Video Calls · 语音与视频通话

[Home / 首页](../README.md) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍实时语音与视频通话的呼入、呼出及媒体收发，整理通话信令、参与者状态、协商、音视频链路、设备路由和通话质量等主题。

### 介绍范围

- 呼出与呼入流程：发起、来电、响铃、接受、拒绝、取消及挂断。
- 通话标识、参与者、设备选择和信令状态机。
- 呼叫建立、协商、超时、重连与通话结束原因。
- 媒体链路、直连或中继、NAT 穿透及候选地址研究。
- 音频采集、编码、传输、接收、解码与播放。
- 视频采集、编解码、分辨率变化和音视频同步。
- 静音、扬声器、蓝牙、音频路由与设备切换。
- 丢包、延迟、抖动、弱网与通话质量指标。
- 群组通话、多设备来电和通话记录。
- 与 RTP / RTCP、SRTP、ICE / STUN / TURN、WebRTC 等技术的关系核验。

上述标准与技术名称是待核验的研究主题；具体使用情况以目标客户端、版本和实际证据为准。

### 文档组织与扩展方式

本页是本主题的功能简介与导航入口。下方路径为后续文章规划，目前尚未创建；具体实现与结论以实际文章和验证记录为准。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="en"></a>

## English

### Overview

This directory introduces incoming and outgoing real-time voice and video calls and their media pipelines. Topics include signaling, participant state, negotiation, audio/video paths, device routing, and call quality.

### Scope

- Outgoing and incoming flows: initiation, incoming notification, ringing, acceptance, rejection, cancellation, and hangup.
- Call identifiers, participants, device selection, and signaling state machines.
- Call setup, negotiation, timeouts, reconnection, and termination reasons.
- Media paths, direct or relayed connections, NAT traversal, and address candidates.
- Audio capture, encoding, transmission, reception, decoding, and playback.
- Video capture, codecs, resolution changes, and audio/video synchronization.
- Mute, speakerphone, Bluetooth, audio routing, and device switching.
- Packet loss, latency, jitter, weak networks, and call-quality metrics.
- Group calls, multi-device ringing, and call history.
- Investigation of relationships to RTP / RTCP, SRTP, ICE / STUN / TURN, and WebRTC.

These standards and technology names identify research topics to verify. Their actual use must be established for the target client and version through evidence.

### Documentation Organization and Expansion

This page provides the topic overview and navigation entry point. The paths below are planned articles or subdirectories and have not been created yet. Implementations and findings are defined by the actual articles and their verification records.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

## 后续文章规划 / Planned Articles

| Proposed path | 中文内容 | English scope |
| --- | --- | --- |
| `outgoing-calls.md` | 呼出、响铃、接受、拒绝、取消与挂断 | Outgoing calls, ringing, acceptance, rejection, cancellation, and hangup |
| `incoming-calls.md` | 来电通知、设备选择、接听与拒接 | Incoming notifications, device selection, answering, and rejection |
| `signaling/` | 通话标识、参与者、状态机与结束原因 | Call IDs, participants, state machines, and termination reasons |
| `connectivity.md` | 链路协商、直连或中继与 NAT 穿透研究 | Path negotiation, direct/relayed connections, and NAT-traversal research |
| `audio-pipeline.md` | 音频采集、编码、发送、接收、解码与播放 | Audio capture, encoding, transmission, reception, decoding, and playback |
| `video-pipeline.md` | 视频采集、编解码、分辨率与音视频同步 | Video capture, codecs, resolution, and audio/video synchronization |
| `audio-routing.md` | 静音、扬声器、蓝牙与设备切换 | Mute, speakerphone, Bluetooth, and device switching |
| `quality-and-recovery.md` | 丢包、延迟、抖动、弱网、超时与重连 | Packet loss, latency, jitter, weak networks, timeouts, and reconnection |
| `group-calls.md` | 群组通话、多设备来电与通话记录 | Group calls, multi-device ringing, and call history |

## 相关主题 / Related Topics

[通信协议与数据格式 / Communication Protocols and Data Formats](../protocol/) · [端到端加密与密钥管理 / End-to-End Encryption and Key Management](../encryption/) · [图片、文件与语音消息 / Media Transfer and Voice Messages](../media/) · [多设备与数据同步 / Multi-Device and Data Synchronization](../multi-device/)
