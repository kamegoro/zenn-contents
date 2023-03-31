---
title: "WebRTC スタートガイド"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["WebRTC"]
published: false
---

## はじめに

初めまして、かめごろと申します。
最近、WebRTC について学習してみました。

## 参考

https://webrtcforthecurious.com/ja/

## TL;DR

-

## WebRTC とは

WebRTC は、**W**eb **R**eal**T**ime **C**ommunication の略称で、API であると同時にプロトコルでもあります。

WebRTC プロトコルとは、双方向の安全にかつ、実際の通信を始める前に、前提条件の確認とかをして通信相手と認識合わせを行うことの一連の流れがまとめられたルールです。

WebRTC API は開発者が WebRTC プロトコルを使用するためのものであり、JavaScript のみで規定されています。
ただ、JavaScript 以外の言語でも WebRTC プロトコルであれば利用することができます。

下記の、ドキュメントでそれぞれ管理されています。

- [WebRTC プロトコル（IETF rtcweb）](https://datatracker.ietf.org/wg/rtcweb/documents/)
- [WebRTC API（W3C webrtc）](https://www.w3.org/TR/webrtc/)

### 歴史

参考

### WebRTC を使っているサービス

- Google Meet
- webex
- Discord
- Microsoft Teams
- zoom
- NeWork

### WebRTC 普及前（2011 年以前）

- Skype
- LINE

VoIP/VoLTE という技術を使って映像・音声通話を行なっていた。

### TCP/IP

コンピューター同士が位置情報（IP）を使って通信を行う際、レスポンスする際も送信元の IP を利用します。

### RTP、RTPC とは

RTP とは、TCP/IP ネットワーク上で音声や動画のように連続するデータの流れをリアルタイムに伝送するための通信プロトコル（通信手順）の一つ。最初の規格は IETF によって 1996 年に RFC 1889 として勧告され、2003 年に暗号化などの規定を追加した RFC 3550 によって置き換えられた。

### WebRTC との違い
