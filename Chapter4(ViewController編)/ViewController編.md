# iOSアプリでスケールしやすいアーキテクチャを考えてみた④-ViewController編

この一連の記事では私的に考えたスケールしやすいアーキテクチャを紹介します。  
記事全体の構成(予定)は以下の通りです。  
(1)設計を理解するためのレイヤードアーキテクチャ編  
(2)設計を理解するためのクリーンアーキテクチャ編  
(3)アーキテクチャ概要編  
(4)**ViewController編←本記事**  
(5)View/Alert編(準備中)  
(6)画面遷移編(準備中)  
(7)ViewModel(Controller/Presenter)編(準備中)  
(8)UseCaseとエラー編(準備中)  
(9)UseCaseとアプリケーションの状態管理編(準備中)  
(10)Repository編(準備中)  
(11)Domain編(準備中)  
(12)Web API/データベース編(準備中)  
(13)その他(準備中)  

本記事ではViewControllerの設計を説明します。  
基本的な設計理論の知識を前提としているので、そこから知りたいという方はレイヤードアーキテクチャ編、もしくはクリーンアーキテクチャ編から読むことをオススメします。  
またアーキテクチャの全体像を知りたい人はアーキテクチャ概要編から読むことをオススメします。  

## 前提
- この記事の設計とはアプリケーションに関するものでライブラリ等の設計は想定していません。  
- SwiftUIは扱いません。  
- 作成したサンプルプロジェクトはMVVMをベースに考えていますが、記事内容はどんなアーキテクチャでも共通する考えとなっているはずです。  
- FluxやReduxのアーキテクチャは概念としては触れる予定ですが、サンプルプロジェクトでは採用されていません。  

## 前回までの内容と本記事の内容
前回までの記事で設計の概論、そしてそれを踏まえた本記事で提案しているアーキテクチャの概要を説明しました。  
この記事ではそのアーキテクチャのViewControllerの設計について説明します。  

## Swift UI時代にViewControllerを学ぶ意義
この記事を読んでくれている人の中には「設計は興味あるけど、今更ViewControllerかあ...」と思っている方もいると思います。  
私もその意見に心の底から同意しますが、それでもモチベーションを持ってこの記事を読んで欲しいので最初に私なりのViewControllerの設計を学ぶ意義を述べます。    
### ViewControllerを採用したのは、ただの偶然
そもそもこの記事の中でViewControllerを採用している理由はただ「このスケールしやすいアーキテクチャの探究を始めた時、SwiftUIがまだ登場して間もなかった」からというだけであって、私も意図的にViewControllerを採用したわけではありません。  
今から始めるならば間違いなくSwiftUIでやっていました。  

### それでも今からViewControllerで設計を学ぶ意義
しかし今から振り返ると今回の試みの中ViewControllerを採用したのは決して悪いことではなかったと考えています。  
その理由はまず、設計をしっかりと理解していくとViewControllerかSwift UIか、またiOSかAndroidかといった開発におけるプラットフォームはあまり関係ないということを強く感じるからです。  
もちろんViewControllerとSwift UIのプログラムの記述の仕方で違ったり、AndroidではViewの実装でxmlを直接操作したりと細かいところで独自に学ばなければいけないことはあります。  
ただ設計というものは本質的にプラットフォームには依存しておらず、その素材にViewController、SwiftUIのどちらを利用しようが、しっかりと学んでいけばそこで得られたノウハウは他のプラットフォームでも充分に活きると思います。  
またSwiftUIに対するiOSプログラマの所感を見る限り、SwiftUIでの開発が主流になっても当分(2021年5月時点)ViewControllerが不必要になるわけではなさそうです。  
そのため長期的にiOSアプリ開発を学ぼうと考えているのならば、設計を学ぶ際に直感的でシンプルに記述できるSwiftUIよりもより原始的で複雑なViewControllerを素材に学んだ方がそこで得た知見は後々の他のプラットフォームで開発する際にも応用が効くように思います。(しかし私はSwiftUIやAndroidもしっかり経験しておらず、あくまで私個人の感覚的な意見です。)  


