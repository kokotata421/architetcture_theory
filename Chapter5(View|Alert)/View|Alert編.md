# iOSアプリでスケールしやすいアーキテクチャを考えてみた⑤-View/Alert/(CollectionView)DataSource編

この一連の記事では私的に考えたスケールしやすいアーキテクチャを紹介します。  
記事全体の構成(予定)は以下の通りです。  
(1)設計を理解するためのレイヤードアーキテクチャ編  
(2)設計を理解するためのクリーンアーキテクチャ編  
(3)アーキテクチャ概要編  
(4)ViewController編  
(5)**View/Alert/(CollectionView)DataSource編←本記事**  
(6)画面遷移編(準備中)  
(7)ViewModel(Controller/Presenter)編(準備中)  
(8)UseCaseとエラー編(準備中)  
(9)UseCaseとアプリケーションの状態管理編(準備中)  
(10)Repository編(準備中)  
(11)Domain編(準備中)  
(12)Web API/データベース編(準備中)  
(13)その他(準備中)  

本記事ではView、AlertそしてCollectionViewのDataSourceの設計を説明します。  
基本的な設計理論の知識を前提としているので、そこから知りたいという方はレイヤードアーキテクチャ編、もしくはクリーンアーキテクチャ編から読むことをオススメします。  
また本記事は単体でも読めるように構成はされていますが、内容としてはViewController編から連続したものとなっています。  
そのためなぜそもそもView、Alert、DataSourceを本記事のように設計する必要があるのか知りたい方はViewController編から読むことをオススメします。


## 前提
- この記事の設計とはアプリケーションに関するものでライブラリ等の設計は想定していません。  
- SwiftUIは扱わず、UIKitを利用して開発しています。  
- UI開発はInterface Builderを利用せずコードのみで行っています。    
- 作成したサンプルプロジェクトはMVVMをベースに考えていますが、記事内容はどんなアーキテクチャでも共通する考えとなっているはずです。  
- FluxやReduxのアーキテクチャは概念としては触れる予定ですが、サンプルプロジェクトでは採用されていません。  

## 前回までの内容と本記事の内容
前回の記事において今回の設計ではViewControllerプログラム構造や可読性の観点からViewControllerの責務のコアを「入出力のイベント処理」としてその他の具体的な処理は外部に移譲すると説明しました。  
本記事ではそのようにViewControllerの外部へと移譲されたView、Alert、DataSourceをどのように設計・実装すれば良いかについて考えていきます。  

## Viewの設計を説明するにあたり
従来のiOSアプリの設計におけるViewControllerとViewの関係があまりに近しいため、前回のViewController設計の記事でViewの設計についても基本的なことは説明してしまいました。    
ただ今回の設計においてViewはViewContorllerと明確に区別され独立した重要な機構です。  
そのため内容は前回記事と重複する箇所もありますが再度1からViewの設計について説明していきます。      
前回の記事を読んだ人はAlertの説明箇所まで読み飛ばしてもらって大丈夫です。    

## Viewの設計
さて、既にお伝えした通り今回の設計ではViewをViewControllerとは明確に切り離しており、実装においてViewController毎の独自RootViewクラスの指定を必須としています。  
ここでは前回の記事の例でも出したHogeRootViewを参考に独自のRootViewクラスをどのように設計していけば良いか説明します。  
なお冒頭の前提でも述べた通り、UIはInterface Builderを利用せずコードを使って生成しています。   


### HogeRootViewの例
さっそくですが、HogeRootViewはHogeViewControllerのRootViewであり画面の構成は以下のようになっています。    
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter5(View%7CAlert)/Images/RootViewの構成.png" alt="RootViewの構成" width=60% > 

実装も以下に記載します。  
```
protocol AppView: UIView {
    func setup()
}

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

最初に今回のケースの概要について簡単に説明をしておきます。  
#### ViewControllerのジェネリクスに自身のRootViewクラスを指定する
まず今回のViewの独立にあたり以下のようなViewControllerを基底クラスとして利用しています。(以下のコードではRootViewに関係のある箇所のみ抽出しています。)  
```
class ViewController<View: AppView>: UIViewController {
    var rootView: View {
        return self.view as! View
    }
    ...
        
