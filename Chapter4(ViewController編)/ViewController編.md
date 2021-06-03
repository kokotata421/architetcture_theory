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
私もその意見に心の底から同意します。  
ただそれでもモチベーションを持ってこの記事を読んで欲しいので最初に私なりのViewControllerの設計を学ぶ意義を述べます。    
### ViewControllerを採用したのは、ただの偶然
そもそもこの記事の中でViewControllerを採用している理由はただ「このスケールしやすいアーキテクチャの探究を始めた時、SwiftUIがまだ登場して間もなかった」からというだけであって、私も意図的にViewControllerを採用したわけではありません。  
今から始めるならば間違いなくSwiftUIでやっていました。  

### それでも今からViewControllerで設計を学ぶ意義
しかし今から振り返ると今回の試みの中ViewControllerを採用したのは決して悪いことではなかったと考えています。  
その理由はまず、設計をしっかりと理解していくとViewControllerかSwift UIか、またiOSかAndroidかといった開発におけるプラットフォームはあまり関係ないということを強く感じるからです。  
もちろんViewControllerとSwift UIのプログラムの記述の仕方で違ったり、AndroidではViewの実装でxmlを直接操作したりと細かいところで独自に学ばなければいけないことはあります。  
ただ設計というものは本質的にプラットフォームには依存しておらず、その素材にViewController、SwiftUIのどちらを利用しようが、しっかりと学んでいけばそこで得られたノウハウは他のプラットフォームでも充分に活きると思います。  
またSwiftUIに対するiOSプログラマの所感を見る限り、SwiftUIでの開発が主流になっても当分(2021年5月時点)UIKitが完全に不必要になるわけではなさそうです。  
そのような状況を踏まえても長期的にプログラミング開発を学ぼうと考えているのならば、設計を学ぶ際に直感的でシンプルに記述できるSwiftUIではなく原始的で複雑なUIKit/ViewControllerを素材にすることによって、そこで得た知見を後々の他のプラットフォームで開発する際に活かせる場面が出てくると思います。(しかし私はSwiftUIやAndroidもしっかり経験しておらず、あくまで私個人の感覚的な意見です。)  


