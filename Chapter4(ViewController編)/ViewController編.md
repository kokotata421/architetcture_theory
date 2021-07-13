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
- SwiftUIは扱わず、UIKitを利用して開発しています。  
- UI開発はInterface Builderを利用せずコードのみで行っています。    
- 作成したサンプルプロジェクトはMVVMをベースに考えていますが、記事内容はどんなアーキテクチャでも共通する考えとなっているはずです。  
- FluxやReduxのアーキテクチャは概念としては触れる予定ですが、サンプルプロジェクトでは採用されていません。  

## 前回までの内容と本記事の内容
前回までの記事で設計の概論、そしてそれを踏まえた本記事で提案しているアーキテクチャの概要を説明しました。  
この記事ではViewControllerの設計について説明します。  

## Swift UI時代にViewControllerを学ぶ意義
この記事を読んでくれている人の中には「設計は興味あるけど、今更ViewControllerかあ...」と思っている方もいると思います。  
私もその意見に同意します。  
ただそれでもモチベーションを持ってこの記事を読んで欲しいため最初に私なりのViewControllerの設計を学ぶ意義を述べます。    
### ViewControllerを採用したのは、ただの偶然
そもそもこの記事の中でViewControllerを採用している理由はただ「このスケールしやすいアーキテクチャの探究を始めた時、SwiftUIがまだ登場して間もなかった」からというだけであって、私も意図的にViewControllerを採用したわけではありません。  
今から始めるならば間違いなくSwiftUIでやっていました。  

### それでも今からViewControllerで設計を学ぶ意義
しかし今から振り返ると今回の試みの中ViewControllerを採用したのは決して悪いことではなかったと考えています。  
その理由は、設計をその原理まで理解しようとするならばViewControllerかSwift UIか、またiOSかAndroidかといった開発におけるプラットフォームはあまり関係ないからです。  
もちろんViewControllerとSwift UIの記述の仕方で異なったり、AndroidではViewの実装でxmlを直接操作したりと細かいところで独自に学ばなければいけないことはあります。  
しかし設計は本質的にプラットフォームには依存しておらず、その素材にViewController、SwiftUIのどちらを利用しようが、そこで得られたノウハウは他のプラットフォームでも間違いなく活きてきます。      
またSwiftUIの現在(2021年5月時点)の仕様を考えると、たとえSwiftUIが主流になろうとも当分の間はUIKitがiOS開発にとって完全に不必要になるわけではなさそうです。  
そのような状況を考えても今このタイミングでSwiftUIではなくUIKit/ViewControllerを素材に設計を学んだとしてもそれは必ず何かしらの形でiOS開発の役に立ちますし、そしてSwiftUIよりもよりローレベル(原始的)な技術であるUIKitを通して設計を学ぶことで他の開発環境でも柔軟に対応できる応用力が身につくと思います。  