    final override func loadView() {
        self.view = View()
        self.rootView.setup()
    }
    
}
```
指定したRootViewクラスのインスタンスにはrootViewプロパティからアクセス可能です。  

今回のHogeRootViewに対応するHogeViewControllerは
```
class HogeViewController: ViewController<HogeRootView> {
   ...
}
```
というように定義しています。  
またこれは後ほど詳しく説明しますが、RootViewクラスのインスタンスはloadView()メソッド内で生成して、自身のviewプロパティに代入しています。  
#### RootViewクラスはAppViewプロトコルに準拠する
次にHogeRootViewのコード冒頭に書かれているAppViewプロトコルについてです。  
このAppViewは各ViewControllerのRootViewであることを明示するためのプロトコルであり、RootViewとなるViewはこのプロトコルに準拠している必要があります。  
そして各RootViewでセットアップ処理を行いたい場合はこのAppViewプロトコルのsetup()メソッドにその処理を実装します。  
今回の例ではsetup()メソッド内でhogeLabelとhogeViewColorChangeButtonにアクセスして、両UIコンポーネントの遅延生成処理を発動させています。  
ちなみに先の基底ViewControllerクラスを見たらわかる通り、このsetup()メソッドはViewControllerのloadView()メソッド内で呼ばれます。  

#### HogeRootViewのsetColorMode(lightMode: Bool)メソッド 
この画面ではhogeViewColorChangeButtonにタップすることで画面全体の色を変えられる仕様になっており、setColorMode(lightMode: Bool)はその色の変更を実行するメソッドとなります。  
今回のようにViewをViewControllerから切り離した設計では、Viewに関する処理のメソッドは全てViewクラスに定義・実装していくことになります。  

### View設計の基本
HogeRootViewの例をみて大体わかったと思いますが、ViewControllerから切り離されたRootViewではViewの宣言、生成処理(Interface Builderを利用していない場合)、View全体の初期化処理、Viewの操作処理、とViewに関わるあらゆる定義と実装がなされることになります。  
そしてこれらをRootViewに定義・実装する際には、特に特別な工夫は必要ありません。    
RootViewからの入力イベントは全てViewController側で管理するので、RootView自体は各Viewコンポーネントの出力に特化しており責務もデータフローも単純です。  
そのため構造としての複雑性は非常に低く、責務をただ順々に書き連ねても開発で問題が起こることはないと思います。  
あえて何かいうならば、一般的な感覚でいうと責務を書き連ねる順番は「宣言(生成処理)->View全体の初期化->Viewの操作メソッド」が妥当であるということぐらいでしょうか。  

### View設計における注意点
上記の通りViewの設計については基本的な責務さえ理解しているだけで十分です。  
ただそれでも2点ほど留意しておきたい点があるのでここではそれらについて説明します。

#### RootViewでは初期化時にデータを渡さない
既に示した基底ViewControllerを見てもわかる通り、ViewController内でRootViewは一切のパラメータなしで初期化されています。  
基底ViewControllerクラスがこのように設計されていることにより全てのRootViewで初期化時のデータの受け渡しができなくなるわけですが、それが原因で何か問題が起きたりしないでしょうか。　　
  
結論を先に言うと、私は大丈夫だと思っています。  
先ほども述べた通りRootView責務は各Viewコンポーネントの出力に特化していて、通常その出力はViewController側のイベントをトリガーに発生します。  
そのためViewに外部からのデータが必要な場合にはViewControllerの出力イベントに合わせて、ViewControllerから渡せば十分要件を満たすことが可能です。  
Viewの初期状態に必要なデータもViewの初期化時ではなくViewControllerのviewDidLoad()メソッドを介したタイミングで行えば問題ないと思います。  

もし各RootViewの初期化時のデータ受け渡しを許すのならば、その初期化のパターンに様々なケースが想定されるため汎用性のあるViewControllerとViewを切り離した設計を考えるのは非常に難しくなります。  
なので今回のRootViewの設計では初期化時のデータ受け渡しを不可で固定することで、単一の基底ViewControllerクラスのみによってあらゆるViewControllerとViewの切り離しを可能にしています。  

#### init(frame:CGRect)の実装が必須
基底ViewControllerではプログラム上RootViewを
```
self.view = View() 
```
とパラメーター無しで生成していますが、実際にはこのRootViewの生成処理内部では`init(frame:CGRect)`を利用しているようです。  
そのため各RootViewクラスでは`init(frame:CGRect)`を実装する必要があります。  

## Alertの設計
ここからはAlertの設計について説明していきます。  
まずデフォルトのAlert開発時の問題点を示しながら、要改善箇所を確認します。  

### デフォルトAlertの問題点
アプリ設計の観点からデフォルトのAlertの問題点は3点があると思います。  
1. 表示するために必要な設定箇所が多くプログラムが命令的
2. アプリの機能との連携が見えづらい
3. データフローが複雑になる

以下では簡単にそれぞれの説明をします。  
#### 1.表示するために必要な設定が多くプログラムが命令的
この1点目に関しては一般的に認識されているため、想像するのは難しくないでしょう。  
以下では簡単なデフォルトAlertの実装例を紹介していますが、通常Alertの実装では
- Alert自身のタイトル、メッセージ、スタイルの設定
- 各アクションのタイトル、スタイル、タップ時の処理の設定
- Alertの表示  

と表示のために様々な設定とメソッドの呼び出しを行う必要があります。  
そのためAlertを一つ表示するだけでもそれなりの量かつ命令的な記述となり、ViewControllerの肥大化および可読性の低下につながってしまう恐れがあります。  

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

#### 2.アプリ機能との連携が見えづらい
1で見たようにデフォルトのAlertでは自身の表示に必要な文字列、スタイル、タップ時の処理を一つ一つ設定していきますが、このような情報は詳細すぎて実装者以外の開発者がみてもそのAlertが何をしているのかイマイチ理解できないと思います。    

個々のAlertは必ず特定のモジュールや状況と対応しています。  
例えば「写真アイテム取得失敗の対応」を促すためのAlert、「決済前の意思確認」のためのAlert、「ログインする必要があることを知らせる」ためのAlert等です。  
当たり前ですが、プログラムを読む上ではこのような背景状況を把握できた上で、詳細な情報を読んでいった方が理解しやすいです。  

しかしデフォルトのAlertの仕様はこうした個々のアプリ機能と完全に切り離されており、Alertの実装ではそこで表示する文言等、詳細な情報だけを確認できるようになっています。      
もちろんUIフレームワークの視点からいえばこのようにUIであるAlertとアプリの機能面が切り離されているのはおかしなことではありません。  
ただアプリ設計の観点からみるとAlertとアプリ機能とのつながりが可視化されるように、Alertを再構築する必要があると思います。  

#### 3.データフローが複雑になる
「設計を理解するためのクリーンアーキテクチャ」編でデータフローのわかりやすさはそのままプログラムのわかりやすさに直結すると説明しました。    
そのため「ViewController」編でもViewController内部で「入力データフロー」と「出力データフロー」を区別しています。    
しかし、デフォルトAlertではその表示(出力)箇所でタップ時の処理(入力)を定義するので、出力と入力のデータフローを切り離せません。        
これは言ってみれば、異なるベクトルを持つデータフローが入れ子構造になっている状態です。    
通常であればViewからの入力は`func addTarget(_ target: Any?, action: Selector, for controlEvents: UIControl.Event)`等を利用してViewの出力とは切り離された形で行われますが、Alertの場合は出力とともに入力を定義するので実質的に出力処理が入力処理を内包しています。  
デフォルトAlertが持つこのような性質はViewControllerのデータフローを複雑にして、プログラムの流れを理解しづらくさせています。      

ちなみにここで指摘されている「表示(出力)箇所でタップ時の処理(入力)を定義する」性質はSwiftUIのAlertでも同様ですが、SwiftUIではこれによって特に問題は起きません。　
この違いにはSwift UIのViewとUIKitのViewControllerのアプリ上での立ち位置が関係しているのですが、それについては後ほど補論にて説明します。  

### デフォルトAlertの問題を解決していく
ここからは各問題点をどのように解決していくのか一つ一つ説明していきます。  

#### 「1.表示するために必要な設定が多くプログラムが命令的」問題の解決
まずAlertの表示に際してプログラムが煩雑になってしまう問題は、以下のように種々のデータを一括して扱うオブジェクト(この例では"AlertStrategy"型と命名)を定義して解決します。  
```
struct AlertStrategy<Action: AlertActionType> {
    var title: String
    var message: String
    var actions: [Action]
    var style: AlertStyle
}

enum AlertStyle {
    case actionSheet
    case alert
}

extension AlertStrategy: Error {}
```
このようにラッパーオブジェクトを定義することで複数のデータを一括で管理するというのはAlertの設計においてよく取られるアプローチなので目新しくはないと思いますが、やはりそれだけに非常に便利な手法です。  

今回のAlertの設計では以下のAlertClientTypeに準拠したオブジェクトにこのAlertStrategyを渡すことでAlertを表示できる仕様になっています。(AlertClientTypeは後ほどまた詳しく説明します。)    
```
protocol AlertClientType: NSObject {
    associatedtype Action: AlertActionType
    init(viewController: UIViewController)
    
    func show(strategy: AlertStrategy<Action>,
              animated: Bool,
              completion: (() -> Void)?)
   ...
```

しかし通常のラッパーオブジェクトと異なる点としてはAlertStrategyでジェネリクスとして宣言している`<Action: AlertActionType>`型によってアプリ機能に
