---
title: 作って理解するコンテナ#1 - 導入編
tags:
  - container
  - Container-Runtime
private: false
updated_at: '2025-02-16T19:51:21+09:00'
id: f22832e7f412375ef8a5
organization_url_name: null
slide: false
ignorePublish: false
---
## シリーズ一覧
1. 作って理解するコンテナ#1 - 導入編　← 今回の記事
2. [作って理解するコンテナ#2 - プロセス編](https://qiita.com/pyxgun/items/341d2623a9d33f68e081)

## はじめに
DockerやPodman、Kubernetesなど、様々なコンテナ管理のツールがあり、アプリケーション開発基盤やCIにおけるRunnerなどでこれらのツールを使っている方はたくさんいるかと思います。  
では、これらが管理しているコンテナがどのように起動されているのか？イメージを利用してコンテナ起動するとは？といった、コンテナ"自体"の仕組みは知っていますか？  
筆者も以前はDockerを利用するのみで全く未知の世界でしたが、調べながら実際に実装をしてみると、普段利用しているDockerやk8sの中でどのような処理が行われているのか、なぜコンテナは軽量/高速といわれているのかなどが以前よりは深く理解できました。  
  
ほかの方やqiita以外でも実際のコードを記載したコンテナ実装の記事はありますが、部分部分にわかれていることが多く、xxの部分は実装できたけどyyの部分がわからないといったことが多発したり、そもそも記載が少ない部分もあったりします。  
一つ例をあげると、 **「コンテナ環境への外部からのアクセス」** を実装する例はあまり見当たりません。大体dockerコマンドの記事が引っかかります。  
**「コンテナ環境への外部からのアクセス」** を実現するための仕組みは、iptablesを利用したパケットフィルタ/ポートフォワーディングなど、一見コンテナとは関係のない別の仕組みを使っているため、なかなか検索してもたどり着かない、というかそもそもその仕組みを使ってたの？に気づくまでに時間がかかります。  
(ただ、本シリーズを通して実装してみると気づくと思いますが、コンテナ自体が様々な技術の組み合わせで実現しているので、「コンテナとは関係のない」という表現のほうが間違いです。)

上記のように、ある程度しっかりとしたコンテナ実装を進めようとすると、様々な記事を調べ、時にはコンテナの文字が一つもでない記事を読み、あれやこれやと組み合わせながら実装することになります。  
それはそれで楽しいですが、ちょっとだけ先人(になれたらいいな)の知恵として、筆者が今までに調べ/実装し/得た知識をもとに 「"ある程度形になる" コンテナランタイムを実装し、理解を深めよう」というのがテーマです。  

### 想定読者
本シリーズの想定読者は以下になります。
1. Docker/Podman/k8sなど、普段コンテナ管理ツールを使っている方
1. コンテナの仕組みを実際に作りながら知ってみたい方
1. Go言語を触ったことがある方

本シリーズはGo言語を利用して実装を進めていきます。実装を進めるうえで必要となる環境のセットアップや基本的なGo言語のビルド方法などは記載しますが、細かい言語仕様に関しては本シリーズでは取り扱いません。  
Go言語の経験がない方でも読み進めながら実装をすることができるようにはなっています(心がけて記載していきます)が、少しでも経験があるほうが読み進めやすいかもしれないです。  

### 本シリーズの最終ゴール
言ってしまえば「コンテナランタイムを実装していく」ようなものですので、ゴールを決めなければ永遠に実装が続いてしまいます。  
そのため、本シリーズは以下の機能を実装することを最終ゴールとします。

1. 名前空間を利用したプロセスの隔離
1. cgroup v2を利用したリソースの制限
1. コンテナ用ネットワーク環境の構築および外部アクセスの制御

Dockerだと当たり前に聞くイメージの管理やコンテナライフサイクルの管理(作成/起動/削除)などは、高レベルコンテナランタイムと呼ばれるものが行っているものであり、今回は実装対象外とします。  
※3. の外部アクセス制御も高レベルコンテナランタイムが行っていることがほとんどですが、今回はせっかくなので実装してみます。
言い換えると、今回のゴールは「たった一つのコンテナ環境をしっかりと作ってみる」です。

## 実装環境前提
本シリーズでは、以下の環境を前提として記載していきます。  
* 基盤: Hyper-V
* OS: Ubuntu 24.04.1 LTS
* Go: version 1.22.2

そのほかのOS/Goバージョンでも良いですが、動作は保障できないため悪しからず。  
※特にコンテナランタイムのような低レベルコーディングは、環境に大きく左右されやすいです。

基盤についてはHyper-VでもVMWareでも良いですが、WSLだけはWSL独自の仕様などもあり、あまり推奨しません。


## 環境準備
ここからは本シリーズを読みながら実装を進められるところまでの準備をします。  
想定読者にも記載はしましたが、なるべくGo言語に触れていない方でも読み進められるように、Go環境の導入部分を重点的に記載します。

### 1. Ubuntu環境
よしなにご準備ください。 (早速適当)

### 2. 必要なパッケージのインストール
以下のコマンドで、実装に必要となるパッケージをインストールします。※インストール済みのものもありますが念のため
```
sudo apt update
sudo apt install -y cgroup-tools iptables
sudo snap install go --classic
```
ちなみにGo言語は色々なインストール方法もありますが、環境変数のセットとか準備するまでに時間がかかる場合があります。  
バージョンにこだわりがない方や初めてGo言語を触る方は、上記のコマンドでインストールすることを推奨します。


### 3. Go言語のインストール確認
Go言語が正しくインストールできたか、以下のコマンドで確認します。
```
go version
```
`go version go1.22.2 linux/amd64` のようにバージョンが表示されれば、正常にインストールが完了です。


### 4. ディレクトリ構成
本シリーズでは、メインとなるコンテナランタイムの実装のほか、要素技術の仕組みを見るコードも実装していきます。  
そのため、以下のようなディレクトリ構成をおすすめします。
```
.
├── container/
└── workspace/
```
本シリーズ内では、`container`ディレクトリを「実装用ディレクトリ」、`workspace`ディレクトリを「作業用ディレクトリ」と呼びます。  
「作業用ディレクトリ内でmain.goファイルを作成」と記載されていた場合、`workspace/main.go`という形になります。  


### 5. Go言語でHello World (Go言語初めての方向け)
コンテナ実装に入る前に、Go言語を初めて触る方向けにHello Worldする方法を説明します。  

#### 1. モジュールの初期化
Goはコンパイラ言語なので、Pythonと異なりいわゆるパッケージの所在などを明確にする必要があります。  
そのため、Go言語で実装する場合は作業ディレクトリ内で必ずモジュールの初期化をします。 ※これをしないと一向に動作しません。
作業用ディレクトリ内で、以下のコマンドを実行します。
```
go mod init container-tsukuru
```
`container-tsukuru` のところは、任意のプロジェクト名なので、好きなものを入力してください。  
コマンドを実行すると、`go.mod` というファイルが作られていると思います。これがあればモジュールの初期化に成功です。

#### 2. コードを書いてみる
詳細な言語仕様の説明は省きますが、Go言語はコンパイラ言語なので、型宣言が必要だったり必要なかったり(?)します。  
あえてGo言語になれるように、ちょっとだけ複雑なHello Worldをしてみます。

ファイル名は `main.go` として、以下のコードを実装します。
```go:main.go
package main  // 今回のパッケージ名は必ず main にしてください

import "fmt"

func generateMessage(name, phrase string) string {
	return fmt.Sprintf("Hi %s, I want to say \"%s\" to you!", name, phrase)
}

func echoMessage(message string) {
	fmt.Println(message)
}

func main() {
	var (
		name   string = "Qiita"
		phrase string = "Hello World"
	)
	message := generateMessage(name, phrase)
	echoMessage(message)
}
```

Pythonしか触れたことがない方だと独特な書き方に見えるかもしれないですし、C/Java等のコンパイラ言語を経験された方は似ているけどちょっと違うな、と思うかもしれないです。  
(sprintfとかfprintfもあります、C言語やってた方はイメージしやすいですね)

関数の引数や戻り値に型を宣言する必要があるのはコンパイラ言語の特徴ですが、`message := generateMessage()` のように型を宣言しなくても良い仕様もあります。(関数の戻り値からの型推論)  
インタプリタ言語とコンパイラ言語が混ざった感じですね。

#### 3. 実行してみる
Go言語で作成したコードは、以下のコマンドで実行できます。
```
go run .
```
`Hi Qiita, I want to say "Hello World" to you!` と表示されれば成功です。

#### 4. ビルドしてみる
コンパイラ言語なのにコンパイルしないのか、と思うかも知れないですが、ビルドして実行ファイルを作成することもできます。
```
go build .
```
これで、作業ディレクトリ内に[1. モジュールの初期化](#1-モジュールの初期化)で指定したプロジェクト名と同じ実行ファイルが作成されていると思います。  
実行ファイルなので、以下のように実行できます。
```
./container-tsukuru
```

## 次回予告
今回は導入編なのでコンテナに関する話は全くしていませんが、次回は「プロセス編」です。  
Go言語に初めて触れている方は、上記例のほかにもいくつかコードを書いて慣れておいてもらうと良いかもしれないです。  

また、はじめにも記載した通り筆者も絶賛コンテナランタイムを開発中です。
今回対象外である高レベルコンテナランタイムの実装やREST APIによるCLIツールからのリクエスト、
リモートノードのコンテナ管理など様々な機能を実装しています。  

本シリーズを読み進めていく中で対象外の機能の実装に興味を持たれた方は、ぜひ確認してみてもらえればと思います。  

[Karakuri - Container runtime for small-scale development environments.](https://github.com/pyxgun/karakuri)
