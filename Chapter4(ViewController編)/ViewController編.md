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
そのような状況を踏まえても長期的にプログラミング開発を学ぼうと考えているのならば、設計を学ぶ際に直感的でシンプルに記述できるSwiftUIではなく原始的で複雑なUIKit/ViewControllerを素材にすることによって、そこで得た知見を後々の他のプラットフォームで開発する際に活かせる場面が出てくると思います。   


## ViewControllerの基本
それでは内容に入っていきますが、まず最初にViewControllerの基本的な情報について整理します。  
参考にしたのはAppleが開発者向けに公開している[UIViewController](https://developer.apple.com/documentation/uikit/uiviewcontroller)と[View Controller Programming Guide for iOS](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/index.html#//apple_ref/doc/uid/TP40007457-CH2-SW1)という2つのドキュメントです。  

### ViewControllerの責務
[UIViewControllerのドキュメント](https://developer.apple.com/documentation/uikit/uiviewcontroller)を見ると、ViewControllerの主な責務以下4点だと書かれています。<sup>[*1](#footnote1)</sup>  
1. Viewに紐づいているデータの変化に応じたViewの更新
2. Viewに関するユーザーインタラクションへの対応
3. Viewの大きさの変更、また画面全体のレイアウトの管理
4. (他のViewControllerも含めた)他のオブジェクトとの連携

このようにViewControllerの責務を総体的に見ると、ViewControllerの責務は文字通り「Viewの管理(Control)」であり、Viewとは直接的に関係のないデータの操作がその主な責務には含まれないことがわかると思います。  
**4.(他のViewControllerも含めた)他のオブジェクトとの連携**はいささか示している責務範囲が広いようには感じますが、これは恐らくアプリ内におけるViewControllerの重要度の高さから通知機能(NotificationCenter)等、Viewとは直接関係ないオブジェクトとの連携も行う場合もあるため具体的に責務を定義できず「他のオブジェクトとの連携」となってしまったのだと推測しています。  
ただ上記で挙げたNotificationCenter等一部例外はあるものの、やはりViewController全体を考えるとその主要な責務は**Viewに関連した処理**であると捉えて問題なく、その方がViewControllerを理解するのにわかりやすいと思います。    

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
そのため各Child ViewController(Content ViewController)さえしっかりとできていれば、Container ViewControllerがやるべきことはChild ViewController間の連携を管理するくらいであり、その具体的な方法に関して特に今回の設計論の観点から説明することはないと考えています。(Conatainer ViewControllerで操作するChild ViewControllerの親Viewの管理も通常のViewの管理と同じです。)  
なのでこの記事ではContent ViewControllerの設計に限定して話を進めます。  

## 現実のViewControllerの開発で起こる問題
ドキュメントを参考にしながらViewControllerの基本について触れましたが、ここでは現実にViewControllerの開発(コードを読む、書く)で起こる問題点について説明します。  
私はViewControllerで普及している一般的な手法で開発すると以下の問題が起こると考えています。  

1. 「Viewの管理(Control)」と一言でいえど、実際には多種多様なプログラムが書かれる
2.  命令的プログラミングと宣言的プログラミングが混在する
3.  対応する画面の仕様に大きく影響を受ける

### 1.「Viewの管理(Control)」と一言でいえど、実際には多様なプログラムが書かれる
#### コードレベルからViewControllerの責務が理解しにくい
記事冒頭の「[ViewControllerの基本](#ViewControllerの基本)」では、[Appleのドキュメント](https://developer.apple.com/documentation/uikit/uiviewcontroller)を引用してViewControllerの主要な責務を4点紹介し、それらはさらに「Viewの管理」へと要約できると説明しました。  
しかしそのように責務を簡潔に表現できるのはあくまで概念上の話です。  
実際のコードレベルにおいてViewControllerにはViewの操作、アニメーション、遷移処理、アラート操作、CollectionViewのデリゲート・データソース、その他オブジェクトの連携等、実にさまざまな処理が書かれており、時には簡単な状態変数の管理も行っているケースも見かけます。  
MVCやレイヤードアーキテクチャといった設計理論の基礎を理解できている開発者であればこのように雑多に思えるコードにおいてもメタ認識によってその責務を理解することができます。    
ただそうでない開発者にとってはこれらのコードからViewControllerの責務を正しく理解するのは難しいと思います。  
恐らく設計理論の理解よりも先にコード(実務)を通してプログラミングを学んでいる多くの開発者からすると上記のように様々な処理が含まれているViewControllerは「Viewを管理する存在」ではなく、「画面開発における便利屋的な存在」に見えてしまうのではないでしょうか。  
私はレイヤードアーキテクチャの記事でFatViewControllerの問題の原因としてViewController-Model間の関係の論理的な理解不足を指摘しましたが、このようにコードレベルにおいてViewControllerが雑多に思える処理を抱えている状況はそうしたViewController-Model間の関係(ViewControllerに何を書いてはいけないか)を理解することをより難しくさせていると思います。  
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/コードレベルにおけるViewController.png" alt="コードレベルにおけるViewController" width=60% > 


### 2.命令的プログラミングと宣言的プログラミングが混在する
#### 命令的プログラミングと宣言的プログラミング
まず始めに命令的プログラミングと宣言的プログラミングの違いを簡単に説明します。  
恐らくこの「[宣言的？ Declarative?どういうこと？](https://qiita.com/Hiroyuki_OSAKI/items/f3f88ae535550e95389d)」という記事が説明が分かりやすいと思います。   
要は  
命令的プログラミング･･･目的を達成するために、細かい指示を記述するプログラミングスタイル(How you want)  
宣言的プログラミング･･･目的のみを記述して、細かい指示はしないプログラミングスタイル(What you want)  
という違いですが、宣言的プログラミングはその性質からハードウェアに対して自然言語(基本英語)で指示を出しているようなコードになり直感的に読みやすいため昨今主流になりつつあります。  
 

#### 命令的プログラミング=ViewController(UIKit),宣言的プログラミング=SwiftUI
ViewController(UIKit)には宣言的な記述に思える箇所もあると思いますが、基本的にはViewController(UIKit)は命令的プログラミング、SwiftUIは宣言的プログラミングと認識して問題ありません。  
ここでは両者のプログミングがどのように異なるのか知るために簡単な例を見てみます。  
下記の画像のように写真とそのタイトルを表示したリストとさらにそれぞれのアイテムを選択した時に詳細画面に遷移できる機能をUIKitとSwiftUIそれぞれで実装してそのコードを比較してみました。(画像はSwiftUIで実装したものです)<sup>[*3](#footnote3)</sup>  
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/命令的プログラミングと宣言的プログラミングの比較例.png" alt="命令的プログラミングと宣言的プログラミングの比較例" width=30% > 

命令的プログラミング・ViewController(UIKit)で実装したコード  
```
class PhotoCollectionViewController: UIViewController {

    let photos: [Photo]
    
    private lazy var tableView: UITableView = {
        let tableView = UITableView(frame: .zero, style: .plain)
        tableView.translatesAutoresizingMaskIntoConstraints = false
        tableView.register(UITableViewCell.self,
                           forCellReuseIdentifier: "PhotoCell")
        tableView.delegate = self
        tableView.dataSource = self
        self.view.addSubview(tableView)
        NSLayoutConstraint.activate([
            tableView.topAnchor.constraint(equalTo: self.view.topAnchor),
            tableView.leadingAnchor.constraint(equalTo: self.view.leadingAnchor),
            tableView.trailingAnchor.constraint(equalTo: self.view.trailingAnchor),
            tableView.bottomAnchor.constraint(equalTo: self.view.bottomAnchor),
        ])
        return tableView
    }()
    
    init(photos: [Photo]) {
        self.photos = photos
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        navigationItem.title = "Photos"
        _ = tableView
    }
}

extension PhotoCollectionViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView,
                   didSelectRowAt indexPath: IndexPath) {
        self.navigationController?
            .pushViewController(PhotoDetailViewController(photo: self.photos[indexPath.row]),
                                animated: true)

    }
}

extension PhotoCollectionViewController: UITableViewDataSource {
    
    func numberOfSections(in tableView: UITableView) -> Int {
        1
    }
    
    func tableView(_ tableView: UITableView,
                   numberOfRowsInSection section: Int) -> Int {
        self.photos.count
    }
    func tableView(_ tableView: UITableView,
                   cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell: UITableViewCell
        if let dequeuedCell = tableView.dequeueReusableCell(withIdentifier: "PhotoCell") {
            cell = dequeuedCell
        } else {
            cell = UITableViewCell(style: .default, reuseIdentifier: "PhotoCell")
        }
        let photo: Photo = photos[indexPath.row]
        
        //print(cell.imageView)
        cell.imageView?.image = UIImage(named: photo.imageName)
        cell.imageView?.contentMode = .scaleToFill
        cell.textLabel?.text = String(photo.title)
        
        return cell
    }
    
    
}

```

宣言的プログラミング・SwiftUIで実装したコード  
```
struct PhotoCollectionView: View {
    @State var photos: [Photo]
    var body: some View {
        
        NavigationView {
            List(photos) { photo in
                NavigationLink(destination: PhotoDetailView(photo: photo)) {
                    PhotoRow(photo: photo)
                }
            }
            .navigationTitle("Photos")
        }
    }
}

struct PhotoRow: View {
    var photo: Photo

    var body: some View {
        HStack {
            Image(photo.imageName)
                .resizable()
                .frame(width: 50, height: 50)
            Text(photo.title)
            
            Spacer()
            
        }
    }
}
```


上記のコードを見て見ると宣言的プログラミングであるSwiftUIのコードの方が自然言語(英語)的で直感的に何をやっているのか理解しやすいのがわかると思います。<sup>[*4](#footnote4)</sup>   
それに比べると命令的プログラミングであるViewController(UIKit)のコードは機械的で細々としておりわかりづらく感じます。  
ViewControllerに慣れている開発者であればそのように細々としたコードでも触りをみるだけでそれがどのような責務を担っているのか大体理解できると思いますが、しかしそのよう例えば不具合が生じて正確にコードを調査する必要が出てきた際などには細々した命令的プログラムを１行１行確認する必要があるため集中力が必要なります。    
#### 多様な処理を含むViewControllerにおいて、命令的なプログラミングはさらに可読性を落としてしまう
先程の[参照記事](https://engineering.mercari.com/blog/entry/20201216-4043b38af1/)にも書かれている通り、宣言的プログラミング(記事内ではDeclarative Style)は見通しが良いと書かれていますが、それは逆にいうと命令的に書かれたViewControllerは見通しが良くないということです。<sup>[*5](#footnote5)</sup>   
最初に挙げたようにViewControllerには実に多様な処理が含まれていますが、そのように多くの処理を行っていることに加えてそれらが見通しの悪くなる命令的なプログラミングによって実装されていることはViewControllerの可読性を大きく落としていると思います。  

### 3.ViewControllerは対応する画面の仕様に大きく影響を受ける
#### ViewControllerのプログラムの構造を理解するためには、実際にコードを読まなくてはいけない
ViewControllerは対応する画面の仕様に大きく影響を受けます。    
既に述べた**多様なプログラムが書かれる**、**命令的プログラミングである**ことに加えてこのように各画面によってその実装が大きく異なるという特徴は、ViewControllerにおいて一般的な性質を理解しているだけでは実際の開発を行うに当たり不十分であることを意味していると思います。    
誤解が生まれそうな表現なので詳細に説明すると、まず私は一般的なViewControllerの性質の理解が実際の開発で役に立たないとは思っていません。    
本記事の冒頭で説明したViewControllerの基本的な責無を理解すればViewControllerに「書くべきこと」と「書くべきではないこと」の区別ができるようになりますし、ライフサイクルの仕組みを理解することでViewController内のコードをどのように読んで(書いて)いけば良いか知る助けとなります。  
しかしViewControllerの開発において上記ような事柄への理解は抽象的な面では役に立ちますが、あまり具体的なガイドラインを示せていないように思います。    
そのため結局のところViewControllerの開発の際には個々のプログラムを実際にみてみないとどうなっているかはわからない側面が大きいというのが私の主張です。

#### UseCaseはその概念的に理解によって、プログラムの構造も具体的にわかる
例えば、UseCaseは「単一の目的を持った処理をカプセル化したもの」であり、ここからUseCase内の主目的となる処理は一つだけであることがわかります。  
そのため各UseCaseに細かな違いはあるものの、この一般的な性質を理解することでプログラム的な構造もかなり具体的に把握することができます。  
詳しく説明するとUseCaseが概念に沿って正しく設計されていたならば、一番上部に主目的となるメソッドが定義されていて、その他全ての変数、内部処理はその目的を達成するための手段であるというプログラム構造となっているはずです。    
他にもRepositoryも「ドメインオブジェクトの操作をカプセル化したもの」であるという性質の理解によって、Repositoryの単位は操作対称であるドメインオブジェクトであり、そこに定義されているメソッドはそのオブジェクトに対するCRUD処理であるという構造が見えてきます。  
例えばARepositoryであれば「A」というオブジェクトを操作対象としており、そこに定義されているメソッドは「A」に対するCRUD処理で、そこで利用しているインスタンス変数はそのCRUD処理のために利用されるという構造です。  
#### ViewControllerには具体的なパターン(型)がない
このように通常のコンポーネントではその基本的な性質によってその具体的なプログラム構造まで決定されますが<sup>[*6](#footnote6)</sup>、ViewControllerの場合その基本的な性質によって具体的なプログラム構造が決定されません。<sup>[*7](#footnote7)</sup>   
このようなVewControllerの特徴はスケールの観点からは大きな欠点だと思います。  
アーキテクチャやデザインパターン等を代表するように具体的な型が決まっていることは開発の効率を大きく向上させます。  
具体的な型があることで開発の際「どのようにコードを読めば(書けば)良いか」悩む機会は大きく減りますし、また一般的なアプリケーション開発では「AViewController」「BViewController」のように同様のコンポーネントを複数定義・実装しますがその際にも型を使い回すことできます。  

それに対してViewControllerのように型が決まっていない場合は個々に対応していく必要があるためコストがかかりますし、またこうした状況で開発を続けているとプロジェクト内の統一性が失われて混乱が起きてしまう恐れがあります。  

## スケールしやすいViewControllerの設計を考える
ViewControllerの基本的な性質と開発時に起こる問題点を見てきました。  
ここではそれらの事柄を踏まえて本記事の本題と言えるスケールしやすいViewControllerの設計を考えていきます。  
### ViewControllerを再定義する
ViewController開発の問題点としてUseCaseやRepository等のコンポーネントではその基本的な性質によってプログラム構造の型が決定され、そのおかげで開発時に悩む機会が少なくなるが、ViewControllerにはそのような型が存在しないことを指摘しました。  
ここではViewControllerの開発にもそのような雛形を作るべくViewControllerを再定義します。  
しかし再定義するといっても、厳密には既存のViewController定義を上書きするようなことはせずそのコアとなる定義を行うことでViewControllerの責務をより明確にします。  
具体的にはViewControllerのコアを以下のように定義します。  
  
>**ViewControllerはUI/システムから(への)イベントを処理する機構である。**
  
今までと何が違うのかよくわからない人もいるかもしれませんが、既存のViewControllerの定義は「Viewの管理」と要約と書いたようにそこでは「View」を主眼としていました。  
しかし今回の定義では「View」ではなく「イベント」が主眼となっています。  
私はこれによって以下の2点の変化が起こると思います。  
- 開発においてイベントの入力、出力処理が一次責務として強調されるようになる
- View、Alert、遷移等の具体的な操作が二次責務と捉えられるようになる


そしてプログラムも以下のような構造を持つことになるでしょう。  

<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/ViewControllerの構造.png" alt="ViewControllerの構造" width=55% > 

プログラム構造として大分わかりやすくなったのではないでしょうか。  
最初に紹介した[4点のViewControllerの責務](#ViewControllerの責務)は概念としてはわかりやすかったのですが、実際にそれを基に実装するとなるとそれぞれの責務の関係に規則性が見えないためそれらの責務がViewController内にどのように書かれ<sup>[*8](#footnote8)</sup>、また全体の構造がどうなっているのか非常に予想しづらかったです。  
しかしViewControllerのコア責務を「イベント処理」と定義することで「入力イベントの処理」と「出力イベントの処理」という対等な関係にある2つの責務がプログラムの骨格を担うようになりその構造が明確になりました。  
これによってViewControllerのプログラムは以下のような構造で統一化することが可能になります。(これはあくまで一例であり他の形式もありえます。しかし大体に似たような形式になると思います。)  


```
// 「入出力イベントの処理」を責務としたViewControllerのプログラム構造(一例)
// ---------------------------------------------------------- 
// クラス本体 -> インスタンス変数と初期化処理
// extension(1) -> 入力イベントの処理
// extension(2) -> 出力イベントの処理
// ---------------------------------------------------------- 

class ViewController: UIViewController {
    // MARK: Viewのインスタンス変数
    ...
    ...
    
    // MARK: 入力に関するインスタンス変数
    ...
    ...
    
    // MARK: 出力に関するインスタンス変数
    ...
    ...
    
    init() {
        // 初期化処理
        ...
        ...
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    deinit {
       // 開放処理
    }
    
    override func loadView() {
        // 初期化処理
        ...
        ...
    }
}

// MARK: 入力処理
extension ViewController {
  // 入力処理の定義・実装
  ...
  ...
  
}

// MARK: 出力処理
extension ViewController {
  // 出力処理の定義・実装
  ...
  ...
  
}
```
<sup>[*9](#footnote9)</sup>

### View、Alert、遷移等の具体的な操作を他コンポーネントに委譲する
ViewControllerのコアを「UI/システムから(への)イベントを処理する機構」と定義することで「View、Alert、遷移等の具体的な操作」が二次責務となりましたが、このアーキテクチャではこれらの二次責務をViewControllerから他のコンポーネントに委譲します。  
しかし留意したいのはこれらの責務の委譲が必須というわけではありません。  
今回の設計では先ほど述べたViewControllerのコアの定義により「View、Alert、遷移等の具体的な操作」を脱着可能な二次責務として認識しています。  
そのため長期的に運用しても規模が大きくならないと決まっているアプリや短期間のみ公開するアプリではViewControllerにこれらの責務を直接実装しても問題ありません。  
ただ後ほど詳しく理由を説明しますが、本記事が目的としているスケールしやすいアーキテクチャは不特定多数の長期的なアプリの開発を前提としており、その場合は「View、Alert、遷移等の具体的な操作」はViewControllerの外部に委譲する方が良いと考えています。  

その場合ViewControllerとそのの構成は以下のようになります。  
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/具体的な操作処理を外部に委譲したViewController.png" alt="具体的な操作処理を外部に委譲したViewController" width=60% > 

そしてこのように具体的な操作を外部に委譲することで[記事内で挙げたViewControllerの問題](#現実のViewControllerの開発で起こる問題)が解決されます。  

#### 『「Viewの管理(Control)」と一言でいえど、実際には多様なプログラムが書かれる』問題の解決

#### 『命令的プログラミングと宣言的プログラミングが混在する』問題の解決

#### 『ViewControllerは対応する画面の仕様に大きく影響を受ける』問題の解決

## 実装方法

### ViewControllerの実装

### 実装例



## 脚注
<a name="footnote1">*1</a>: 複数点あり原文(英語)も載せると見づらくなってしまうため、意訳のみ載せています。  
<a name="footnote2">*2</a>: アプリの仕様としてContainer ViewControllerを積極的に利用する方針にしているケースもなくはないと思いますが、全体から見ればごく限られたケースだと思います。  
<a name="footnote3">*3</a>: あくまで目的はViewController(命令的プログラミング)とSwiftUI(宣言的プログラミング)の違いを把握することなので、ViewController側ではCellのレイアウト等何点か省略している実装があります。またSwiftUI側も@Stateの利用方法などベストプラクティスとは言えない実装があります。   
<a name="footnote4">*4</a>: ViewController(UIKit)側ではAuto Layoutをコードで実装しているためInterface Builderより煩雑になっていると言えると思います。しかし実際開発ではViewController(UIKit)側にはさらにプリフェッチ処理、差分検知、その他省略した何点かの実装が加わるためAuto Layoutをコード実装していなくてもプログラムは例よりさらに煩雑になっているはずです。  
<a name="footnote5">*5</a>: ここで述べている命令的プログラミングの見通しが良くないという評価は相対的なものではなくある程度客観性を持っていると思います。機械的で細々した命令的プログラミングは人間の一般的な認知能力からして見やすいモノではないはずです。  
<a name="footnote6">*6</a>: PresenterやViewModelといったコンポーネントもUseCaseやRepositoryと比べると責務が広範囲に及んで漠然としている印象を受けますが、それでもその
性質を突き詰めると「View-BusinssLogic間のデータ変換を行うコンポーネント」であると定義することができます。そしてここから「View->Business Logicのデータ変換とBusiness Logic->Viewのデータ変換」とい構造を持ったプログラムであるべきことが見えてきます。  
<a name="footnote7">*7</a>: 繰り返しのようになりますがライフサイクルの仕組み等を理解することでViewControllerのプログラム構造が見えてくる面もありますし、また[脚注5](#footnote5)で挙げたPresenter(ViewModel)の定義のようにViewControllerの定義を「View-Presenter(ViewModel)間の処理を行うコンポーネント」というように定義することも可能だと思います。しかし少なくともViewControllerのドキュメントに沿った定義ではViewControllerがCollectionViewのデリゲート、データソースとして振る舞うことやそれらに関連する状態管理も許容しており、私はこれらの処理がViewControllerのプログラム構造を捉える上で切り捨てて良い瑣末なモノであるとは思えません。そのためViewControllerのプログラム構造は一般的な性質のみでは理解できず、実際にコードを見て見ないとわからないと主張しています。  
<a name="footnote8">*8</a>: ドキュメントで紹介した4点の責務を順番に記述しているかもしれませんし、プロダクト機能(モジュール)を構成単位としているかもしれません。恐らく一般的にはプロダクト機能を構成単位としている場合が多いと思いますが、その場合にも各画面によってプロダクト機能は異なるためそれぞれのViewControllerを実際に確認しなければその構造を把握できません。  
<a name="footnote9">*9</a>: ViewControllerのライフサイクルメソッドのうちloadView()はクラス本体(初期設定)に含まれていますが、ViewDidload等その他のライフサイクルメソッドに関しては入力処理に含まれるべきです。