## ViewControllerの基本
それでは内容に入っていきますが、まず最初にViewControllerの基本的な情報について整理します。  
参考にしたのはAppleが開発者向けに公開している[UIViewController](https://developer.apple.com/documentation/uikit/uiviewcontroller)と[View Controller Programming Guide for iOS](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/index.html#//apple_ref/doc/uid/TP40007457-CH2-SW1)という2つのドキュメントです。  

### View Controllerの責務
[UIViewControllerのドキュメント](https://developer.apple.com/documentation/uikit/uiviewcontroller)を見ると、ViewControllerの主な責務以下4点だと書かれています。<sup>[*1](#footnote1)</sup>  
1. Viewに紐づいているデータの変化に応じたViewの更新
2. Viewの操作への対応
3. Viewの大きさの変更、また画面全体のレイアウトの管理
4. (他のViewControllerも含めた)他のオブジェクトとの連携

このようにViewControllerの責務を総体的に見ると、ViewControllerの責務は文字通り「Viewの管理(Control)」であり、Viewとは直接的に関係のないデータの操作がその主な責務には含まれないことがわかると思います。  
**4.(他のViewControllerも含めた)他のオブジェクトとの連携**はいささか示している責務範囲が広いようには感じますが、これは恐らくアプリ内におけるViewControllerの重要度の高さから通知機能(NotificationCenter)等、Viewとは直接関係ないオブジェクトとの連携も行う場合もあるため具体的に責務を定義できず「他のオブジェクトとの連携」となってしまったのだと推測しています。  
ただ上記で挙げたNotificationCenter等一部例外はあるものの、ViewController全体を考えるとその主要な責務は**Viewに関連した処理**であると理解して問題ないと思います。  

### 2種類のViewController
ViewControllerの責務は先程の4点なのですが、それでもその種類は利用ケースによって大きく2つに分かれます。  
1. Content ViewController: 自身に紐づいた**Viewを管理**することを責務としたViewController
2. Container ViewController: 自身の画面内で表示される複数の**ViewController&#40;Container ViewControllerの文脈ではChild ViewControllerと言う&#41;の管理**を責務としたViewController

#### Content ViewControllerとContainer ViewControllerの違い
大きな違いはViewController内で具体的なViewを操作するかどうかという点でしょうか。  
Content ViewControllerでは自身に表示されたUIButtonやUILabelといったViewを直接操作するのに対して、Container ViewControllerでは自身が管理するViewControllerとその親View(Root View)のみを操作するためUIButtonやUILabelといったViewを直接操作することはありません。  
一般的にアプリ内で独自に定義するViewControllerのほとんどはContent ViewControllerだと思います。<sup>[*2](#footnote2)</sup>  
Container ViewControllerはあまり頻繁に独自で定義することはないと思いますが、ただ私たちがアプリ内でよく利用するNavigation ControllerやTab Bar ControllerはContainer ViewControllerに該当します。  
#### 記事で扱うのはContent ViewControllerのみ
この記事で扱うのはこのうちContent ViewControllerに限定されます。  
Container ViewControllerの設計において何より重要なのは、**自身のChild ViewControllerとなるContent ViewControllerへの干渉を最低限とすること**にあります。  
これは言い換えると、Container ViewControllerの設計において重要なのはまず各Child ViewController(Content ViewController)の設計であるということです。  
各Child ViewController(Content ViewController)さえしっかりとできていれば、Container ViewControllerがやるべきことはChild ViewController間の連携を管理するくらいであり、その具体的な方法に関して特に今回の設計論の観点から説明することはないと考えています。(Conatainer ViewControllerで操作するChild ViewControllerの親Viewの管理も通常のViewの管理と同じです。)  
そのためこの記事ではContent ViewControllerの設計に限定して話を進めます。  

## 現実のViewControllerの開発で起こる問題
ドキュメントに基づいてViewControllerの基本について触れましたが、ここでは現実にViewControllerの開発(コードを読む、書く)で起こる問題点について説明します。  
私は典型的な手法でViewControllerを開発すると以下の問題が起こると考えています。  

1. 「Viewの管理(Control)」と一言でいえど、実際には多種多様なプログラムが書かれる
2.  命令的プログラミングと宣言的プログラミングが混在する
3.  ViewControllerによってプログラムのサイズが大きく異なる(対応する画面の仕様に大きく影響を受ける)

### 1.「Viewの管理(Control)」と一言でいえど、実際には多様なプログラムが書かれる
#### コードレベルでViewControllerの責務が理解しにくい
記事冒頭の「ViewControllerの基本」では、Appleのドキュメントを引用してViewControllerの主要な責務を4点紹介し、それらはさらに「Viewの管理」と要約できると説明しました。  
ViewControllerの責務は概念上は上記のように簡潔にまとめることが可能ですが、実際のコードレベルのViewControllerにはViewの操作、アニメーション、遷移処理、アラート操作、CollectionViewのデリゲート・データソース、その他オブジェクトの連携等、実にさまざまな処理が書かれており、中には簡単な状態変数の管理もViewControllerが行っているケースも見かけます。  
このように雑多にも思えるコードが書かれているViewControllerにおいて、MVCやレイヤードアーキテクチャといった設計理論の基礎を理解できている開発者であればメタ認識にすることでその責務を理解することはできますが、そうでない開発者にとっては**コードレベルからViewControllerの責務を正しく理解するのは難しい**と思います。  
恐らく設計理論の理解よりも先にコード(実務)を通してプログラミングを学んでいる開発者からすると上記のように様々な処理が含まれているViewControllerは「Viewを管理する」存在ではなく、「画面開発における便利屋」的な存在だと認識されてしまうことが多いのではないでしょうか。   
私はレイヤードアーキテクチャの記事でFatViewControllerの問題の原因としてViewController-Model間の関係の論理的な理解不足を指摘しましたが、このようにコードレベルにおいてViewControllerが雑多に思える処理を抱えている状況はそうしたViewController-Model間の関係(またはViewControllerに何を書いてはいけないか)を理解することをより難しくさせていると考えています。  

#### ViewControllerには確立されたコード形式がない
またこれは後ほど詳しく説明しますが、ViewControllerに多様な処理が含まれるということは、ViewControllerの実装は対応する各画面の仕様によって大きく異なるということです。  
そのためViewControllerの概念的な理解は開発する上で基本的なガイドラインを示してくれはするものの、実際にコードを読む・書くという作業においては各ViewController毎に考えなくてはいけない部分が大きいです。

### 2.命令的プログラミングと宣言的プログラミングが混在する
まず始めに命令的プログラミングと宣言的プログラミングの違いを簡単に説明します。  
恐らくこの[記事](https://qiita.com/Hiroyuki_OSAKI/items/f3f88ae535550e95389d)の説明が分かりやすいと思います。  
要は
命令的プログラミング･･･目的を達成するために、細かい指示を記述するプログラミングスタイル(HOW you want)
宣言的プログラミング･･･目的のみを記述して、細かい指示はしないプログラミングスタイル(WHAT you want)
という違いであり、宣言的なプログラミングの方は機械に対して自然言語(基本英語)で指示を出しているようなコードになり読みやすいです。  
ただ宣言的なプログラミングは内部的には命令的プログラミングによって実装されているため、こうした対称性は形式的な次元においてのみ意味があることは注意が必要でしょう。(C++とSwiftはプログラミングスタイル(形式的な次元)において比較することには意味がありますが、Swift自体が内部的にはC++で実装されている(基本的には)ためある観点から言えばこれらの比較が意味がないことと同じです。)  

#### 命令的プログラミング=ViewController(UIKit),宣言的プログラミング=SwiftUI
ViewController(UIKit)には宣言的な記述に思える箇所もありますが、全体的にはViewController(UIKit)は命令的プログラミング、SwiftUIは宣言的プログラミングと捉えて問題ないでしょう。  
両者のプログラムがどのように違うのかに関しては「[こちらの記事](https://engineering.mercari.com/blog/entry/20201216-4043b38af1/)」の冒頭に書かれているコードが分かりやすいと思います。(両者の比較について分かりやすい例を作るとなると状況設定やコードの説明等それなりの文量になってしまうので本記事では割愛します。)  
参照記事内のViewControllerとSwiftUIのコードを見てみるとSwiftUIのコードは自然言語(英語)的で直感的に何をやっているのか理解できるのに対して、ViewControllerのコードは機械的で細々としているのが分かります。  
ViewControllerに慣れている開発者にとっては、ある程度のコード量であろうとその触りさえ知ればその挙動の概要を理解することは可能でしょう。  
しかしそのように慣れている開発者であっても、例えば不具合が生じて正確にコードを調査する必要が出てきた際などは細々した命令的コードを一つ一つ確認する必要があるため集中力が必要なってきます。  
#### 多様な処理を含むViewControllerにおいて、命令的なプログラミングはさらに可読性を落としてしまう
先程の参照記事にも書いてある通り、宣言的プログラミング(記事内ではDeclarative Style)は見通しが良いと書かれていますが、それは逆にいうと命令的に書かれたViewcControllerは見通しが良くないということです。(これは相対的に見通しがよくないという話ではなく、人間の一般的な認識能力を考えると見通しが悪いということです。)  
最初に挙げたようにViewControllerには実に多様な処理が含まれていますが、そのように多くの処理を行っていることに加えそれらが見通しの良くない命令的なプログラミングによって実装されていることはViewControllerの可読性を大きく落としていると思います。  

### 3.ViewControllerは対応する画面の仕様に大きく影響を受ける
ViewControllerは対応する画面の仕様に大きく影響を受けます。    
既に述べた**多様なプログラムが書かれる**、**命令的プログラミングが含まれる**等の要素に加えてこのように各画面によってその実装が大きく異なっていることは、ViewControllerにおいて一般的な性質を理解しているだけでは実際の開発を行うに当たり不十分であるということを意味していると思います。    
色々誤解が生まれそうな表現なので詳細に説明すると、まず私は一般的なViewControllerの性質の理解が実際の開発で役に立たないと言っているわけではありません。  
本記事の冒頭で説明したViewControllerの基本的な責務の理解はViewControllerに「書くべきこと」と「書くべきではないこと」の区別を可能にさせますし、またライフサイクル等の仕組みを理解することでViewController内のコードをどのように読んで(書いて)いけば良いか知る助けとなります。  
しかしViewControllerの開発においては、このような基本的な理解は大まかなところでは助けになりますがあまり具体的ではなく、結局のところの個々のViewControllerを実際にみてみないとどうなっているかはわからない面が大きいというのがここでの私の主張です。(例え正しく実装されていたとしても)  

例えば、UseCaseは「単一の目的を持った処理をカプセル化したもの」であり、このUseCaseの一般的な理解を得ることによってUseCase内の主目的となる処理は一つであることがわかります。  
そのためUseCase毎に細かな違いはもちろんありますが、その一般的な性質を理解することでプログラム的な構造もかなり具体的に把握することができます。  
他にもRepositoryも「ドメインオブジェクトの操作をカプセル化したもの」であるという性質の理解によって、Repositoryの単位は操作対称であるドメインオブジェクトであり、そこに定義されているメソッドはそのオブジェクトに対するCRUD処理であることまでわかります。  

このように通常のコンポーネントではその基本的な性質を理解することでその実装における型も具体的に見えてくるのに対して、ViewControllerは基本的な性質を理解できてもその実装は見えてきません。  
このようなVewControllerの特徴はスケールしやすさを考えると大きな欠点だと思います。  
アーキテクチャやデザインパターンなど開発において特定のパターン(型)が決まっている場合はその型に基づいて作業をすれば良いため円滑に物事を進めることができるのですが、ViewControllerのように型が無い場合だと実際にそれぞれのViewControllerを見て個々に対応していく必要があるのです。  

<a name="footnote1">*1</a>: 複数点あり原文(英語)も載せると見づらくなってしまうため、意訳のみ載せています。  