## ViewControllerの基本
それでは内容に入っていきますが、まず最初にViewControllerの基本的な情報について整理します。  
参考にしたのはAppleが開発者向けに公開している[UIViewController](https://developer.apple.com/documentation/uikit/uiviewcontroller)と[View Controller Programming Guide for iOS](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/index.html#//apple_ref/doc/uid/TP40007457-CH2-SW1)という2つのドキュメントです。  

### ViewControllerの責務
[UIViewControllerのドキュメント](https://developer.apple.com/documentation/uikit/uiviewcontroller)を見ると、ViewControllerの主な責務以下4点だと書かれています。<sup>[*1](#footnote1)</sup>  
1. Viewに紐づいているデータの変化に応じたViewの更新
2. Viewに関するユーザーインタラクションへの対応
3. Viewの大きさの変更、また画面全体のレイアウトの管理
4. (他のViewControllerも含めた)他のオブジェクトとの連携

このようにViewControllerの責務を総体的に見るとその責務はViewと関係しており、Viewと関係のないデータの操作はその主な責務には含まれないことがわかると思います。  
**4.(他のViewControllerも含めた)他のオブジェクトとの連携**はいささか示している範囲が広いようには感じますが、この場合にもやはりその責務の中心には「View」があることは間違いありません。  
そのためViewControllerのその名前の通り「**Viewの管理&#40;Control&#41;するコンポーネント**」と捉えるのは、その役割をシンプルに理解する上で大事な考えだと思います。    
### 2種類のViewController
ViewControllerの責務は先程の4点であることは変わらないのですが、その種類は利用ケースによって大きく2つに分かれます。  
1. Content ViewController: 自身に紐づいた**Viewを管理**することを責務としたViewController
2. Container ViewController: 自身の画面内で表示する**ViewController&#40;Container ViewControllerの文脈ではChild ViewControllerと言う&#41;の管理**を責務としたViewController

#### Content ViewControllerとContainer ViewControllerの違い
大きな違いはViewController内で具体的なViewコンポーネントを操作するかどうかだと思います。    
Content ViewControllerでは自身に表示されたUIButtonやUILabelといったViewを直接操作するのに対して、Container ViewControllerでは自身が管理するViewControllerとその親View(Root View)のみを操作するためUIButtonやUILabel等を直接操作することはありません。  
一般的にアプリ内で独自に定義するViewControllerのほとんどはContent ViewControllerだと思います。<sup>[*2](#footnote2)</sup>  
Container ViewControllerはあまり独自で定義することはないと思いますが、ただ私たちがアプリ内でよく利用するNavigation ControllerやTab Bar ControllerはContainer ViewControllerに該当します。  
#### 記事で扱うのはContent ViewControllerのみ
この記事で扱うのはContent ViewControllerに限定されます。  
[ドキュメント](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/ImplementingaContainerViewController.html#//apple_ref/doc/uid/TP40007457-CH11-SW1)に書かれている通りContainer ViewControllerでは**自身のChild ViewControllerへの干渉を最低限とするべき**であり、それはつまるところContainer ViewController内にある各Content ViewControllerの設計の重要性を意味しています。  
各Content ViewControllerが独立した形でしっかりと設計がなされていれば、Container ViewControllerからChild ViewControllerへの干渉はその生成・破棄の管理、Root Viewの操作、Child ViewController間の連携、あたりに限定されるからです。<sup>[*3](#footnote3)</sup>  
そのため設計論においてContainer ViewControllerに関して固有に考えなくてはいけないことは特になく、本記事ではContent ViewControllerの設計に限定して話を進めることとします。<sup>[*4](#footnote4)</sup>   

## 現実のViewControllerの開発で起こる問題
ドキュメントを参考にしながらViewControllerの基本について触れましたが、ここでは現実にViewControllerの開発で起こる問題点について説明します。  
私はViewControllerをその一般的な手法で開発すると以下の問題が起こると考えています。  

1. 「Viewの管理(Control)」と一言でいえど、実際には多種多様なプログラムが書かれる
2.  命令的プログラミングと宣言的プログラミングが混在する
3.  対応する画面の仕様に大きく影響を受ける

### 1&#046;&#x300C;Viewの管理&#40;Control&#41;&#x300D;と一言でいえど&#x3001;実際には多様なプログラムが書かれる
#### コードレベルからViewControllerの責務が理解しにくい
記事冒頭の「[ViewControllerの基本](#ViewControllerの基本)」では、[Appleのドキュメント](https://developer.apple.com/documentation/uikit/uiviewcontroller)を引用してViewControllerの主要な責務を4点紹介し、それらはさらに「Viewの管理」へと要約できると説明しました。  
しかしそのように責務を簡潔に表現できるのはあくまで概念上の話です。  
実際のコードレベルにおいてViewControllerにはViewの操作、アニメーション、遷移処理、アラート操作、CollectionViewのデリゲート・データソース、その他オブジェクトの連携等、実にさまざまな処理が書かれており、中には簡単な状態変数の管理も行っているケースもあります。    
恐らく設計理論の基礎を押さえている開発者であれば、こうした雑多に思えるコードにおいてもメタ認識によってその責務をある程度理解できると思います。            
しかしViewControllerの責務はMVCやMVPなどのアーキテクチャ、またアプリの仕様といった外部環境によって変化します。    
そのため一般的に様々な処理が実装されているViewControllerでそれぞれがどのような責務を担っているかコードレベルから判断するのはそれなりに大変な作業になります。  

<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/コードレベルにおけるViewController.png" alt="コードレベルにおけるViewController" width=60% > 


### 2&#046;命令的プログラミングと宣言的プログラミングが混在する
#### 命令的プログラミングと宣言的プログラミング
まず始めに命令的プログラミングと宣言的プログラミングの違いを簡単に説明します。  
恐らくこの「[宣言的？ Declarative?どういうこと？](https://qiita.com/Hiroyuki_OSAKI/items/f3f88ae535550e95389d)」という記事が説明が分かりやすいと思います。   
要は  
命令的プログラミング･･･目的を達成するために、細かい指示を記述するプログラミングスタイル(How you want)  
宣言的プログラミング･･･目的のみを記述して、細かい指示はしないプログラミングスタイル(What you want)  
という違いです。  
宣言的プログラミングはその性質からハードウェアに対して自然言語(基本英語)で指示を出しているようなコードになり直感的に読みやすいため昨今UI実装において主流になりつつあります。  
 

#### 命令的プログラミング=ViewController(UIKit),宣言的プログラミング=SwiftUI
ViewController(UIKit)には宣言的な記述に思える箇所もあると思いますが、基本的にはViewController(UIKit)は命令的プログラミング、SwiftUIは宣言的プログラミングと認識して問題ありません。  
ここでは両者のプログミングがどのように異なるのか知るために簡単な例を見てみます。  
下記の画像のように写真とそのタイトルのリストを表示して、それぞれのアイテムを選択した時に詳細画面に遷移できる機能をUIKitとSwiftUIで実装しそのコードを比較してみました。(画像はSwiftUIで実装したものです)<sup>[*3](#footnote3)</sup>  
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
        
        cell.imageView?.image = UIImage(named: photo.imageName)
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


上記のコードを見てみると宣言的プログラミングであるSwiftUIのコードの方が自然言語(英語)的で何をやっているのか理解しやすいと思います。<sup>[*4](#footnote4)</sup>   
それに対して命令的プログラミングであるViewController(UIKit)のコードは機械的で細々としていてわかりづらく感じてしまいます。    
ViewControllerに慣れている開発者であればそのように細々としたコードでも経験によってその触りをみるだけで大体理解できるので問題ないのかもしれません。   
しかしその場合でも例えば不具合が生じる等して正確にコードを調査する際にはコードを１行１行確認していくため集中力が必要なります。    
#### 多様な処理を含むViewControllerにおいて、命令的なプログラミングはさらに可読性を落としてしまう
先程のコード例を見ても明らかなようにViewController(UIKit)のコードは見通しがよくありません。<sup>[*5](#footnote5)</sup>  
そしてそうした命令的プログラミングの見通しの悪さに加えて、最初に挙げた[多様な処理を含んでいる特徴](#1Viewの管理Controlと一言でいえど実際には多様なプログラムが書かれる)はViewControllerプログラムの可読性を大きく下げてしまっています。  

### 3.ViewControllerは対応する画面の仕様に大きく影響を受ける
#### ViewControllerのプログラムの構造を理解するためには、実際にコードを読まなくてはいけない
ViewControllerは対応する画面の仕様に大きく影響を受けます。    
この特徴は既に述べた[多様なプログラムが書かれている](#1Viewの管理Controlと一言でいえど実際には多様なプログラムが書かれる)、[命令的プログラミングである](#2命令的プログラミングと宣言的プログラミングが混在する)こととも合わさって実際の開発においてViewControllerの一般的な性質を理解しているだけでは不十分であることを意味しています。  
誤解が生まれそうな表現なので詳細に説明します。  
もちろん一般的なViewControllerの性質理解によって開発で役に立つ面もあります。    
本記事の冒頭で説明したViewControllerの基本的な責務を理解すればViewControllerに「書くべきこと」と「書くべきではないこと」の区別がある程度できるようになりますし、ライフサイクルの仕組みを理解することでViewController内のコードをどのように読んで(書いて)いけば良いか知る助けとなります。  
しかしViewControllerの開発においてこのような理解は抽象的な面では役に立ちますが、あまり具体的なガイドラインを示してくれません。    
そのため結局のところ開発時には個々のViewControllerのプログラムを実際にみてみないとどうなっているかはわからない側面が大きいというのがここでの主張です。

#### UseCaseはその概念的な理解によって、プログラムの構造も具体的にわかる
例えば、UseCaseは「単一の目的を持った処理をカプセル化したもの」であり、ここからUseCase内の主目的となる処理は一つだけであることがわかります。  
そのため各UseCaseに細かな違いはあるものの、この一般的な性質を理解することでプログラム的な構造もかなり具体的に把握することができます。  
もしUseCaseがその概念に沿って正しく設計・実装されているならば、自然と上部に主目的となるメソッドが定義されていて、その他全ての変数、内部処理はその目的を達成するための手段であるというプログラム構造になっているはずです。  
他にもRepositoryも「ドメインオブジェクトの操作をカプセル化したもの」であるという性質を理解することで、Repositoryの単位は操作対象であるドメインオブジェクトであり、そこに定義されているメソッドはそのオブジェクトに対するCRUD処理であるという構造が見えてきます。  
ARepositoryであれば「A」というオブジェクトを操作対象としていて、そこに定義されているメソッドは「A」に対するCRUD処理であり、インスタンス変数もそれらCRUD処理のために利用されているという構造になっていると思います。    
#### ViewControllerには具体的なパターン(型)がない
このように通常のコンポーネントでは基本的な性質によって具体的なプログラム構造まで決定されますが<sup>[*6](#footnote6)</sup>、ViewControllerの場合その基本的な性質によってどのような責務が構成要素として含まれるかはわかっても具体的なプログラム構造までは決定されません。<sup>[*7](#footnote7)</sup>   
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/コンポーネントの性質と構造.png" alt="コンポーネントの性質と構造" width=75% >  

こうしたVewControllerの特徴はスケールの観点からは大きな欠点だと思います。  
アーキテクチャやデザインパターン等を代表するように具体的な型が決まっていることは開発の効率を大きく向上させます。  
具体的な型があることで開発の際「どのようにコードを読めば(書けば)良いか」悩む機会は減りますし、またアプリケーション開発では一般的に「AViewController」「BViewController」のように同様のコンポーネントを複数定義・実装しますがその際にも型を使い回すことで開発コストは大きく短縮されます。  

それに対してViewControllerのように型が決まっていない場合は個々に対応していく必要があるためコストがかかります。  
さらにそのような状況で開発を続けることでプロジェクト内の統一性が失われて混乱が起きてしまう恐れもあると思います。    

## スケールしやすいViewControllerの設計を考える
ViewControllerの基本的な性質と開発時に起こる問題点を見てきました。  
ここではそれらの事柄を踏まえて本記事の本題と言えるスケールしやすいViewControllerの設計を考えていきます。  
### ViewControllerを再定義する
ViewControllerの問題点として基本的なプログラム構造の型が存在しないため個々に対応していく箇所が多いことを挙げました。  
ここではViewControllerの開発にも雛形を作るべくViewControllerを再定義します。  
しかし再定義するといっても、厳密には既存のViewController定義を上書きするようなことはせずそのコアとなる定義を行うことでViewControllerの責務をより明確にするだけです。    
具体的にはViewControllerのコアを以下のように定義します。  

***
**ViewControllerはUI/システムから(への)イベントを処理する機構である。**
***
今までと何が違うのかよくわからない人もいるかもしれませんが、既存のViewControllerの定義は「Viewの管理」と要約と書いたようにそこでは「View」を主眼としていました。  
しかし今回の定義では「View」ではなく「イベント」が主眼となっています。  
これによって以下の2点の変化が起こります。    
- イベントの入力・出力処理が一次責務として強調されるようになる
- View、Alert、遷移等の具体的な操作が二次責務と捉えられるようになる

### &#12300;イベント&#12301;を中心に据えることでViewControllerのプログラム構造も決まってくる
そして「イベント」を中心に据えることでプログラム構造も以下の形に収まっていきます。     

<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/ViewControllerの構造.png" alt="ViewControllerの構造" width=55% > 

プログラム構造として大分わかりやすくなったのではないでしょうか。  
最初に紹介した[4点のViewControllerの責務](#ViewControllerの責務)は概念としてはわかりやすかったものの、実際にそれを基に実装するとなるとそれぞれの責務の関係に規則性が見えないためそれらがViewController内にどのように書かれ<sup>[*8](#footnote8)</sup>、また全体としてどのような構造になるのか客観的に決定できませんでした。        
しかしViewControllerのコア責務を「イベント処理」と定義することで「入力イベントの処理」と「出力イベントの処理」という対等な関係にある2つの責務がプログラムの骨格を担うようになった結果、その構造が明確になります。      
これによってViewControllerのプログラムは以下のような構造で統一化されると思います。(これはあくまで一例であり細かい箇所において異なる形式もありえますが、大枠などれも以下のようになっているはずです。)    


```
// 「入出力イベントの処理」を責務としたViewControllerのプログラム構造(一例)
// ---------------------------------------------------------- 
// ViewCotnrollerの構造 {
//    クラス本体 -> インスタンス変数と初期化処理
//    extension(1) -> 入力イベントの処理
//    extension(2) -> 出力イベントの処理
// }
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
ViewControllerのコアを「UI/システムから(への)イベントを処理する機構」と定義することで「View、Alert、遷移等の具体的な操作」が二次責務となりました。  
この記事で提案するアーキテクチャではViewControllerのプログラム構造をよりシンプルかつ統一的にするためにこれらの二次責務をViewControllerから他のコンポーネントに委譲します。  
  
その場合ViewControllerの構造とその周辺関係は以下のようになります。  
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/具体的な操作処理を外部に委譲したViewController.png" alt="具体的な操作処理を外部に委譲したViewController" width=60% > 

こうして具体的な操作を外部に委譲することで記事内で挙げた[ViewControllerの問題](#現実のViewControllerの開発で起こる問題)が解決されます。  


> 補足:  
> 「View、Alert、遷移等の具体的な操作」の責務の委譲は必須というわけではありません。  
> 今回の設計ではViewControllerのコアを「イベント処理」と定義して、「View、Alert、遷移等の具体的な操作」を脱着可能な二次責務として認識しています。  
> そのため個々のアプリの事情に合わせてViewControllerにそれらの責務を直接実装しても良いと思います。    
> しかし本記事ではスケールしやすいアプリを目指すためにはこれらを外部に委譲することをオススメします。    


#### 『「Viewの管理(Control)」と一言でいえど、実際には多様なプログラムが書かれる』問題の解決
ViewControllerの一部の責務を他コンポーネントに委譲しても全体の責務の量が減るわけではありません。  
しかしそれによってViewControllerプログラムの煩雑さはなくなりコードが画一化され、読みやすくなります。    
これは具体的な操作を他コンポーネントに委譲することでそれまでViewControllerで行っていた多様な処理が「イベント処理」、すなわち「他コンポーネント(Presenter/View/Alert/Router等)への指示」へと集約されるからです。  

ここではいくつか例を紹介することでViewControllerのコードが画一化され、読みやすくなったことを確認します。  
##### 例1:Viewの操作
通常の実装であれば各ViewコンポーネントはViewControllerに宣言されているためそれらを操作する場合はViewControllerが直接行う必要があります。  
例えばBarViewのインタラクションの状態が変更される場合にその画像や背景色も変更される処理はViewControllerに以下のように実装することになります。  
  
コード例: Viewの操作をViewControllerで直に行った場合の実装
```   
   // userInteractionEnabledはインタラクションが有効かどうかを示す引数とする


   barView.isUserInteractionEnabled = userInteractionEnabled
   barView.image = userInteractionEnabled ? 有効の場合の画像 : 無効の場合の画像
   barView.backgroundColor = userInteractionEnabled ? UIColor.clear : UIColor.black.withAlphaComponent(0.6)
```
上記のコードはたった3行ですが、ViewContrller内で直接View操作を行うと似たような処理があちこちで実装されるのでコード全体が煩雑化していきます。  
  
これに対して今回提案した設計では各ViewコンポーネントはRoot Viewに宣言され(Root Viewに関しては後ほど詳細を説明します)、それらの直接的な操作もViewControllerではなくRoot View内で行われています。      
従ってViewController側の実装は以下のようになります。  
  
コード例: Viewの操作をRoot Viewに委譲した場合のViewControllerの実装
```
   // rootViewはViewControllerの画面全体のviewを参照している
   // userInteractionEnabledはインタラクションが有効かどうかを示す引数とする


   self.rootView.setBarViewInteraction(enabled: userInteractionEnabled)
```
こちらの設計ではBarViewのインタラクションの状態が変更された際のViewコンポーネントの操作はRoot View側に実装しているためViewControllerはRoot Viewの処理を呼び出すだけです。  

##### 例2:Alertの表示
続いてAlertの表示を例に説明します。  
Alertの表示をViewControlerで直に行った場合以下のような実装になります。  
  
コード例：  Alertの表示をViewControllerで直に行った場合の実装
```
   // dataはPresenterから渡された引数とする

   let alert: UIAlertController = UIAlertController(title: "データの保存確認",
                                                    message: "データを保存してもいいですか？",
                                                    preferredStyle:  UIAlertControllerStyle.Alert)
    let defaultAction: UIAlertAction = UIAlertAction(title: "OK",
                                                     style: UIAlertActionStyle.Default, 
                                                     handler: { (action: UIAlertAction!) -> Void in
                                                        presenter.save(data)
                                                     })
    let cancelAction: UIAlertAction = UIAlertAction(title: "キャンセル",
                                                    style: UIAlertActionStyle.Cancel,
                                                    handler:nil)
    
    alert.addAction(cancelAction)
    alert.addAction(defaultAction)

    presentViewController(alert, animated: true, completion: nil)
```
Alert表示の場合、View操作とは異なり単体でもそれなりのコード量になってコードの肥大化・煩雑化の原因になり得ます。  

なのでAlert表示も外部のAlertコンポーネントに委譲します。  
AlertコンポーネントではAlertStrategyという表示したいアラートの情報を持ったデータを引数として受け取ることでAlertを表示します。(アラートの実装の詳細については次編でお話します。)  
その結果ViewController側のAlert表示の実装は以下のようになります。  
  
コード例:Alertの表示をAlertコンポーネントに委譲した場合のViewControllerの実装
```
   //alertStarategyはアラート表示に関する情報を持った引数
   
   self.alert.show(strategy: alertStrategy)
```
こちらも先程のViewの操作を委譲した場合と同様にViewController側の実装は他コンポーネントへの処理依頼のみとなってとても簡潔になりました。  

##### 具体的な処理を委譲することでViewControllerの処理は画一化される
Viewの操作とAlertの表示を例にViewControllerに直接実装した場合と外部に責務を委譲した場合のコードを比較してみました。  
改めてそれぞれの変更後のコードだけを再掲します。  
  
コード例: Viewの操作を委譲した場合のViewControllerの実装
```
   self.rootView.setHogeViewInteraction(enabled: userInteractionEnabled)
```  

コード例:Alertの表示を委譲した場合のViewControllerの実装
```
   self.alert.show(strategy: alertStrategy)
```  
上記2つの変更後のコードを見ると、それまでViewControllerで実装していた様々な処理が画一的になったのがわかると思います。       
具体的な処理を外部のコンポーネントに委譲することでほとんどのケースに対するViewController側の処理は他コンポーネントの単一のメソッドの呼び出し、もしくはプロパティの操作のみとなり単純化されています。        
 
#### &#12302;命令的プログラミングと宣言的プログラミングが混在する&#12303;問題の解決
処理を委譲するということは、ViewControllerは外部コンポーネントのメソッドもしくはプロパティを通して処理を依頼するということです。  
そのためそれら外部コンポーネントの設計と命名を適切に行うことでViewControllerの大部分において宣言的プログラミングによる実装可能になります。  

一般的なViewControllerの開発において宣言的プログラミングを難しくさせている原因としてViewの存在がありました。  
iOSの基本的な設計ではViewはViewController自身に宣言することになっているため、Viewの操作の際にはUIViewのプロパティ・メソッドにアクセスして命令的プログラミングで実装する箇所がそれなりに出てききてしまっていました。          
しかし後ほど実装については詳しく説明しますが、今回の設計ではViewControllerからViewの責務も完全に切り離しています。  
そのためViewController自身のAPIにアクセスする一部の処理を除いて、ほとんどの箇所において宣言的プログラミングでの実装が可能になっています。  

このようにViewControllerのプログラムが宣言的になることは、先程の責務の統一化と合わさってViewControllerプログラムの可読性を大きく向上させます。      

#### 『ViewControllerは対応する画面の仕様に大きく影響を受ける』問題の解決
##### プログラム構造の画一化
これは既に[この章の冒頭](#イベントを中心に据えることでViewControllerのプログラム構造も決まってくる)でお伝えしたとおりです。  
ViewControllerのコア責務を「イベント処理」と定義することで「入力イベントの処理」と「出力イベントの処理」が主要な責務として明確になりその基本的な構造が決定されます。    
また実際の実装ではこれらの「入力イベントの処理」と「出力イベントの処理」はさらにViewコンポーネント毎に分割されるため、ViewController内の自身のタスク関連する箇所を特定しやすくなり今まで以上にスムーズな開発が行えるはずです。    
例えばFooViewControllerにはRoot View以外に「Aコンポーネント」「Bコンポーネント」「Cコンポーネント」と3つのViewコンポーネントがあるとして、その場合ViewControllerの構造は以下のようになります。  
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/実践におけるViewControllerの入出力イベントの構造.png" alt="実践におけるViewControllerの入出力イベントの構造" width=60% > 

##### プログラムサイズの画一化
ViewControllerの具体的な処理を外部に委譲するようになるとそのプログラムサイズは処理するイベントの量によって決定され、それによりほとんどのViewControllerプログラムは200~300行の範囲に収まっていきます。      
その理由はプロダクト(UX)的な観点から画面毎のイベント量には上限があるからです。  
ここでの画面のイベント量は画面の機能数とも言い換えることができると思いますが、一般的に一つの画面にあまりに多くの機能を搭載することはユーザーを混乱させてしまいます。  
そのためある画面のイベントの量が他の画面と比べて極端に少なくなることはあっても反対に極端にイベント量が多くなることは考えにくく、一部では100行以下の小さなものも出てくると思いますが大体のViewControllerのプログラムサイズは200&sim;300行あたりに収まるはずです。  

##### ViewControllerの責務をイベント処理に集中させ、プログラムサイズをコンパクトにする意味はあるのか
このように外部に処理を委譲してViewControllerをコンパクトにしてもアプリ全体のコード量が減るわけではなく意味がないのではないかと思う人もいるかもしれません。    
しかしViewControllerは画面開発の要であり、開発者にとってはその画面全体の機能(イベント)を示す役割を担っていることは間違いありません。            
そのためViewControllerのプログラムサイズをコンパクトにしてその機能(イベント)のみを宣言的プログラミングで記述する形式は非常に合理的であると考えています。  

## 実装方法
ViewControllerのコア責務を「UI/システムから(への)イベントを処理」と定義することで既存の問題が解決されることを説明しました。  
ここではその実現のために実際にViewControllerをどのように実装すれば良いか説明します。   
### ViewControllerからどのようにViewを切り離すか
ViewControllerのコアを「イベント処理」と定義することで具体的な操作をViewControllerの外部に委譲すると説明しましたが、その実装の際ポイントとなるのは**ViewControllerとViewの切り離し**です。  
View以外の責務の外部化については特別なテクニックが必要なわけではないので補論で簡単に取り上げる程度に留めています。(しかし外部化以上の話、つまり外部化した上でどのように設計するべきかについては次編以降で必要に応じて説明しています。)       
ViewControllerとViewの分離の話に戻すと、[『命令的プログラミングと宣言的プログラミングが混在する』問題の解決](#命令的プログラミングと宣言的プログラミングが混在する問題の解決)でも述べたとおりUIKitを利用したiOSアプリ開発ではViewControllerとViewは密接に関わっておりそれらを完全に切り離すのが難しいです。      
そのためもともとViewControllerが担っていた責務を外部に委譲する上で、どのように汎用性のある形でViewを切り離したViewControllerを設計するかが重要になってきます。  

### ViewControllerのRoot&nbsp;View型をジェネリクスで指定する
結論から言うとViewControllerからViewを切り離すためにジェネリクスを利用して自身のRoot Viewのクラスを指定できるベースViewControllerを設計しました。  
アプリケーション内の全てのViewControllerにはこのベースViewControllerクラスを継承させます。    

ちなみに以下のベースViewControllerクラスの実装はInterface Builder(Storyboard/Xib)を利用せずコードのみでViewを生成・管理することを前提としています。  
そのためInterface Builderを利用する場合は多少実装が変わると思いますが、ただその場合もジェネリクスを利用してRoot Viewクラスを指定できる仕様は必要になります。    

```
protocol AppView: UIView {
    func setup()
}

class ViewController<View: AppView>: UIViewController {
    var rootView: View {
        return self.view as! View
    }
    
    init() {
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
        
    final override func loadView() {
        self.view = View()
        self.rootView.setup()
    }
}


```
デフォルトのUIViewControllerからViewを切り離すことが難しかった原因はUIViewControllerのviewプロパティにあります。    
UIViewControllerではviewプロパティを通して自身のRoot Viewにアクセスできる仕様となっていましたが、このviewプロパティはUIViewクラスのインスタンスであるためそこからアクセスできるのはUIViewのメソッド・プロパティのみです。      
そのためRootView内で表示する各ViewコンポーネントはViewController内で宣言され、そこで直接操作されることが基本でした。  

しかし上記で実装したベースViewControllerではジェネリクスを利用してRoot Viewのクラスを指定しています。(Root Viewに指定するクラスはAppViewというプロトコルに準拠している必要があります。)   
これによりViewControllerのrootViewプロパティを通して自身が指定したRoot Viewクラスのインスタンスにアクセスできるようになるため、ViewControllerではなくRoot Viewクラスに各Viewコンポーネントの宣言しても問題なくViewの処理を行えるようになりViewControllerとViewの切り離しが可能になります。      

> 補足:  
> ベースViewController内部では初期化時にviewプロパティに自身が指定したRootViewクラスのインスタンスを代入しており、rootViewプロパティではviewプロパティをRootView型に強制キャストしています。  
> そのためViewControllerは自身のviewプロパティにRootViewクラス以外のインスタンスを代入する恐れがある場合には安全ではなくなってしまうのですが、一般的な仕様を考えればViewControllerのRoot Viewが途中で差し替えられるゼロなので心配する必要はないと思います。  


> 補足:  
> このベースViewControllerのrequired init?(coder: NSCoder)メソッドには  
> fatalError("init(coder:) has not been implemented")が実装されていますが、これは厳密にはよろしくないようです。  
> 詳しくは[こちらの記事](https://qiita.com/coe/items/9723381ec0046fd8d8ad)を読んで欲しいのですが、
> どうやらinit?(coder: NSCoder)はInterface Builder以外でもViewController復元時に呼び出されるためしっかり実装しておいた方が良いみたいです。  
> そのため私はまだしっかり対応できていないため手をつけていませんが、  
> required init?(coder: NSCoder)メソッドの実装はこの例を真似せず各自調べて適切に実装しましょう。  
> この記事では他にもViewControllerやViewの実装例が紹介されますが、上記のinit?(coder: NSCoder)の実装に関してはそれらに対しても同様です。　　

## 実装例
最後に先程のベースViewControllerを利用してViewControllerからViewを切り離した簡単なアプリ実装例を紹介します。  

紹介するのは以下のようにボタンを押すことで画面全体の色が切り替わる簡単なアプリです。(本来ダークモードへの切り替えはこのようにアプリ内のボタンを押して実現するものではありませんが、簡易的な例で示すため今回はこのようなアプリを素材とします。)  
以降このアプリをHogeアプリと呼ぶことにします。  
ちなみにHogeアプリではViewロジックの実装にPresenterを利用していますが、サンプルプロジェクトではViewModelを使っており個人的にPresenterの実装に慣れていません。  
そのため細かい箇所でよろしくない実装があるかもしれないため、このHogeアプリからPresenterの実装を参考にするのはオススメしません。  

<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/darkModeExample.gif" alt="Hogeアプリ" width=30% > 

### Hogeアプリの画面構成

Hogeアプリの画面構成は以下の通りです。  
各Viewから伸びてる線の先にある情報はプログラム上でのインスタンス名とそのクラス名です。  
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter4(ViewController編)/Images/ViewControllerからViewを切り離したアプリ例の画面構成.png" alt="ViewControllerからViewを切り離したアプリ例の画面構成" width=60% > 

### Hogeアプリのプログラム
画面構成がわかったところでHogeアプリのプログラムについて見ていきます。  
最初にHogeViewControllerのRoot Viewに当たるHogeRootViewクラスの実装から見てみましょう。  

#### HogeRootViewのプログラム
HogeRootViewクラスの実装  
```
final class HogeRootView: UIView, AppView {
    private(set) lazy var hogeLabel: UILabel = {
        let label: UILabel = .init(frame:.zero)
        label.translatesAutoresizingMaskIntoConstraints = false
        self.addSubview(label)
        NSLayoutConstraint.activate([
            label.centerYAnchor
                .constraint(equalTo: self.safeAreaLayoutGuide.centerYAnchor),
            label.centerXAnchor
                .constraint(equalTo: self.safeAreaLayoutGuide.centerXAnchor),
            label.heightAnchor.constraint(equalToConstant: 300),
            label.widthAnchor.constraint(equalTo: self.safeAreaLayoutGuide.widthAnchor, multiplier: 0.6)
        ])
        label.text = """
                    Hoge
                    Hoge Hoge
                    Hoge Hoge Hoge
                    Hoge Hoge Hoge Hoge ...
                    Hoge Infinity!
                    """
        
        label.numberOfLines = 0
        return label
    }()
    
    private(set) lazy var hogeViewColorChangeButton: UIButton = {
        let button: UIButton = .init(frame:.zero)
        button.translatesAutoresizingMaskIntoConstraints = false
        self.addSubview(button)
        NSLayoutConstraint.activate([
            button.topAnchor
                .constraint(equalTo: self.hogeLabel.bottomAnchor, constant: 15),
            button.centerXAnchor
                .constraint(equalTo: self.safeAreaLayoutGuide.centerXAnchor),
            button.heightAnchor.constraint(equalToConstant: 50),
            button.widthAnchor.constraint(equalToConstant: 200)
        ])
        
        return button
    }()
    
    override init(frame:CGRect) {
        super.init(frame: frame)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    
    func setup() {
        _ = self.hogeLabel
        _ = self.hogeViewColorChangeButton
    }
    
    func setColorMode(lightMode: Bool) {
        if lightMode {
            self.backgroundColor = .white
            self.hogeLabel.textColor = .black
            self.hogeViewColorChangeButton.backgroundColor = .lightGray
            self.hogeViewColorChangeButton.setTitle("Hoge Dark Mode!!!",
                                                    for: .normal)
            self.hogeViewColorChangeButton.setTitleColor(.black, for: .normal)
        } else {
            self.backgroundColor = .black
            self.hogeLabel.textColor = .white
            self.hogeViewColorChangeButton.backgroundColor = .darkGray
            self.hogeViewColorChangeButton.setTitle("Hoge Light Mode!!!",
                                                    for: .normal)
            self.hogeViewColorChangeButton.setTitleColor(.white, for: .normal)
        }
    }
}

```

HogeRootViewはViewControllerからViewの責務を切り離すことが目的であり、そのためこのクラスにはそれまで基本的にViewControllerの責務であったViewコンポーネントの宣言、Viewの操作の処理が実装されています。  

具体的にはhogeLabel・hogeViewColorChangeButtonの宣言と初期化処理、また画面の色のモード(ライト/ダーク)が変更された時のView操作を行う`setColorMode(lightMode: Bool)`メソッドが実装されています。  

そしてHogeRootViewはViewControllerのRoot Viewであることを明示するため、[ルートViewControllerの説明](#ViewControllerのRootView型をジェネリクスで指定する)でも紹介したAppViewプロトコルに準拠しています。  
Root View内で何かセットアップが必要な場合は、このAppViewプロトコルの`setup()`メソッド内に記述します。  
このメソッドはViewControllerの`loadView()`メソッド内で呼び出されます。  

HogeRootViewではsetupメソッドでhogeLabelとhogeViewColorChangeButtonにアクセスしてそれぞれの遅延初期化処理を呼び出しています。  

#### HogePresenterのプログラム
この記事ではViewController-Viewの関係が主題ではありますが、アプリ全体の様子をより詳しく理解してもらうためPresenter説明もします。    
HogeViewControllerより先にHogePresenterの説明をするのはHogeViewControllerはPresenter側に定義しているHogePresenterOutputに準拠しているためです。  

HogePresenterクラスの実装  
```
protocol HogePresenterInputs: AnyObject {
    init(output: HogePresenterOutputs)
    func changeColorMode()
}

protocol HogePresenterOutputs: AnyObject {
    func updateColorMode(lightMode: Bool)
}

final class HogePresenter: HogePresenterInputs {
    var isLightMode: Bool = true
    private weak var output: HogePresenterOutputs!
    init(output: HogePresenterOutputs) {
        self.output = output
        output.updateColorMode(lightMode: self.isLightMode)
    }
    
    func changeColorMode() {
        self.isLightMode.toggle()
        output.updateColorMode(lightMode: self.isLightMode)
    }
}
```

クリーンアーキテクチャ編でも説明した通り、テスト等の観点から依存関係においてはプロトコルを積極的に利用した方が良く、ここでもそのようにしています。  
具体的に定義しているのはHogePresenterInputsとHogePresenterOutputsという2つのプロトコルです。  
HogePresenterInputsは画面から流れてきた入力イベントを処理する機構であり、これはHogePresenter自身が準拠します。  
そしてHogePresenterOutputsはPresenterの出力を受け取って処理する機構であり、こちらはHogeViewControllerが準拠します。  
全体の処理の流れでいうとHogeViewController->HogePresenterInputs(HogePresenter)->HogePresenterOutputs(HogeViewController)という感じになります。    

さて、HogePresenterクラスに話を移すと、このクラスでは画面の色の状態(ライト/ダーク)を管理しており、それが変更された時にHogePresenterOutputsに通知しています。    
具体的にはHogePresenterInputsで定義した`changeColorMode()`メソッド呼び出し時に現在の状態を変更してHogePresenterOutputsに通知しています。  
また色の状態の管理はこのHogePresenterのみで行われるべきなので、HogeViewControllerもHogeRootViewも最初の色の状態がわかりません。     
そのためHogePresenterでは初期時の色のモードを`init()`内でHogePresenterOutputsに通知します。 

> 補足:  
> 依存関係においてはプロトコルを積極的に利用した方が良いと述べましたが、  
> ViewController-View間ではAppViewという包括的なプロトコルしか利用しておらずHogePresenterInputsのように個々のコンポーネントに対応したプロトコルは定義していません。  
> これはViewController-View間でコンポーネントを差し替える必要があまりないからです。  
> 基本的にViewのレイアウトや挙動に関するテストはデザイン側のソフトウェアを使って行うことが可能ですし、実装の差し替えのためにプロトコルが必要になるケースは考えにくいです。  
> そのためViewController-View間ではAppViewプロトコルのみで十分で個々のRootView毎にプロトコルを定義する必要はないと思います。    

#### HogeViewControllerのプログラム
最後にここでの主題であるViewControllerの実装を見てみます。  

HogeViewControllerクラスの実装
```
final class HogeViewController<Presenter: HogePresenterInputs>: ViewController<HogeRootView>, HogePresenterOutputs {
    
    private var presenter: Presenter!
    
    override init() {
        super.init()
        self.presenter = Presenter(output: self)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    //MARK: HogeViewController Inputs
    override func viewDidLoad() {
        super.viewDidLoad()
        self.rootView
            .hogeViewColorChangeButton
            .addAction(
                UIAction(handler: { [weak self] _ in
                    self?.presenter.changeColorMode()
                }),
                for: .touchUpInside
            )
    }
    
    
    //MARK: HogeViewController Outputs
    func updateColorMode(lightMode: Bool) {
        self.rootView.setColorMode(lightMode: lightMode)
    }
}
```

> 補足:  
> [記事内で以前紹介したViewControllerのプログラム構造の例](#イベントを中心に据えることでViewControllerのプログラム構造も決まってくる)では「入力イベント処理」と「出力イベント処理」はそれぞれ別にextensionで書き出されていましたが、今回は全てViewController本体に実装されています。  
> これはジェネリクスを利用したクラスではObjective-C由来のメソッドをextension内で実装できないため、「入力イベント処理」にあたるviewDidLoadメソッドをViewController本体に実装せざるをえなかったからです。  
> ただこのような形式の違いは些細なことであり、大切なのはViewControllerが「初期化処理/入力イベント処理/出力イベント処理」と単純な構造を取っていてそれらの実装形式がアプリケーション全体で統一されていることです。  
> それさえ守られていたならば開発者にとってそのViewControllerプログラムは十分に見やすいものになっているはずです。     

##### HogeViewControllerの外部構造
まずその外部構造から見ていこうと思いますが、どれも簡単な説明になるため以下で箇条書きで記します。  
1. [記事内](#ViewControllerのRootView型をジェネリクスで指定する)で紹介したベースViewControllerを継承している(具体的にはRootViewにHogeRootViewクラスを指定したViewController&lt;HogeRootView&gt;クラスを継承)  
2. HogePresenterの出力先となるためHogePresenterOutputsに準拠
3. 入力イベントを処理するHogePresenterInputsの実体型をジェネリクスで指定

3については後ほど補論で説明しますが、ジェネリクスでHogePresenterInputsの実体型を指定する手法を取ることで初期化時におけるDI(Dependency Injection)を可能にしています。  

##### HogeViewControllerの内部構造
次にHogeViewControllerの内部構造を見ていきますが、その概要を簡単に説明すると`init()`で初期化を行い、実装内のコメントでも記した通りviewDidLoad()で入力イベント処理、updateColorMode(lightMode: Bool)で出力イベント処理を行なっています。  
##### HogeViewControllerの初期化処理
初期化処理から順にその詳細を説明していくと、この初期化処理では外部からコンポーネントは一切注入していません。  
その理由はHogeViewControllerがジェネリクスで指定したPresenter以外に外部コンポーネントを必要としていないからです。  
Presenter型が準拠しているべきHogePresenterInputsプロトコルの定義を見てみると
```
init(output: HogePresenterOutputs)
```
というHogePresenterOuputsを引数に取るinit処理が定義されていることがわかり、また既に説明した通りHogeViewControllerはHogePresenterOutputsに準拠しています。  
そのためHogeViewControllerのジェネリクスで指定したPresenter型は外部から注入せずとも、HogeViewController内の初期化処理内で自身を引数とすることで生成可能となっています。  


##### HogeViewControllerの入力イベント処理
次に入力イベント処理の実装ですが、これはほとんどのケースにおいてViewControllerがデフォルトで持っているviewDidLoad()メソッド内で行えば良いと思います。  
ViewControllerにおける入力イベント処理とは言い換えれば画面内のViewもしくはシステムにおいて何らかのアクションが起こった時にPresenter(ViewModel)にある特定の処理を呼び出すように登録することです。  
そのためViewControllerのライフサイクルにおいて最初に一度だけ呼ばれるviewDidLoad()メソッド内でその処理登録を行うのが合理的だと思います。  
HogeViewControllerにおいては`viewDidLoad()`内でHogeRootViewのhogeViewColorChangeButtonボタンがタップされた時、HogePresenterInputs(HogePresenterクラス)の`changeColorMode()`が呼び出されるように登録しています。  

##### HogeViewControllerの出力イベント処理
最後に出力イベント処理の実装ですが、このHogeViewControllerに出力イベント処理の実装はHogePresenterOutputsに準拠することと同義です。  
ViewControllerにおける出力イベント処理とは言い換えればPresenterもしくはViewModel等からの出力に対応することであり、それは今回の例ではHogeViewContrllerがHogePresenterOutputsに準拠することを意味します。  
具体的にはHogePresenterOutputsに準拠してその`func updateColorMode(lightMode: Bool)`メソッド内で
```
   self.rootView.setColorMode(lightMode: lightMode)
```
と自身のRoot Viewに当たるHogeRootViewのメソッドを呼び出しています。  
ここに今回のViewController設計の特徴が一番出ていると思います。  
一般的なViewControllerの設計では具体的なViewの操作はViewController自身が行い、命令的プログラミングで実装していました。        
しかし今回の設計ではRootView側に具体的なViewの操作を実装しているため、ViewController側ではRootView側の処理の呼び出しのみ行っています。    

> 補足:  
> この設計ではPresenterからViewController(HogePresenterOutputs)のupdateColorMode(lightMode: Bool)メソッドを呼び出し、またそこからHogeRootViewのsetColorMode(lightMode: Bool)メソッドを呼び出していますが、これがパフォーマンス的によくないのではと思う人がいるかもしれません。  
> 確かにここではPresenter->ViewControlelr->Root Viewと処理依頼を垂れ流してオーバーヘッドが発生しているように見えます。  
> しかしまだ確認できてないので断言はできませんが、恐らくコンパイル時の最適化により最終的なプロダクトコードではオーバーヘッドが起こらないと思います。(そのうち確認します。)    
> また先にお伝えした通りViewControllerにおいて一番重要な役割は画面全体の機能(イベント)を開発者に伝えることです。  
> そのため仮にオーバーヘッドが起こっていたとしても、処理を外部に移譲してViewControllerの可読性を上げられるならば十分にその価値があると思います。      

## Hogeアプリを振り返る
ViewControllerからViewを切り離した実装例としてHogeアプリを見てきました。  
非常に単純なアプリであったためViewControllerからViewを切り離すメリットがイマイチよく伝わらなかった人もいるかもしれません。  
しかし、UIをInterface Builderではなくコードで生成したという留意が必要ですが、こんな小さなアプリでさえ今回の設計によって本来ViewControllerに実装するはずだった50程のコードをRootViewに書き出すことができています。(hogeLabelとhogeViewColorChangeButtonの宣言と初期化処理、そしてsetColorMode(lightMode: Bool)メソッドは本来であればViewControllerに記述されていたはずです。)    
実際のプロダクトではほとんどのViewControllerにおいて今回以上のViewコンポーネントの宣言とその操作が必要であるのは間違いありません。    
その時にそれら全てをRootViewに書き出し、またその他の具体的な処理に関してもViewControllerの外部のコンポーネントに移譲したならば、Interface Builderを利用していたとしても各ViewControllerで少なくとも100行以上のコードが外部へと書き出されるはずです。  
そのように考えた場合、具体的な操作を外部に移譲することでViewControllerの開発のしやすさ(特にコードの見通し、可読性)が大きく変わることが想像できるのではないでしょうか。    
またViewControllerは誇張するまでもなくアプリ開発の一番の要です。  
そのようなViewControllerの開発容易性が向上は1コンポーネントの開発容易性が向上以上の価値があると思います。  

## 本記事のまとめ
- Appleのドキュメントによれば、ViewControllerの責務は「Viewの管理(Control)」と要約できる
- しかしそのようなViewControllerの定義は概念上は有効だが、抽象的すぎるあまり実装上はコードが煩雑になる等の問題を起こす
- そのため本記事の設計ではViewControllerのコア責務を「イベント処理」と定義して、「View、Alert、遷移等の具体的な操作」をViewControllerの外部に移譲する
- そうした再定義によりViewControllerは概念上のみならず実装上も単純な構造を持つようになり、プログラムの可読性・変更容易性は大きく向上する


## 補論:コードから設計を学ぶことの限界
本記事内で説明したようにViewControllerの責務は採用しているアーキテクチャやアプリ規模等によって変わりますが、このように状況で責務が変化する性質は程度こそ小さいものの他のコンポーネントも持っています。      
そのため設計論を押さえずにコードから設計を学ぼうとすると混乱してしまう場合があります。    

例えばSwift UIの公式チュートリアルではLandmarkというDomainオブジェクトがSwift UIとCoreLocationのフレームワークをインポートしてそのAPIを利用していますが、このようなオブジェクト構造は本来の設計論的な観点からは正しくありません。   
設計の基本原則からするとDomainオブジェクトはUIや外部フレームワークとは切り離されているべきです。
```
import Foundation
import SwiftUI
import CoreLocation

struct Landmark: Hashable, Codable {
    var id: Int
    var name: String
    var park: String
    var state: String
    var description: String

    private var imageName: String
    var image: Image {
        Image(imageName)
    }

    private var coordinates: Coordinates
    var locationCoordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(
            latitude: coordinates.latitude,
            longitude: coordinates.longitude)
    }

    struct Coordinates: Hashable, Codable {
        var latitude: Double
        var longitude: Double
    }
}
```
<sup>引用元： [https://developer.apple.com/tutorials/swiftui/building-lists-and-navigation](https://developer.apple.com/tutorials/swiftui/building-lists-and-navigation)</sup>  
しかしこの例は決して間違っているわけではなく、Appleはチュートリアルの趣旨がSwiftUIの実装方法にあることを考慮してあえてDomainオブジェクトを便宜的に設計しているのだと思います。  
  
このように私たちが普段ネットで目にするコードは特定の目的に沿って便宜的に実装されていて、設計論的観点からは正しくない場合が多いです。        
そのため設計を理解を深めるためにはコードから学ぶだけではなく、理論を通して様々な状況に通底する原理・原則を学んでいく必要があります。  


## 補論:View以外の責務の委譲
本記事では本来ViewControllerに属していた責務を外部に委譲する設計を提案しましたが、その具体的な方法についてはView以外特別なテクニックを要するわけではないので言及はしませんでした。  
しかしそれらに全く言及しないのもおかしいと思うので、ここでは本記事内で触れなかった責務の委譲方法について簡単に説明します。 
### Router
まず基本的に[こちらのVIPERの記事](https://qiita.com/hicka04/items/09534b5daffec33b2bec)で述べられているようにRouterにViewControllerを渡してそちらで通常の遷移処理を行うだけです。  
ただ遷移処理のRouterへの外部化ではなく遷移処理自体の実装に関しては独自な設計をしている箇所があり、その詳細は画面遷移編でお話しします。  
### Alert 
AlertもRouterと同じです。  
Alertの表示はつまるところViewControllerのpresentメソッドで行うため、大きく言えば遷移の一種です。  
そのため機構としては別なものとして扱っていますがViewControllerからの責務の外部化についてはRouterと同じ方法になります。  
Alertについても独自な実装を施している箇所がありますが、その詳細についてはView/Alert編でお話しします。  
### Notification Center
NotificationCenterについてはデフォルトでViewControllerの外にあるので、責務の外部化については特に話すことはありません。  
ただNotificationCenterの実装については2点ほど説明すべき点があります。 
#### Notification Centerからの通知処理の実装方法
まずNotificationCenterからの通知を処理する際にはCombineフレームワークの利用をオススメします。
Combineを利用することで通常よりもスマートな実装が可能になるからです。  
#### Notification CenterとViewController間の連携
##### Notification Centerへの通知の登録
NotifactionCenterへの通知の登録はViewControllerの初期設定時に行います。    
具体的にはViewControllerのviewDidLoadメソッドで行われることが多いでしょう。  
##### Notification Centerへの通知処理(入力処理)
ViewControllerからNotificationCenterへ通知イベントを投げる場合は、ViewControllerの出力処理
送信(入力)は画面の出力の結果として行われるため、と連携する形となります。  
 
##### Notification Centerからの通知処理(出力処理)
NotificationCenterからの通知処理(出力)はViewControllerの入力処理と連携します。  
これは考えてみるとわかるのですが、ViewControllerはNotificationCenterから通知を受け取ってそれを自身の画面で対応するためPresenter(ViewModel)に処理を依頼します。      
そのためNotificationCenterからの通知処理(出力)が連携するのはViewControllerの出力処理ではなく入力処理になります。  
  
### Delegate/DataSource
次にCollectionViewのDelegate/DataSourceについて説明します。  
ちなみに直接言及はしませんが、ここでのCollectionViewの内容はそのままTableViewにも当てはまります。  
またここでは概要のみ説明しますが、DataSourceに関しては次のView/Alert/Data Source編でもう少し詳細に触れる予定です。  
#### Delegate-ViewControllerの入力処理、DataSource-ViewControllerの出力処理
まず基本的なことから説明すると、CollectionViewのDelegateの実装はViewControllerにおける入力処理、DataSourceの実装は出力処理と関係しています。    
これはDelegateに定義されているのがセルのタップ等CollectionViewにおけるイベント発生時の処理に関するメソッドであること、またDataSourceに定義されているのがセルの表示に関するメソッドであることを考えればわかると思います。  
####  Delegate(入力処理)の実装
Delegateの実装に関してはいくつかパターンがあり、それぞれ一長一短あるためどの方法を採用するかは開発者自身で決めるのが良いでしょう。  

##### ViewControllerに直に実装する
DelegateはViewControllerに直接実装するのも一つの手です。    
本記事ではViewControllerの責務をイベント処理の機構として具体的な処理を外部に委譲することを提案しましたが、基本的にCollectionViewのDelegateメソッドはその一つ一つがCollectionViewからの入力処理に対応しており、また各メソッドの実装は対応するPresenter(ViewModel)側のメソッドの呼び出しのみとなるはずです。    
そのためDelegateを直接ViewControllerで実装しても本記事が目的とする統一的なViewControllerの構造が破壊されることも、ViewController内の宣言的プログラミングの方針が破られる恐れもあまりありません。  

ただ以下のコードで示すようにViewControllerをUICollectionViewDelegateに準拠させた場合、CollectionView以外のイベント入力処理はviewDidLoad()メソッド内でCollectionViewのイベントはDelegateメソッドで実装することになり若干統一性が落ちます。  
またDelegateメソッドには純粋な入力処理以外にも入力処理に関する設定をする(インタラクションの管理)メソッドもあり、ここでロジック判定が必要な場合には手続き的プログラミングで実装する必要があるため他の方法を採用する等の検討が必要です。  

HogeアプリでHogeViewControllerがUICollectionViewDelegateに準拠した場合のコード例
```
final class HogeViewController<Presenter: HogePresenterInputs>: ViewController<HogeRootView>, HogePresenterOutputs, UICollectionViewDelegate {
    ...
    
    //MARK: HogeViewController Inputs
    override func viewDidLoad() {
        super.viewDidLoad()
        self.rootView
            .hogeViewColorChangeButton
            .addAction(
                UIAction(handler: { [weak self] _ in
                    self?.presenter.changeColorMode()
                }),
                for: .touchUpInside
            )
    }
    
    func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) {
        // セルTap時のイベントに対応したPresenterメソッドの呼び出し
    }
    
    func collectionView(_ collectionView: UICollectionView, didHighlightItemAt indexPath: IndexPath) {
        // セルハイライト時のイベントに対応したPresenterメソッドの呼び出し
    }
    
    func collectionView(_ collectionView: UICollectionView, shouldSelectItemAt indexPath: IndexPath) -> Bool {
        // セルのインタラクションの有無
        true
    }
    ...
```


##### Delegateのラッパークラス/Delegateに準拠したベースCollectionViewクラスを実装する
Delegateをラップしたクラスを実装する方法もあります。    
この場合ViewControllerの入力処理を行う箇所でラッパークラスに各Delegateのイベントに対応する処理をクロージャで渡す方法が想定されます。  
似たようにDelegateに準拠したCollectionViewクラスを実装して、アプリ内でそれをベースクラスとして利用する方法もありでしょう。    

これらの方法ではラッパークラスで各Delegateメソッドに対応するメソッドもしくは変数を再定義する必要があるため手間がかかりますが、ViewController上での記述形式に関してはViewControllerに直に実装するに比べると統一感が生まれると思います。  

##### RxCocoaを利用する
RxCocoaを利用するのも一つの解決策です。  
RxCocoaは外部ライブラリなので好き嫌いはあると思いますが、UI層自体がiOSアプリではUIKitフレームワークなのでここで外部ライブラリを利用することは設計論的には特に問題ありません。  
何よりRxCocoaを利用することで以下のようにDelegateメソッドをCollectionView自身の一部のように扱うことができるようになり、さらに先程の自作ラッパークラスを実装する方法のように開発の手間が増えることもありません。    
```
collectionView.rx.itemSelected
```
 
今回のサンプルプロジェクトではこのRxCocoaを利用してDelegateメソッドを利用しています。  
ただこのサンプルプロジェクトではアプリ全体でRxを利用していたためRxCocoaの採用を躊躇う理由がありませんでしたが、他の技術構成をとっているアプリでは全体との兼ね合いを踏まえて慎重に検討する必要があると思います。  

####  DataSource(出力処理)の実装
DataSource(出力処理)の実装に関してはDelegateと比べて選択肢は少なく、大きくいうとDataSourceのラッパークラスを作るしかないと思います。  
その理由を説明するとまずDataSourceのCollectionViewのセルの表示に関する実装では細かなCellの設定やロジック判定などを行います。  
そのため命令的プログラミングが必要になる箇所があり、DataSourceを直接ViewControllerに実装すると本記事で提案していた記述スタイルや責務が維持できなくなってしまいます。  
またDelegate時のように気軽にRxCocoa(とRxDataSources)を利用することもできません。  
DelegateはViewControllerの入力処理であり画面イベントの起点であるためRxCocoaの局所的な利用が可能でした。  
しかし出力処理であるDataSourceの実装でRxCocoaやRxDataSourcesを利用しようとすると基本的にDataSourceにイベントを流すPresenter(ViewModel)でもRxを利用する必要が出てくるため設計全体にも大きな影響を及ぼすことになります。  
そして仮にRxをアプリで利用して、DataSourceの実装でRxCocoaやRxDataSourcesを利用したとしても本記事が提案した内容に沿って実装するためにはDataSourceのラッパークラスは必要になってきます。  

### その他の外部化
遷移処理(Router)・Alert・Notification・(CollectionViewの)Delegate/DataSourceの責務のViewControllerからの外部化について説明しましたが、アプリ内では他にもViewControllerの外部に書き出すべき責務が発生するかもしれません。  
ただどのような責務を外部化する場合にも基本的にはこれまでの例で見てきた通り、ViewControllerで実装していた際に必要だったコンポーネント(ViewController自身を含む)を移譲先に渡してそちらの方でViewcontrollerでやっていたように実装するだけです。  

## 補論:3つのDI(Dependency Injection)
本文内の例で示したHogeアプリのHogeViewControllerでは以下のようにPresenterに当たるHogePresenterInputsプロトコルの実体型をジェネリクスで指定する形式をとっています。  
```
class HogeViewController<Presenter: HogePresenterInputs>
```
この箇所でジェネリクスを利用した目的としてはDependency Injection(以下DI)が関係しているのですが、ここではDIの説明をしながらその理由を説明していきます。    
### DIの基本的な説明
まず最初に既に多くの人が知っているとは思いますが、DIの説明からします。  
DIはあるコンポーネントがその挙動のために必要なデータを外部から渡す所作を指します(より正確には外部から必要なデータを渡すような設計を指しています。)    
要するに初期化時に引数としてデータを受け取るのも外部からデータを渡されてるわけですからDIです。    
DIという言葉に慣れていないと「初期化時に値を渡すとか基本的なことなのに"Dependency Injection"とか大層な命名して理解しづらいし、IT界隈カッコつけすぎだわ」とか思うのですが、慣れてくるとDIというだけで相手に簡単に状況を伝えることができる便利な言葉になります。  
さて話を戻すとDIの方法には3つの方法、  
1. 初期化時のDI(Constructor Injection)   
2. プロパティを通したDI(Setter Injection)  
3. メソッドを通したDI(Method Injection)  

があります。
インスタンスに外部から作用する方法としてはinit(初期化)、プロパティ、メソッドしかないのでDIの方法がこれら3つであるというのは特に驚くようなことではないと思います。  
### Constructor Injectionの優位性
ただここで強調したいのは1の方法が他の方法とは質的に異なるという点です。  
簡単にいうと一般的に1の初期化時のDIは他の方法に比べて設計的に優れています。  
その理由は初期化時のDIは依存しているデータをそのインスタンス生成時に渡しているため、挙動の原因を自身の実装内部で発見することができるからです。    
必要なコンポーネントを渡されない限り生成されない、そういう制約を構造として持つことで生成後の挙動を自身で制御しやすくしています。    
それに対して2と3の方法はインスタンス生成後にどのような方法かはわかりませんが、プロパティかメソッドを通して外部からコンポーネントが渡されることを前提としています。  
そのため自身が正しく動くかどうかは外部に依存しています。  
これは開発時のバグ温床になります。  
2と3の方法では自身の実装も、依存しているコンポーネントの実装も変更していないにも関わらず外部のどこかしらのコードを変更したことでバグが起こってしまうことがあるからです。  
もちろん元々の仕様によってプログラム起動中に依存しているデータを差し替える必要がある場合等には2と3の方法を取ることは致し方ないことです。  
しかしそうでない場合には1の方法でDIを行うのが理想だと思います。  
また1の方法の利点としてオプショナル(?)・有値オプショナル(!)型を利用する必要がない点もあります。  
2と3の方法では生成後に値を代入するためオプショナル(?)・有値オプショナル(!)型を利用する必要があります。  

### HogeViewControllerの状況
さてHogeViewControllerでなぜジェネリスクを使っているのか理解するためにDIを説明しましたが、ここでジェネリクスを使っているのは初期化時のDIを可能にするためです。  
今回のケースではHogeViewControllerはHogePresenterInputsプロトコルに準拠するHogePresenterクラスを必要としており、またそのHogePresenterはHogePresenterOutputsプロトコルに準拠するHogeViewControllerを必要としていました。  
要するにHogeViewControllerとHogePresenterはそれぞれがもう一方に依存している関係だったわけです。  
このように相互に依存しているインスタンス同士では基本的にどちらか一方でプロパティかメソッドを通したDIを採用することでその依存関係を解決します。  
しかしここではジェネリクスを利用することで初期化時のDIを可能にしています。  
具体的にはHogePresenterInputsプロトコルには以下のような初期化処理が定義されていて
```
init(output: HogePresenterOutputs)
```
この初期化処理により、HogeViewControllerでは初期化処理内で自身が指定したPresenterジェネリクスの実体型を生成することができるようになっています。  
```
final class HogeViewController<Presenter: HogePresenterInputs>: ViewController<HogeRootView>, HogePresenterOutputs {
    
    private var presenter: Presenter!
    
    override init() {
        super.init()
        self.presenter = Presenter(output: self)
    }
    ...
}
```
つまり、Presenterの実体型はHogePresenterInputsプロトコルに準拠しているためinit(output: HogePresenterOutputs)によって初期化可能であり、またHogeViewControllerはHogePresenterOutputsプロトコルに準拠しているためPresenterの実体型のインスタンスを自身を引数とすることで生成可能になっています。  
それによってHogeViewControllerはPresenterのDIを行う必要がなくなり、Setter Injection・Method Injectionを利用せずにConstructor Injectionのみで依存関係が解決することが可能になりました。  

### DI実装時はConstructor Injectionの方法を模索する
今回、HogeViewControllerのジェネリクスを通して見てきたようにDIを初期化時に行うことは(Constructor Injection)設計において重要です。  
Constructor Injectionによってそのコンポーネントは挙動に関する原因を自身の実装内に限定することができます。  

DIを初期化時に行うことで(Constructor Injection)そのコンポーネントは挙動を自分自身で制御しやすくなります。  

## 脚注
<a name="footnote1">*1</a>: 複数点あり原文(英語)も載せると見づらくなってしまうため、意訳のみ載せています。  
  
<a name="footnote2">*2</a>: アプリの仕様としてContainer ViewControllerを積極的に利用する方針にしているケースもなくはないと思いますが、全体から見ればごく限られたケースだと思います。  

<a name="footnote3">*3</a>: もちろんContainer ViewControllerのChild ViewControllerもContainer ViewControllerである場合もありえます。しかしその場合も重要なのはChild ViewControllerであるContainer ViewControllerの中にあるContent ViewControllerの設計であり、そのContent ViewControllerの設計がしっかりなされれば大抵の場合はその親であるContainer ViewControllerの責務も限定され設計しやすくなるはずであり、またその親のContainer ViewControllerも...と連鎖的に解決していくはずです。  

<a name="footnote4">*4</a>: Container ViewControllerの実装について知りたい場合はAppleの[こちら](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/ImplementingaContainerViewController.html)のドキュメントを読むことをおすすめします。日本語の記事を読みたい方は「Container ViewController Swift」等と調べればいくつか出てくるはずです。    

<a name="footnote4">*4</a>: あくまで目的はViewController(命令的プログラミング)とSwiftUI(宣言的プログラミング)の違いを把握することなので、ViewController側ではCellのレイアウト等何点か省略している実装があります。またSwiftUI側も@Stateの利用方法などベストプラクティスとは言えない実装があります。  
  
<a name="footnote5">*5</a>: ViewController(UIKit)側ではAuto Layoutをコードで実装しているためInterface Builderより煩雑になっていると言えると思います。しかし実際開発ではViewController(UIKit)側にはさらにプリフェッチ処理、差分検知、その他省略した何点かの実装が加わるためAuto Layoutをコード実装していなくてもプログラムは例よりさらに煩雑になっているはずです。 
  
<a name="footnote6">*6</a>: ここで述べている命令的プログラミングの見通しが良くないという評価は相対的なものではなくある程度客観性を持っていると思います。機械的で細々した命令的プログラミングは人間の一般的な認知能力からして見やすいモノではないはずです。  
  
<a name="footnote6">*</a>: PresenterやViewModelといったコンポーネントもUseCaseやRepositoryと比べると責務が広範囲に及んで漠然としている印象を受けますが、それでもその
性質を突き詰めると「View-BusinssLogic間のデータ変換を行うコンポーネント」であると定義することができます。そしてここから「View->Business Logicのデータ変換とBusiness Logic->Viewのデータ変換」とい構造を持ったプログラムであるべきことが見えてきます。  
  
<a name="footnote7">*7</a>: 繰り返しのようになりますがライフサイクルの仕組み等を理解することでViewControllerのプログラム構造が見えてくる面もありますし、また[脚注5](#footnote5)で挙げたPresenter(ViewModel)の定義のようにViewControllerの定義を「View-Presenter(ViewModel)間の処理を行うコンポーネント」というように定義することも可能だと思います。しかし少なくともViewControllerのドキュメントに沿った定義ではViewControllerがCollectionViewのデリゲート、データソースとして振る舞うことやそれらに関連する状態管理も許容しており、私はこれらの処理がViewControllerのプログラム構造を捉える上で切り捨てて良い瑣末なモノであるとは思えません。そのためViewControllerのプログラム構造は一般的な性質のみでは理解できず、実際にコードを見て見ないとわからないと主張しています。  
  
<a name="footnote8">*8</a>: ドキュメントで紹介した4点の責務を順番に記述しているかもしれませんし、プロダクト機能(モジュール)を構成単位としているかもしれません。恐らく一般的にはプロダクト機能を構成単位としている場合が多いと思いますが、その場合にも各画面によってプロダクト機能は異なるためそれぞれのViewControllerを実際に確認しなければその構造を把握できません。  
  
<a name="footnote9">*9</a>: ViewControllerのライフサイクルメソッドのうちloadView()はクラス本体(初期設定)に含まれていますが、ViewDidload等その他のライフサイクルメソッドに関しては入力処理に含まれるべきです。  



