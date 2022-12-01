---
title: "Cluster Version Operator"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
# はじめに
OpenShift Advent Calendar #2 の1日目です。

OpenShiftをインストールすると、構築直後からたくさんのPodが動いています。
手元の構築直後のOpenShiftクラスター(AWS上のv4.11)を見てみると、Namespaceが67個あり、その中でPodが200個動いていました (DaemonSet等がいるので、ノード数によって多少変わります)。

```console
$ oc get ns --no-headers | wc -l
67
$ oc get pod -A --no-headers | grep Running | wc -l
200
```

名前を見ただけで何をするPodかだいたいわかるものもあれば、何をしているのかよくわからないPodもいます。今年のAdvent Calendarで、OpenShiftの基盤を構成するPodを紹介していきたいと思います。

このOperator紹介シリーズの内容は、[こちら](https://rheb.hatenablog.com/entry/unknownpod)に書いた方法を使って調べました。

# Cluster Version Operator
初回はCluster Version Operatorを取り上げます。OpenShiftのドキュメントでは「CVO」と略されていることもあります。

