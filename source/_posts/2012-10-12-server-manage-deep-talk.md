---
title: '&#8220;Server Manage Deep Talk #1&#8243; の振り返りメモ'
date: 2012-10-12 00:00:00 +0900
author: hatak
layout: post
permalink: /2012/10/12/17536
categories:
  - report
tags:
  - infra
  - memo
  - server
---

先日、&#8221;[Server Manage Deep Talk #1][1]&#8221; に参加してきました。

YAPC の懇親会での話がきっかけで @riywo さんに企画していただいたイベントでした。趣旨としては、既成のツールに焦点を絞るのではなく、サーバを効率的（かつ、可能であれば統一的）に管理するにはどうすればいいかというざっくりとした情報交換の場だったのですが、各社の苦悩が感じられてとても面白いものでした。

そのあと、ちょっと自分なりに考えてみたので覚え書きしてみます。

<!--more-->

今回の話を踏まえると、最初に考えなければならないのは大きく2つの点だと思います。 （解釈間違えなどあればご指摘ください。。）

### 「管理」の対象がどこまでなのか

サーバ管理といっても様々な管理があります。

* 物理と論理
    * 物理 : どのデータセンタ/ラック/スロットにあるか
    * 論理 : どのセグメントでどのように利用されているか
* 資産と運用
    * 資産 : 固定資産の減価償却やリース期間 (業務監査のイメージ)
    * 運用 : どのサービスにどういった用途で利用しているか

このほかにも、オンプレミス/クラウド(VM含)やスペック毎/役割毎など、いろんな分類基準があるわけです。全てをカバーするのも一つの選択肢ですが、ここは目的から軸となる区分を明確化しておく必要がありそうです。 @kentaro さんがおっしゃられていたように、まさに利用シーンの現状や想定を整理した上で、実際どのあたりを吸収するか考える必要があるのだろうと思います。

### データを集約するか、proxy するか

@riywo さんのアイディアでは、既存ツールのデータを活かすために Core が欲しいデータを持っている各ツールに問い合わせて返ってきた情報を集約する形になっています。 一方で、@kuwa_tw さんからの意見で全部まとめた箱としてしまうというアイディアもあり、これもありだと思います。

それぞれのメリットを考えてみます。

* 集約式
    * データが一カ所にまとめられる
    * 他サービスへの依存が無い
    * ここだけ変更すればいい
* proxy式
    * 既存ツールはそのままにできる
    * Core の機能はシンプルに抑えられる

これは現状の利用状況に依るのかも知れないですね。ツールがどの程度(種類・頻度)利用されていて移行のコストがどれくらいかかるのか、あるいは部署を横断的にまとめていくことが社内政治的に可能なのかどうか、そもそも現状のデータをどれくらい Core が吸収できるのか、といったあたりが焦点になる気がします。

私自身の利用経験や環境から考えると、集約してしまう方式が敷居は低そうかなと感じました。連携のために既存ツールに手を入れなければならないのは導入コストとして高くつくかなと。（でも、結局はバランスを見てのハイブリッドになりそう。。）

自分なりにちょっとずつ整理しながら、ひとまずはシンプルなモックを作ってみようかなと思っています。

次回、が行えるようであればさらに掘り下げて、少しでもインフラ担当者が楽になれると幸せですね。 あと、Web サービス事業者だけに限らず、ホスティングなどでまとまった台数を管理していそうな DC 事業者さんとかの話も伺ってみたいと思いました。

 [1]: https://www.facebook.com/events/346462502114960/