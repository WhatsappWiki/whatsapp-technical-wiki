# Media Transfer and Voice Messages · 图片、文件与语音消息

[Home / 首页](../README.md) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍图片、视频、文档、贴纸和语音消息等可传输内容，整理媒体处理、上传下载、加解密、完整性校验及缓存。实时通话单独归入 calls 目录。

### 介绍范围

- 图片、视频、音频、文档、贴纸与语音消息的数据结构。
- 媒体预处理、缩略图、类型识别和文件元数据。
- 上传、下载、媒体引用与缓存生命周期。
- 媒体密钥、加解密、摘要和完整性校验研究。
- 语音消息录制、编码、时长、波形和播放状态。
- 下载失败、链接失效、缓存缺失和重新获取。

语音消息是可发送和下载的音频消息；实时语音通话在下一章单独整理。

### 文档组织与扩展方式

本页是本主题的功能简介与导航入口。下方路径为后续文章规划，目前尚未创建；具体实现与结论以实际文章和验证记录为准。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="en"></a>

## English

### Overview

This directory introduces transferable content such as images, videos, documents, stickers, and voice messages. It organizes media preparation, uploads and downloads, encryption, integrity checks, and caching. Real-time calls are covered separately in the calls directory.

### Scope

- Data structures for images, videos, audio, documents, stickers, and voice messages.
- Media preparation, thumbnails, type detection, and file metadata.
- Uploads, downloads, media references, and cache lifecycles.
- Media keys, encryption, decryption, hashes, and integrity verification.
- Voice-message recording, encoding, duration, waveforms, and playback state.
- Download failures, expired references, missing cached files, and retrieval.

Voice messages are audio messages that can be sent and downloaded. Real-time voice calls are covered separately below.

### Documentation Organization and Expansion

This page provides the topic overview and navigation entry point. The paths below are planned articles or subdirectories and have not been created yet. Implementations and findings are defined by the actual articles and their verification records.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

## 后续文章规划 / Planned Articles

| Proposed path | 中文内容 | English scope |
| --- | --- | --- |
| `media-types.md` | 媒体消息结构、类型与文件元数据 | Media message structures, types, and file metadata |
| `preprocessing.md` | 缩略图、类型识别与媒体预处理 | Thumbnails, type detection, and media preparation |
| `upload.md` | 上传流程、媒体引用与错误处理 | Upload flows, media references, and error handling |
| `download.md` | 下载流程、缓存读取与文件恢复 | Download flows, cached files, and retrieval |
| `media-encryption.md` | 媒体密钥、加解密、摘要与完整性 | Media keys, encryption, decryption, hashes, and integrity |
| `voice-messages.md` | 语音录制、编码、时长、波形与播放 | Voice recording, encoding, duration, waveforms, and playback |
| `cache-and-expiration.md` | 缓存生命周期、链接失效与重新获取 | Cache lifecycles, expired references, and retrieval |

## 相关主题 / Related Topics

[消息发送与接收 / Sending and Receiving Messages](../messaging/) · [端到端加密与密钥管理 / End-to-End Encryption and Key Management](../encryption/) · [语音与视频通话 / Voice and Video Calls](../calls/)