## ViewControllerの基本
それでは内容に入っていきますが、まず最初にViewControllerの基本的な情報について整理します。  
参考にしたのはAppleが開発者向けに公開している[UIViewController](https://developer.apple.com/documentation/uikit/uiviewcontroller)と[View Controller Programming Guide for iOS](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/index.html#//apple_ref/doc/uid/TP40007457-CH2-SW1)という2つのドキュメントです。  

### View Controllerの責務
[ドキュメント](https://developer.apple.com/documentation/uikit/uiviewcontroller)を見ると、ViewControllerの主な責務以下4点だと書かれています。(複数あるため原文も載せると見づらくなってしまうため、ここでは意訳のみ載せます。)  
1. Viewに紐づいているデータの変化に応じたViewの更新
2. Viewの操作への対応
3. Viewの大きさの変更、また画面全体のレイアウトの管理
4. (他のViewControllerも含めた)他のオブジェクトとの連携

このようにViewControllerの責務を総体的に見ると、ViewControllerの責務は文字通り「Viewの管理(Control)」であり、Viewとは直接的に関係ないデータの操作がその主な責務には含まれないことがわかると思います。  
**4(他のViewControllerも含めた)他のオブジェクトとの連携**はいささか示している範囲が広いようには感じますが、これは恐らくアプリ内におけるViewControllerの重要度の高さから通知機能(NotificationCenter)等、Viewとは直接関係ないオブジェクトとの連携も行う場合もあるため抽象的に表現しているのだと推測しています。  
なのでやはりViewController全体の責務を考えるとその要は「Viewに関係した責務」であると思います。

### 2種類のViewController
ViewControllerの責務は先程の4点なのですが、それでもその種類は利用ケースによって大きく2種類に分かれます。  
1. Content ViewController: 自身に紐づいた**View**を管理することを責務としたViewController
2. Container ViewController: 自身の画面内で表示される複数の**ViewController(Container ViewControllerの文脈ではChild ViewControllerと言います)**の管理を責務としたViewController

#### Content ViewControllerとContainer ViewControllerの違い
大きな違いはViewController内でUIButtonやUILabel等具体的なViewを操作するかどうかいう点でしょうか。  
Content ViewControllerでは自身に表示されたUIButtonやUILabelといったViewを直接操作するのに対して、Container ViewControllerでは自身が管理するViewControllerとその親View(Root View)のみを操作するため具体的なViewを操作することはありません。  
一般的にアプリ内では独自に定義するViewControllerはContent ViewControllerだと思います。  
Container ViewControllerはあまり頻繁に独自で定義することはないと思いますが、しかし私たちがアプリ内でよく利用するNavigation ControllerやTab Bar ControllerはこのContainer ViewControllerに当たります。  

#### 記事で扱うのはContent ViewControllerのみ
この記事で扱うのはこのうちContent ViewControllerに限定されます。  
Container ViewControllerの設計において何より重要なのは、**自身のChild ViewControllerとなるContent ViewControllerへの干渉を最低限とすること**にあります。  
これは言い換えると、Container ViewControllerの設計において重要なのはまず各Child ViewController(Content ViewController)の設計であるということです。  
各Child ViewController(Content ViewController)さえしっかりとできていれば、Container ViewControllerがやるべきことはChild ViewController間の連携くらいであり、その具体的な方法に関して特に設計論の観点から説明することはないと考えています。(Child ViewControllerの親Viewの管理も通常のViewの管理と同じです。)  
そのためこの記事ではContent ViewControllerの設計に限定して話を進めます。  

## 現実のViewControllerの開発で起こる問題
ドキュメントに基づいてViewControllerの基本について触れましたが、ここでは現実にViewControllerの開発(コードを読む、書く)で起こる問題点について説明します。  
私は典型的な手法でViewControllerを開発すると以下の問題が起こると考えています。  

1. 「Viewの管理(Control)」と一言でいえど、実際には多種多様なプログラムが書かれる
2.  手続的なプログラミングと宣言的なプログラミングが混同する
3.  ViewControllerによってプログラムのサイズが大きく異なる(対応する画面の仕様に大きく影響を受ける)

### 1.「Viewの管理(Control)」と一言でいえど、実際には多種多様なプログラムが書かれる
記事冒頭の「ViewControllerの基本」では、Appleのドキュメントを引用してViewControllerの主要な責務を4点紹介し、それらはさらに「Viewの管理」と要約できると説明しました。  
この要約はViewControllerにおけるいくつかの細かな責務を捨象してはいるものの、その全体の概要を理解するのには非常に役に立つと私は思います。  
しかし概念上そのように簡潔にまとめることはできても実際のコードレベルのViewControllerには、Viewの操作、アニメーション、遷移処理、アラート操作、CollectionViewのデリゲート・データソース、その他オブジェクトの連携等、実にさまざまな処理が書かれており、中には簡単な状態変数の管理もViewControllerが行っているケースも見かけます。  
このような雑多にも思えるコードが書かれているViewControllerにおいて、MVCやレイヤードアーキテクチャといった設計理論の基礎を理解できている開発者であればそれらをメタ認識にすることでViewControllerの責務を理解することはできると思いますが、そうでない開発者にとっては**ViewControllerの責務がなんであるのか理解しづらい**というのが第一の問題点です。  
恐らくそのように設計理論の理解よりも先にコード(実務)を通してプログラミングを学んでいる開発者からすると上記のような処理が含まれているViewControllerは「Viewを管理する」存在ではなく、　「画面の開発における便利屋」的な存在だと認識されているように思います。  
私はレイヤードアーキテクチャの記事でFatViewControllerの問題の原因としてViewController-Model間の関係の論理的な理解不足があると指摘しましたが、このようにコードレベルにおいてViewControllerが雑多に思える処理を抱えている状況はそうしたViewController-Model間の関係(またはViewControllerに何を書いてはいけないか)を理解することを難しくさせていると思います。  







