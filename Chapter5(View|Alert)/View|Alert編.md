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
最初に既存のAlert開発を踏まえながら、今回のAlert設計に当たって解決すべき問題点を確認します。  

### 既存のAlertの問題点
私はアプリ設計の観点から既存のAlertには以下3点の問題点があると考えています。  
1. 表示するために必要な設定データが多くプログラムが命令的
2. アプリ機能との連携が見えづらい
3. データフローが複雑になる

なんのことを言っているか大体検討がついている点もあるかと思いますが、以下ではそれぞれの詳細について説明していきます。    
#### 1.表示するために必要な設定が多くプログラムが命令的
これは一般的に認識されている問題なので、想像するのは難しくないと思います。    
デフォルトAlertの実装では、**Alert自身のタイトル、メッセージ、スタイルの設定**,**各アクションボタンのタイトル、スタイル、そしてタップ時の処理の設定**と、その表示のために様々なデータの設定とメソッドの呼び出しを行う必要があります。  
なのでAlertを一つ表示するだけでもそれなりの量、かつその記述スタイルは命令的であるため、ViewControllerの肥大化および可読性の低下の一因となってしまっています。  

以下では簡単なデフォルトAlertの実装例を示しておきます。  
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

    viewController.present(alert, animated: true, completion: nil)
```

#### 2.アプリ機能との連携が見えづらい
1で見たようにデフォルトAlertではその表示のため色々とデータを設定していきますが、それら設定データの情報はとても細かく、開発者はそのAlertが何をしているのか理解するためにコードを丁寧に読んでいく必要があります。  

Alertの読解がこのように面倒な原因として、プログラム上にそのAlertのコンテクストが明示されていないことがあります。   
個々のAlertは必ずアプリ内の特定のモジュールと対応しています。  
例えば「写真アイテム取得失敗の対応を促す」ためのAlert、「決済前の意思確認」のためのAlert、「ログインする必要があることを知らせる」ためのAlert等です。  
プログラムを読む上でこうしたコンテクストを最初に確認できるようになっているならば、その可読性は高くなります。  

もちろんAlertがUIKitフレームワークの一部であることを考えると、デフォルトAlertがアプリの機能面と切り離されていることはしょうがないことです。    
しかしアプリケーションの設計という観点考えると、Alertとアプリ機能とのつながりが可視化されるようにAlertを再構築する必要があると思います。   

#### 3.データフローが複雑になる
「設計を理解するためのクリーンアーキテクチャ」編でデータフローのわかりやすさはそのままプログラムのわかりやすさに直結すると説明しました。    
そのため「ViewController」編でもViewControllerの「入力データフロー」と「出力データフロー」が区別されるように設計していました。      
しかし、デフォルトAlertではその表示(出力)箇所でタップ時の処理(入力)も定義するので、出力と入力のデータフローを切り離すことができません。         
これは言ってみれば、異なるベクトルを持つデータフローが入れ子構造になっている状態です。    
通常であればViewからの入力は`func addTarget(_ target: Any?, action: Selector, for controlEvents: UIControl.Event)`等を利用してViewの出力とは切り離された形で行われますが、Alertの場合は出力とともに入力も定義するので実質的に入力処理が出力処理に内包されている構造となっています。  
デフォルトAlertが持つこのようなデータフローの複雑性は、プログラムの流れを追うことを難しくしています。   

ちなみにここで指摘されている「表示(出力)箇所でタップ時の処理(入力)を定義する」性質はSwiftUIのAlertも同様に持っていますが、そちらでは特に問題は起きません。　
この違いにはSwift UIのViewとUIKitのViewControllerのアプリ上での立ち位置が関係しているのですが、それについては後ほど補論にて説明します。  

### デフォルトAlertの問題を解決していく
さて、それでは上記の問題に対して解決策を提案していこうと思いますが、ただその前に今回のAlert設計ではViewControllerの外部に委譲されているという大前提があるため、まずその外部化をどうやって実現方法から説明していきます。  

#### ViewControllerの代理でAlertの表示を行うAlertClient
本記事ではViewControllerの代理としてAlertの表示を行うコンポーネントをAlertClientとします。  
AlertClientの基本的な構造はRouterと同じで、AlertClientにViewControllerを渡してそちら側でAlert表示の実装を行います。  
これはAlertの表示が技術的にはViewControllerの`present(_:animated:completion:)`メソッド、すなわち遷移処理によって実行されていることを考えればわかると思います。    

しかし、技術的には同類でもやはりサービスの観点からいうと「遷移」と「アラートの表示」は異なっているべきです。    
そのためAlertの表示を行うコンポーネントをAlertClientとしてRouterと区別しているわけですが、本記事が提案する設計ではAlertClientはAlertClientTypeプロトコルに準拠する形式でAlert表示処理を実装します。      
以下はAlertClientTypeプロトコルのAlertの表示に関する定義です。(AlertClientTypeプロトコルの定義全体は後ほど示します。)　　

```
protocol AlertClientType: AnyObject {
    associatedtype Action: AlertActionType
    init(viewController: UIViewController)
    
    func show(strategy: AlertStrategy<Action>,
              animated: Bool,
              completion: (() -> Void)?)
   ...
```
AlertClientはこの`show(strategy: AlertStrategy<Action>, animated: Bool, completion: (() -> Void)?)`メソッドの呼び出しによってAlertを表示しますが、このメソッドはViewControllerでAlertを表示する際のpresentメソッドと非常によく似ているためiOS開発者は特に違和感なく利用することができると思います。  
```
func present(_ viewControllerToPresent: UIViewController, 
             animated flag: Bool, 
             completion: (() -> Void)? = nil)
```
ただAlertClientTypeとそのshowメソッドで宣言、定義されている**AlertStrategy**、**Action: AlertActionType**は本設計で独自に定義している型です。  
これらの型については先に示したAlertの問題群の解決と不可分であるため、後ほどそれらとともに説明していきます。  
とりあえずここでは**AlertStrategy**、**Action: AlertActionType**という独自型を利用しながら、AlertClientTypeのshowメソッドでAlertを表示していることだけ理解してもらえれば十分です。  

#### 「1.表示するために必要な設定が多くプログラムが命令的」問題の解決
まずAlertの表示プログラムが煩雑になってしまう問題は、先で紹介したAlertStrategy型によって解決します。  
以下その定義です。  
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
AlertStategyは簡単に言えば、Alertに関する煩雑なデータ群を一括で操作するラッパーオブジェクトと言えます。
これはAlertの設計においてよく取られるアプローチであるため目新しくはないと思いますが、やはりそれだけに非常に便利な手法であり、このAlertStrategyを利用することで煩雑なデータ操作は全てAlertの内部で実装されるため、Alertを表示する側で意識することはなくなります。  

しかしAlertStategyが通常のAlertデータのラッパーオブジェクトと異なる点としては、ジェネリクスとして`Action: AlertActionType`を利用していることでしょうか。  
この`Action: AlertActionType`はAlertのモジュール化、および入出力データフローの切り離しを可能にしているのですが、それについては後ほどまた説明します。  

ちなみに`enum AlertStyle`はUIKitの`UIAlertController.Style`型と同義なのですが、わざわざ自作で定義し直しているのはAlertStrategy型はその性質上ViewModel等でも利用するためUIKitの型に依存した設計にしたくなかったからです。  
AlertのUI側では以下のように`UIAlertController.Style`を拡張して`enum AlertStyle`から`UIAlertController.Style`型を生成できるように定義しています。  
```
extension UIAlertController.Style {
    init(style: AlertStyle) {
        switch style {
        case .alert:
            self = .alert
        case .actionSheet:
            self = .actionSheet
        }
    }
}
```

また`extension AlertStrategy: Error {}`とAlertStrategy型をErrorプロトコルに準拠させているのも、その性質上Result型のFailure型として扱われる場合があるためです。  

#### 「2.アプリ機能との連携が見えづらい」問題の解決
既に簡単に触れましたが、今回の設計ではAlertActionTypeプロトコルを使ってアプリ固有の機能に合わせたAlertのモジュール化を実現しています。  
AlertActionTypeとそれに関連する定義は以下の通りです。  
```
protocol AlertActionType: Equatable {
    var title: String { get }
    var style: AlertActionStyle { get }
}


enum AlertActionStyle {
    case `default`
    case cancel
    case destructive
}
```
AlertActionTypeの`var title: String`はAlertボタンのタイトルなる文字列、`var style: AlertActionStyle `はAlertボタンのスタイルを指し、これは`UIAlertAction.Style`と同じ役割を持っています。  

アプリ内でAlertを表示したい場合には、そのアラート表示に関わる機能や状況が明示されるようなAlertActionTypeの実体型を定義します。  
例えば今回のサンプルプロジェクトでは写真アイテムの取得失敗時にアラートを表示するのですが、その実装のためにAlertActionTypeに準拠したFetchPhotoErrorAction型を定義・実装しています。  
```
enum FetchPhotoErrorAction: String, AlertActionType {
    case retry = "Retry"
    case cancel = "Cancel"
    case setting = "Setting"
    case signIn = "Sign In"
    case none = "Confirm"
    
    var title: String {
        return self.rawValue
    }
    
    var style: AlertActionStyle {
        return self == .cancel ? .cancel : .default
    }
}
```

このようにAlertActionTypeを使うと、Alertの機能毎に開発をしていくことになるので、アプリの仕様との親和性はとても高くなります。  
また実際にAlertを利用するプログラムでは`AlertClient<FetchPhotoErrorAction>`や`AlertStrategy<FetchPhotoErrorAction>`等、AlertActionTypeの実態型を明示したAlertコンポーネントを宣言する必要があるため、開発者は利用されているAlertコンポーネントの型名を確認すればそのコンテクストを把握できます。  

ちなみに基本的にAlertActionTypeの実体型は上記のようにEnumで定義し、ユーザーがAlertに対して取りうる手段をcaseとして宣言していくのが良いと思います。    
また`UIAlertAction.Style`と同義であるにも関わらず、わざわざAlertActionStyle型を独自定義しているのは先程のAlertStyleと同じ理由です。  
AlertActionStyle型もUI側でUIAlertAction.Styleへ変換できるように拡張実装を行なっています。  
```
extension UIAlertAction.Style {
    init(style: AlertActionStyle) {
        switch style {
        case .default:
            self = .default
        case .cancel:
            self = .cancel
        case .destructive:
            self = .destructive
        }
    }
}
```

> 補足:  
> 例で示した`AlertClient<FetchPhotoErrorAction>`を見てもわかるとおり、AlertClientコンポーネントがモジュール化をジェネリクスで実現しているということは、ViewControllerで複数のAlertモジュールを利用したい場合にはその数だけAlertClientコンポーネントを宣言する必要があるということです。  
> これは一見すると冗長に思えますが、これで良いのでしょうか。      
>   
> 結論だけいうと私はこれで良いと思います。    
> プログラムには冗長さがあえて必要な場合がありますが、このケースがまさにそれです。  
> AlertClientのジェネリクスはそのAlertのコンテクストを伝える役割を担っており、もしAlertClientを集約化するためにジェネリクスを消去したならばそのコンテクストも失われてしまいます。  
> インスタンスの型が`AlertClient<FetchPhotoErrorAction>`であれば、それだけで「写真取得失敗時の」Alert表示コンポーネントであることが伝わりますが、`AlertClient`のみだとAlertを表示すること以外何もわからず開発者はその"概要"を把握するためにプログラムの”詳細”を読む必要があります。  
> アプリケーション開発では機械上で動くプロダクトを作っているわけですが、その開発においてプログラムを読むのは結局"人"です。  
> そのため人が読むことを考えてあえて冗長でも具体性を持ったプログラムを実装する判断はスケールしやすい設計を考える上でとても重要です。  
> 
> またこの後説明しますが、今回のケースではAlertClientコンポーネントのジェネリクスは入出力処理の切り分けでもとても重要な役割を担っています。  
  
ちなみにAlertStrategyをAction型の制約付きで拡張することで、Action型毎の初期化処理を定義することが可能です。  
```
// RepositoryDomainError型はRepository関係のError型
extension AlertStrategy where Action == FetchPhotoErrorAction {
    
    init(error: RepositoryDomainError) {
        // ActionがFetchPhotoErrorAction型である時の初期化処理
    }
}
```
  
#### 「3.データフローが複雑になる」問題の解決
それでは最後にAlertの入出力データフローを切り離す方法を説明しますが、今回の設計においてその役割を担っているのはAlertClientTypeです。  
AlertClientTypeは既に説明したようにAlertを表示する役割を担っていますが、それに加えて入出力データフローを切り離しも行なっています。  
以下では先に示したshowメソッドも含め、AlertClientTypeプロトコル全体とその関連オブジェクトの定義を示します。    
```
public struct RegistryKey: Hashable {
    private let _uuid: UUID
    
    init() {
        self._uuid = UUID()
    }
    
    public static func ==(lhs: RegistryKey, rhs: RegistryKey) -> Bool {
        return lhs._uuid == rhs._uuid
    }
}

protocol AlertClientType: AnyObject {
    associatedtype Action: AlertActionType
    init(viewController: UIViewController)
    
    func show(strategy: AlertStrategy<Action>,
              animated: Bool,
              completion: (() -> Void)?)
    
    func register(_ handler: @escaping (Action) -> Void) -> RegistryKey
    
    func register(on action: Action,_ handler: @escaping (Action) -> Void) -> RegistryKey
    
    func register(on actions: [Action],_ handler: @escaping (Action) -> Void) -> RegistryKey
    
    func unregister(key: RegistryKey) -> Void?
}

```
上記のコードを見て大体察しはついていると思いますが、AlertClientはregisterメソッドを呼び出してAlertボタンタップ(入力)時の処理を登録します。  
以前はAlertボタンタップ時の処理は、Alertを表示する際Alertボタンに直接定義していました。  
それを考えると今回の設計ではAlertを表示する出力処理(showメソッド)とAlertのボタンが押された際の入力処理の登録(registerメソッド)が全く異なるタイミングで呼び出し可能で、両者が切り離されたのがわかると思います。    
registerメソッドの呼び出し時にはタップ時の処理をクロージャとして渡しますが、そのクロージャ内では引数として受け取る`Action: AlertActionType`から何のAlertボタンが押されたか判別します。  
例えばAlertClientが先程例に出した`FetchPhotoErrorAction`をAction型として指定してる場合には以下のようにクロージャを渡すことでAlertボタンタップ時の処理を登録しています。  

```

　　　let alertClient: AlertClient<FetchPhotoErrorAction> = .init(viewController: viewContrller)
   
　　　let key:RegistryKey = alertClient
    　                       .register { action in
                 　　　            switch action {
                 　　　            case .retry: // retryボタンがタップされた時の処理
                 　　　            case .cancel: // cancelボタンがタップされた時の処理
                 　　　            case .setting: // settingボタンがタップされた時の処理
                 　　　            case .signIn: //signInボタンがタップされた時の処理
                 　　　            case .none: return //「確認」ボタン等、特にタップされても行う処理がない場合
                 　　         }
              　　　}
```
従来のAlert実装よりも自然言語的なコードとなって大分読みやすくなったのではないでしょうか。  
  
そしてここにもAlertClientTypeの`associatedtype Action: AlertActionType`によるモジュール化の効果が現れています。  
上記の例ではAlertClient<FetchPhotoErrorAction>型のジェネリクスによってregisterメソッドのクロージャが受け取るAction型が明確であるため、その登録処理内のswitch文による分岐は非常にシンプル直感的です。  
これはFetchPhotoErrorActionだからというわけではなく、一つのAlertモジュールが一般的にユーザーに与える選択肢の数(良いかればAlertActionTypeに準拠したEnumのcaseの数)を考えると、ジェネリクスを使ってAlertClientが対応するモジュールを一つに限定する限り、クロージャ内の分岐が煩雑になる可能性はとても低いと思います。  
もしここで`Action: AlertActionType`を型のジェネリクスとして利用していなければ、クロージャが受け取るAlertActionTypeの実体を想定することは非常に難しくなり、登録処理の内容は複雑になってしまうはずです。  
このようにAlertClientTypeの`associatedtype Action: AlertActionType`によるモジュール化はAlertに関するコードを読みやすくさせるだけではなく、そのコードの記述にも役にたっています。  

ちなみに
 - `func register(on action: Action,_ handler: @escaping (Action) -> Void) -> RegistryKey`  
 - `func register(on actions: [Action],_ handler: @escaping (Action) -> Void) -> RegistryKey`  
    
は特定のActionが発生した場合のみ呼び出したい処理を登録します。  
     
そしてRegistryKeyという独自型を利用していますが、これはAlertに登録した処理を管理するのに利用するキーの役割であり、もし登録した処理が呼びされるのをやめたい場合は、該当の処理登録時に返り値として受け取ったRegistryKeyを`func unregister(key: RegistryKey) -> Void?`に渡すことで登録を解除します。  

### AlertClientTypeの実体型
デフォルトAlertの問題を独自に定義したAlertClientType、そしてAlertStrategy、AlertActionTypeを使いながら解決していきました。  
しかし論理的には解決策を提示したものの、肝心のAlertを表示するAlertClientTypeの実装については触れていないのでここではそれについて説明したいと思います。  
AlertClientTypeの実体型はそのテスト等、その実行環境によっていくつか定義する必要があるかもしれません。  
ただ本番環境に限って言えばモジュールの多様性はジェネリクスによって実現しているためAlertClientの実体型は一つ定義すれば十分なはずであり、またAlertClientTypeの要件を考えてもその実装内容も大きく変わることはないと思います。    
以下では私が実装したAlertClientクラスを示します。  

```
final class AlertClient<Action: AlertActionType>: AlertClientType {
    private weak var vc: UIViewController!
    private var handlers: [RegistryKey: (Action) -> Void] = [:]
    
    required init(viewController: UIViewController) {
        self.vc = viewController
    }
    
    
    func show(strategy: AlertStrategy<Action>,
              animated: Bool,
              completion: (() -> Void)?) {

        let alert: UIAlertController = UIAlertController(title: strategy.title,
                                                         message: strategy.message,
                                                         preferredStyle: UIAlertController.Style(style: strategy.style))

        for action in strategy.actions {
            alert.addAction(UIAlertAction(title: action.title,
                                          style: UIAlertAction.Style(style: action.style),
                                          handler: {(alertAction: UIAlertAction) -> Void in
        
                                            self.handlers.values.forEach{
                                                $0(action)
                                            }
                                          }))
        }
        self.vc.present(alert,
                          animated: animated,
                          completion: completion)
        
    }
    
    func register(handler: @escaping (Action) -> Void) -> RegistryKey {
        let key: RegistryKey = .init()
        self.handlers[key] = handler
    }
    
    func register(on action: Action, handler: @escaping (Action) -> Void) -> RegistryKey {
        let key: RegistryKey = .init()
        self.handlers[key] = { _action in
            if action == _action {
                handler(action)
            }
        }
        return key
    }
    
    func register(on actions: [Action], handler: @escaping (Action) -> Void) -> RegistryKey {
        let key: RegistryKey = .init()
        self.handlers[key] = { action in
            if actions.contains(action) {
                handler(action)
            }
        }
        return key
    }
    
    
    func unregister(key: RegistryKey) -> Void? {
        if let _ = self.handlers.removeValue(forKey: key) {
            return ()
        } else {
            return nil
        }
    }
}


```
#### AlertClientのインスタンス変数
最初にインスタンス変数の説明からすると、宣言されているのは`private weak var vc: UIViewController!`と`private var handlers: [RegistryKey: (Action) -> Void] = [:]`の2つだけです。  
ViewControllerがAlertClientを保持するおり循環参照を避けるため、AlertClient側ではViewControllerを弱参照(weak)しています。  
handlers変数はRegistryKeyをキーとして登録されたタップ時の処理を保持するディクショナリー型です。  
    
#### AlertClientのメソッド  
次にメソッドの説明をしようと思いますが、showメソッドの前にregister/unregisterメソッドから見ていきます。  
まず各registerメソッドは引数の違いによって登録処理にフィルター機能を追加するといった違いはありますが、基本的には
1. RegistryKeyインスタンスを生成
2. 生成したRegistryKeyインスタンスをキーとして、引数で渡されたクロージャをhandlers変数に格納
3. ViewController側で登録処理を解除できるようにRegistryKeyインスタンスを返り値として渡す  
    
だけです。  
処理登録といっても辞書に格納するだけなので、とてもシンプルな実装だと思います。  
そしてunregisterメソッドはRegistryKeyを受け取り、該当の登録処理がhandlers変数に含まれていたら削除します。  

さて、それでは最後にAlertClientの肝であるshowメソッドについてですが、これも基本的にはAletStrategyとそれが保持しているAlertActionのデータからUIAlertControllerとUIAlertActionを生成、セットアップしていっているだけでとりわけ説明しなければいけないことはありません。  
しかし、設計上重要な点はUIAlertAction、つまりAlertボタンタップ時の処理としてhandlers変数が持っている登録処理群を呼び出していることです。  
そしてこの登録処理呼び出しの際、引数には自身のAlertボタンに照応したAlertActionを渡しています。  
このようにAlertボタンタップ時の処理として登録処理を呼び出すようにすることで、データ入出力の切り離し、またタップ時の処理の集約化(各UIAlertAction一つ一つにタップ時の処理定義していくのではなく、Alertモジュール毎に一括したタップ時の処理の定義)が可能になりました。  

### Alert設計のまとめ
デフォルトAlertの問題点を挙げ、それに対応する形の新しいAlert設計を考えていきました。  
後で簡単な例を使って、実際にこの新しいAlertを使ったアプリの実装がどのようなものになるのか見ていきたいと思いますが、一度ここで本記事のAlert設計の内容をまとめます。  
    
今回の設計においてデフォルトAlertに加えた変更は以下4点です。
1. AlertClientTypeとそのshowメソッドによって、ViewControllerが行っていたAlertの表示を代わりに行う
2. AlertStrategyによるAlert関連のデータの一括化によって、Alertの煩雑なデータ設定を内部実装に閉じ込める
3. AlertActionによってモジュール毎のAlert開発を可能にさせ、また各Alertモジュールがユーザーに提示する選択肢を記号化(Enumのcase)する
4. AlertClientTypeのregisterメソッドと前述のAlertActionによって、Alertボタンタップ時の処理の登録(入力)とAlertの表示(出力)を切り離す  
これら4つの変更によってViewControllerにおけるAlertのコードは*簡潔かつ宣言的*に記述できるようになります。  

以下イメージとして先に示した`enum FetchPhotoErrorAction: String, AlertActionType`を使ったViewControllerの実装例を示します。(もう少し本格的な例は本記事の最後に紹介します)    
    

```
struct Photo {
    //写真に関するデータ...
}
protocol ExamplePresenterInputs: AnyObject {
    
    init(outputs: ExamplePresenterOutputs)
    func fetchPhotos()
    func goToSetting()
    func signIn()
}

protocol ExamplePresenterOutputs: AnyObject {
    func showPhotos(photos: [Photo])
    func fetchPhotosfailed(strategy: AlertStrategy<FetchPhotoErrorAction>)
}

class ExampleViewController<Presenter: ExamplePresenterInputs,
                            FetchPhotoErrorAlertClient: AlertClientType>: UIViewController,
                                                                          ExamplePresenterOutputs
                            where FetchPhotoErrorAlertClient.Action == FetchPhotoErrorAction {
    
    
    private var presenter: Presenter!
    private var alertClient: FetchPhotoErrorAlertClient!
    
    init() {
        self.presenter = Presenter.init(outputs: self)
        self.alertClient = FetchPhotoErrorAlertClient.init(viewController: self)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    //入力処理の登録
    override func viewDidLoad() {
        //その他の入力処理
        //...
        
        
        // Alertボタンタップ時の入力処理
        _ = alertClient
            .register { action in
                switch action {
                case .retry: self.presenter.fetchPhotos()
                case .setting: self.presenter.goToSetting()
                case .signIn: self.presenter.signIn()
                case .cancel, .none: return
                }
            }
    }
    
    func showPhotos(photos: [Photo]) {
        //写真の表示
    }
    
    //写真取得失敗時にAlertを表示
    func fetchPhotosfailed(strategy: AlertStrategy<FetchPhotoErrorAction>) {
        self.alertClient
            .show(strategy: strategy,
                  animated: true,
                  completion: nil)
    }
}

```

どうでしょう、デフォルトAlertと比べてその構造が洗練され、直感的に理解しやすくなったのではないでしょうか。  
最初に載せたデフォルトAlertの実装を下に再掲しますが、一見してデフォルトAlertが表示する際に行っていた細々とした処理を変更後の設計ではViewController上で意識することがなくなったのがわかります。  
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

    viewController.present(alert, animated: true, completion: nil)
```
もちろん変更後の設計ではAlertStrategyの生成処理をPresenterで行っており、上記の両者のコードを単純に比較できない面はあります。  
しかしここで重要なのはAlertがそれを利用するコンポーネントに適した設計になったということです。  
PresenterはView関係のデータを操作するコンポーネントなので、そこでAlertStrategyを生成することはなんの問題もありません。  
そしてViewControllerの責務は「画面入出力イベントを管理する」(ViewController編参照)であり、また開発者からすると「その画面がアプリ上どのような役割を持っているか」把握するためにあります。  
これは言い換えれば、ViewControllerからするとAlertのタイトルやメッセージの文言やその詳細の設定は究極的にはどうでもよく、重要なのは「何が原因でアラートを表示するのか(利用状況)」、「そのAlertの対応としてViewControllerはどのように振る舞うのか」にあるということです。  
    
そのようにアプリケーション内のコンポーネントの責務を考えた場合、今回の変更後のAlertはそれを扱う各コンポーネントにおける要件に適宜応えており、実装しやすい設計となっていると思います。    

## CollectionViewのDataSource
最後に本記事で取り上げたコンポーネント設計を簡単なアプリで実践しようと思いますが、その前にCollectionViewのDataSourceについて話します。  
ちなみにここでのDataSourceはUICollectionViewDataSourceに準拠した実体型ではなく、それを内包したラッパーオブジェクトを指しています。  
    
今回の設計ではAlertと同様にCollectionViewのDataSourceもViewControllerから外部化しています。   
ただその外部化の構造はAlertに比べると単純で、基本的にはそのCollectionViewが表示するアイテムを渡したら内容が更新されるようなDataSourceのラッパーオブジェクトを作成するだけです。  
    
### ViewControllerからみたDataSource
それを示すためにここではDataSourceの一部とそれを利用しているViewControllerの例を紹介します。  
ちなみにDataSourceとしてはUICollectionViewDiffableDataSourceを利用します。  
```
// DataSourceのラッパーオブジェクト
class HomeCollectionDataSourceWrapper {
    enum Section {
        case homePhotos
    }
    ...
    typealias DataSource = UICollectionViewDiffableDataSource<Section, Photo>
    
    ...
    
    private let _datasource: DataSource
   
    ...
    
    func update(newItems: [Photo]) {
        let snapshot: NSDiffableDataSourceSnapshot<Section, Photo> = .init()
        snapshot.appendSections([.homePhotos])
        snapshot.appendItems(newItems,
                             toSection: .homePhotos)
        self._datasource.apply(snapshot)
    }
}
    
class HomeViewController: UIViewController {
    private var datasource: HomeCollectionDataSourceWrapper
    ...
    
    func updateItems(_ photos: [Photo]) {
       self.datasource.update(newItems: photos)
    }
}

```
わざわざ実例出すまでもなかったかもしれませんが上記のコードのように、DataSourceの役割はCollectionViewの内容の表示、もしくは更新を行うためであり、基本的にはそのための処理をラップしてViewControllerから宣言的に利用できるようにするだけです。  
もちろん状況によっては、現在の表示内容、状態の取得等を行いたい場合もあると思いますが、それらがDataSourceの構造を変えるようなことはなく、機能が必要な際にはそこに適宜追加していけば問題ありません。  
    
ちなみにDataSourceの外部化をラッパーオブジェクトによって実現している理由は、直にUIKitのUICollectionViewDataSourceプロトコルを準拠・もしくはUICollectionViewDiffableDataSourceを継承したオブジェクトを利用するとそのケースに必要のないAPIまで晒してしまうことになるからです。  
ViewControllerからDataSourceを利用する場合、その要件はケースによって振り幅が大きいと思うのでラッパーオブジェクトとして定義して各ケースに必要なAPIのみを公開する設計が良いと思います。  

### 表示するための前準備も必要
ただ、CollectionViewの内容を表示するだけというのはあくまで外から見た構造で、実際にはその表示をするための準備処理を内部で行う必要があります。  
基本的にそうした準備処理はDataSourceのinit内で実装されることになると思います。  
以下では先程例に挙げたHomeCollectionDataSourceWrapperの準備処理を含めたコードを示します。  
なおこのHomeCollectionDataSourceWrapperは私のサンプルプロジェクトで使っているDataSourceであるため、ViewModelおよびRxSwift、RxCocoaを利用しています。  
できれば本記事の流れに沿ってViewModelをPresenterに書き換えたかったのですが、技術的な理由でやめました。  
その理由も後ほど説明します。  
```
class HomeCollectionDataSourceWrapper<CellViewModel: HomeCollectionCellViewModelPort> {
    enum Section {
        case homePhotos
    }
    
    typealias CellRegistration = UICollectionView.CellRegistration<PhotoViewCell, CellViewModel>
    typealias DataSource = UICollectionViewDiffableDataSource<Section, Photo>
    typealias ViewModelProvider = (_ photoData: Photo,
                                   _ indexPath: IndexPath) -> CellViewModel
    private let _datasource: DataSource
   
    init(collectionView: UICollectionView,
         viewModelProvider: ViewModelProvider) {
        
        let selectedItem = collectionView
                                .rx
                                .itemSelected
                                .share(replay: 1, scope: .forever)
  
        self._datasource = DataSource(collectionView: collectionView) {
            (collectionView: UICollectionView,
             indexPath: IndexPath,
             photo: Photo) -> UICollectionViewCell? in
            let registration: CellRegistration = .init(handler: { cell, indexPath, viewModel in
                
                viewModel.disposeBag.extension.addDisposables(disposables:
                        selectedItem
                                .bind(to: viewModel.inputs.selectedIndexPath)
                )
                
                viewModel.disposeBag.extension.addDisposables(disposables:
                    viewModel
                        .outputs
                        .photoImageData
                        .do(onNext: { [weak cell] _ in
                            cell?.isUserInteractionEnabled = true
                        })
                        .bind(to: cell.rx.imageData)
                )
            })
            return collectionView
                .dequeueConfiguredReusableCell(using: registration,
                                               for: indexPath,
                                                item: viewModelProvider(photo,
                                                                    indexPath)
                                                )
            
      
        }
        
    }
    
    func update(newItems: [Photo]) {
        let snapshot: NSDiffableDataSourceSnapshot<Section, Photo> = .init()
        snapshot.appendSections([.homePhotos])
        snapshot.appendItems(newItems,
                             toSection: .homePhotos)
        self._datasource.apply(snapshot)
    }
}

```

init内でCellの生成処理、またCellに関する入出力イベントの設定を行っています。  
上記の例は比較的単純なケースであって、他にもヘッダーやフッターを表示したり、UICollectionViewDataSourceプロトコルに関して実装したい処理があった場合はinitのパラメーターを追加して適宜処理、設定を行って行きます。    

### ラッパーオブジェクトはCell毎に定義した方が良さそう
DataSourceの設計としては先ほどの例のようにCellの種類毎にラッパーオブジェクトを定義していくか、もしくは汎用性のあるラッパーオブジェクトを作りそれを使い回す方向がありますが、私は前者の方が良いと思います。  
何度も述べている通りDataSourceオブジェクトの構造は単純ですが、その詳細の仕様については対応するCell毎に結構異なりますし、そもそもDataSourceで普遍的に共有化できるコードはあまり多くありません。  
そのためDataSourceを実装する側と利用する側の両方のコストを考えても、個別に定義していった方が楽になると思います。      

### Cellの入出力イベント処理
#### DataSoure内部でのイベント処理
先のHomeCollectionDataSourceWrapperを見てもわかるように、本設計ではDataSource内でCellの入出力イベント処理をしています。    
しかしAlertの際には「ViewControllerは『画面の入出力を管理』して、『画面の機能を把握』するためのコンポーネントであるため、その『データフローの設計は重要』である」と説明していましたが、Cellの入出力イベントの管理はViewControllerから把握できないDataSource内部で行って良いんでしょうか。  

結論を言うとCellの入出力イベントはDataSource内部で処理して問題ないと思います。  
CollectionViewのCellはUIの中でもとても特殊な立ち位置にあるコンポーネントです。  
UIKitでもCellが選択されるそのイベントはCell自身ではなく、CollectionView側で検知するような設計になっています。  
そのためアプリ側でもCellは画面本体の機能とは切り離し、そのイベント処理はCell自身の視覚的な操作に関するものに限定した設計であるべきだと思います。  
そのようにCellを画面本体の機能と切り離した設計では、Cellのイベント処理をDataSource内部で行っても特に問題が起こることはありません。    

#### UICellConfigurationStateを積極的に利用する
またCellに関する視覚的な操作の多くはCollectionViewのイベントに起因していると思いますが、こうしたケースではCollectionViewからイベントを受けとるのではなくUICellConfigurationStateを利用して自身で状態の変化を検知して処理するようにしましょう。  
UICellConfigurationStateはiOS14で加わったAPIであり、Cellはこれによって自身の状態が変わった際の処理を実装できます。  
このUICellConfigurationStateを使えばCellにおけるViewModelやPresenterを介したイベント処理はほとんど必要なくなるはずです。  

#### 外部を介したイベント処理が必要な場合の問題
それでもアプリのiOSバージョンやプロダクトの仕様によっては、CellでもViewModel/Presenterを介したイベント処理が必要になることもあると思います。      
先に示したHomeCollectionDataSourceWrapperでも通信処理でURLから画像を取得、また取得失敗した場合にはCellをタップすることで再取得を試みるという仕様のため、ViewModelを介したイベント処理がどうしても必要でした。  

ただCellのイベント処理に関する実装には注意しなければならない点があります。  
Cellでイベントを処理したい場合のほとんどがCollectionView関連だと思いますが、通常、CollectionViewの選択イベントは単一のデリゲートオブジェクトに実装されるため、Cellに関するイベント処理と他のイベント処理とを切り離すのが難しいのです。    
先ほどのHomeCollectionDataSourceWrapper内でもCollectionViewの選択イベントを処理していますが、これはRxCocoaの機能を利用することによって初めて実現可能となっています。  
本記事ではCellのイベント処理を他と区別してDataSourceの内部で行うことを提唱していました。  
しかし通常のCollectionViewのデリゲートでその実現が難しいのであれば、その方法について考えなければいけません。  
    
#### 「外部を介したイベント処理が必要な場合の問題」の対処
ここで強調しておきたいのが、私はこのような問題があったとしても「Cellに関するイベントと他のイベントとの切り離し」は第一に優先すべき事柄だと考えています。  
もしここで上記の問題に対して愚直にリアクションして、Cellとその他のイベント処理の切り離しを諦め混在させてしまうと、ViewControllerの入出力のデータフローを切り離す構造まで破壊してしまいます。  
UIKitの中でも特殊な立ち位置にいるCellのためにアプリ開発の要であるViewControllerの統一的な設計を破壊してしまうのならば、それはまさに「木を見て森を見ず」です。  
そのためここでは多少不規則な形になろうともUIの中で特殊なCellとそのイベント処理をDataSourceの内部に閉じ込めて設計全体への影響を防ぐ方針を取るべきだと思います。    
    
その上でRxCocoaを除いたこの問題への対処としてはCellで自身の状態変更を検知する方法があります。    
具体的には以下のようにCellでイベントに関わる状態変数をオーバーライドして、変更検知するように実装します。  
```
override var isSelected: Bool {
   didSet { 
    if oldValue != self.isSelected {
        // ViewModel,Presenterにイベント通知処理
    }
   }
}
```
このように自身の状態変更をイベントのトリガーとすれば、CollectioViewのデリゲート側でCellのイベント処理を行う必要はなくなります。  

#### Cellのイベント処理に関する最適な設計は難しい
しかしこの対処法も問題がないわけではありません。  
通常のインタラクションを端にしたイベント処理と異なりこの方法では状態変更をトリガーとしておりコードの意図が分かりづらいですし、もしCollectionView以外のイベントをCell側で処理する場合には通常のインタラクションと状態変更によるイベント処理が混在し複雑になってしまいます。   

このようにCellに関する設計は一つの問題を解決しようとすると、別の問題が浮かび上がってきて中々納得のいく形を見出せません。  
ただそれでも再三述べている通り、結局はUIKitでCellが特殊であるようにアプリ側でもCellを扱うDataSourceを例外的なコンポーネントとしてその内部で変則的な実装を許容する代わりに、その影響が設計全体に
でないようにするしかないと思います。  
    
### DataSource設計のまとめ
以上のDataSourceは終わりですが、細かい説明が多くなってしまったのでAlert同様以下に要約を載せます。  
- DataSourceの役割はCellの表示であり、DataSourceではCell表示メソッドとそのための準備処理を実装する
- UIにおいてCellの存在は特殊であるため、多少変則的になろうともCellに関する実装はDataSource内部に閉じ込め設計全体への影響がでないようにする

## AlertとDataSourceのサンプルアプリ
では最後に本記事で説明したView、Alert、そしてDataSourceの設計に基づいたサンプルアプリを紹介します。  
アプリの内容は以下に示します。  
ただ動物の写真を切り替えているだけです。  
切り替え時にわざわざアラートで確認を取っている等アプリの仕様は不可解で、PresenterやDomainデータ型の設計もよくない箇所がありますが、とりあえずここではプログラム上のViewController-AlertClient/DataSourceの関係にのみ注目してみてください。  
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter5(View%7CAlert)/Images/sample-app.gif" alt="サンプルアプリ" width=20% >  

### アプリの概要
まず簡単にアプリの概要を説明します。  
内容は先ほどの画像を見ればわかる通り、犬or猫の写真がCollectionViewで表示され下部のボタンを起点に表示動物を切り替えているだけです。  
下部のボタンを押した際には一度確認のAlertが表示され、そこで「See cats(dogs)」ボタンを押すと写真の動物が切り替わります。  
また上部のラベルの文言には現在表示中の動物名、下部のボタンの文言は切り替える動物名(犬が表示されているなら「See cats」、猫なら「See dogs」)が表示されます。  

### アプリの各コンポーネントの紹介
それではView、Alert、DataSource、Presenterとアプリに登場する各コンポーネントを紹介して、最後にそれらがViewControllerでどのように利用されるのかみていきます。
#### View
まず最初にViewについて説明します。  
本記事で[既に述べたとおり](#RootViewクラスはAppViewプロトコルに準拠する)、本設計でViewControllerのRootViewとして定義されるViewはAppViewプロトコルに準拠した上でViewに関するあらゆる定義や処理が実装されます。  
具体的に本アプリでは「上部のLabel」、「CollectionView」、「下部のButton」の各Viewの定義、セットアップ処理、そして表示動物変更時のLabelとButtonの文言を変更する操作が実装されます。  
コードは以下の通りです。  
```
class HogeRootView: UIView, AppView {


    private(set) lazy var label: UILabel = {
        let label = UILabel(frame: .zero)
        label.translatesAutoresizingMaskIntoConstraints = false
        
        self.addSubview(label)
        NSLayoutConstraint.activate([
            label.topAnchor
                .constraint(equalTo: self.safeAreaLayoutGuide.topAnchor, constant: 20),
            label.centerXAnchor
                .constraint(equalTo: self.safeAreaLayoutGuide.centerXAnchor),
            label.heightAnchor.constraint(equalToConstant: 50),
            label.widthAnchor.constraint(equalTo: self.safeAreaLayoutGuide.widthAnchor, multiplier: 0.4)
        ])
        label.textAlignment = .center
        
        return label
    }()
    
    private(set) lazy var collectionView: UICollectionView = {
       

        let layout: UICollectionViewCompositionalLayout = {
            let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(0.333),
                                                  heightDimension: .fractionalHeight(1.0))
            let item = NSCollectionLayoutItem(layoutSize: itemSize)

            item.contentInsets = NSDirectionalEdgeInsets(top: 2,
                                                         leading: 2,
                                                         bottom: 2,
                                                         trailing: 2)
            let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                                   heightDimension: .fractionalWidth(0.333))
            let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize,
                                                           subitems: [item])
            
            let section = NSCollectionLayoutSection(group: group)
            return .init(section:section)
        }()
        
        let collectionView = UICollectionView(frame: .zero,
                                              collectionViewLayout: layout)
        collectionView.translatesAutoresizingMaskIntoConstraints = false
        
        self.addSubview(collectionView)
        NSLayoutConstraint.activate([
            collectionView.topAnchor
                .constraint(equalTo: self.label.bottomAnchor, constant: 20),
            collectionView.centerXAnchor
                .constraint(equalTo: self.safeAreaLayoutGuide.centerXAnchor),
            collectionView.heightAnchor.constraint(equalToConstant: 390),
            collectionView.widthAnchor.constraint(equalTo: self.safeAreaLayoutGuide.widthAnchor)
        ])
        
        return collectionView
    }()
    
    private(set) lazy var button: UIButton = {
        let button = UIButton(frame: .zero)
        button.translatesAutoresizingMaskIntoConstraints = false
        
        self.addSubview(button)
        NSLayoutConstraint.activate([
            button.topAnchor
                .constraint(equalTo: collectionView.bottomAnchor, constant: 50),
            button.centerXAnchor
                .constraint(equalTo: self.safeAreaLayoutGuide.centerXAnchor),
            button.heightAnchor.constraint(equalToConstant: 25),
            button.widthAnchor.constraint(equalTo: self.safeAreaLayoutGuide.widthAnchor, multiplier: 0.25)
        ])
        button.setTitleColor(.black, for: .normal)
        
        return button
    }()

    func setup() {
        self.backgroundColor = .white
        _ = label
        _ = collectionView
        _ = button
    }
    
    func update(animalType type: Animal) {
        self.label.text = type.rawValue
        button.setTitle("See \(type.nextAnimal)", for: .normal)
    }
}
```
`func setup()`で自身の背景色の設定と各Viewの生成を行い、また`func update(animalType type: Animal)`で表示動物が切り替わった際のLabelとButtonの文言の変更を行なっています。  
    
#### Alert
Alertに関しては、AlertClientType、AlertClientの実体型、AlertStrategyは本記事で紹介したものをそのまま流用するためアプリ側で実装する必要があるのはモジュールに応じたAlertActionTypeの実体型の実装のみになります。(しかしコードの重複を避けるためこの後説明するモジュール毎のAlertStarategyの初期化処理も拡張実装をすることも勧めます。)  
本アプリでは動物の切り替えの際に確認を行うための`ConfirmChangeAnimalAction`を定義しており、そこで「変更」するか「キャンセル」するかの選択をするようにしています。  
`ConfirmChangeAnimalAction`の定義、実装のコードは以下のようになっています。  
```
enum ConfirmChangeAnimalAction: AlertActionType {
    var title: String {
        switch self {
        case .change(let animal):
            return "See \(animal)s"
        case .cancel:
            return "Stay"
        }
    }
    
    var style: AlertActionStyle {
        switch self {
        case .change:
            return .default
        case .cancel:
            return .cancel
        }
    }
    
    case change(to: Animal)
    case cancel
}
```
ちなみに上記で登場するAnimal型は単純なEnum型です  
    
```
enum Animal: String, Equatable {
    case cat = "Cat"
    case dog = "Dog"
    
    ...
}
```
そしてAlertStarategy側ではそのAction型が`ConfirmChangeAnimalAction`である時のメッセージとタイトルは定式化されており、複数箇所から利用される際にコードの重複を避けるため以下のような拡張実装を行います。  
```
extension AlertStrategy where Action == ConfirmChangeAnimalAction {
    init(animal: Animal) {
        self.init(title: "Change Animal Photos",
                  message: "Are you sure to see \(animal)'s photos?",
                  actions: [.change(to: animal),
                            .cancel])
    }
}
```
そしてここで定義した`ConfirmChangeAnimalAction`をAction型としてAlertStarategyはのちに見るようにPresenter側でそのインスタンスが生成され、ViewController側でAlertを表示するためのパラメーターとして利用されます。  
#### DataSource
記事で説明したように本設計におけるDataSourceはUIKitで提供される既存のDataSourceのラッパークラスとなります。  
既存のDataSourceをラッピングすることでCellの生成処理を主とした様々な実装をViewControllerの外部へと移譲して、ViewControllerの責務を「イベント処理」に限定することを目的としています。  
さて本記事ではiOS13で登場したUICollectionViewDiffableDataSourceして以下のようなDataSourceラッパークラスを定義、実装しています。  
```
class AnimalCollectionDataSourceWrapper {
    enum Section {
        case animalPhotos
    }
    
    typealias CellRegistration = UICollectionView.CellRegistration<UICollectionViewCell, Data>
    typealias DataSource = UICollectionViewDiffableDataSource<Section, Data>
    
    private let _datasource: DataSource
   
    init(collectionView: UICollectionView) {

        self._datasource = DataSource(collectionView: collectionView) {
            (collectionView: UICollectionView,
             indexPath: IndexPath,
             imageData: Data) -> UICollectionViewCell? in
            let registration: CellRegistration = .init(handler: { cell, indexPath, data in
                
                guard let image: UIImage = UIImage(data: data) else {
                    return
                }
                
                cell.contentMode = .scaleToFill
                let imageView: UIImageView = .init(image: image)
                imageView.contentMode = .scaleToFill
               
                imageView.translatesAutoresizingMaskIntoConstraints = false
                cell.contentView.addSubview(imageView)
             
                NSLayoutConstraint.activate([
                    imageView.topAnchor
                        .constraint(equalTo: cell.contentView.topAnchor),
                    imageView.centerXAnchor
                        .constraint(equalTo: cell.contentView.centerXAnchor),
                    imageView.heightAnchor.constraint(equalTo: cell.contentView.heightAnchor),
                    imageView.widthAnchor.constraint(equalTo: cell.contentView.widthAnchor)
                ])
            })
            return collectionView
                .dequeueConfiguredReusableCell(using: registration,
                                               for: indexPath,
                                                item: imageData)
            
      
        }
        
    }
    
    func update(newItems: [Data]) {
        var snapshot: NSDiffableDataSourceSnapshot<Section, Data> = .init()
        snapshot.appendSections([.animalPhotos])
        snapshot.appendItems(newItems,
                             toSection: .animalPhotos)
        self._datasource.apply(snapshot)
    }
}
```
具体的にはCellの生成処理と更新処理を実装し、更新処理`func update(newItems: [Data])`をインターフェースとして外部に公開することでViewControllerはCollectioViewのアイテム更新時に新しいアイテムを引数としてこのメソッドを呼び出しています。  
ちなみに`AnimalCollectionDataSourceWrapper`クラス内部でCell生成処理のため使用している`UICollectionView.CellRegistration`はiOS14以降で利用できます。  

#### Presenter
Presenterに関して本記事では主題として扱っていないですが、サンプルアプリの挙動を示すために簡単に説明します。(Presenterのデータ設計も妥協して実装しており、参考にしない方が良いと思います。)
##### Presenterに関するプロトコル
ViewController編でも説明しましたが、Presenterの設計はViewからの入力を処理する「PresenterInputs」プロトコルとその結果の出力先である「PresenterOutputs」プロトコルが基礎となっていて、
通常それぞれの実体型はPresenterクラス、ViewControllerクラスであるため全体のデータフローとしてはViewController->PresenterInputs(Presenterクラス)->PresenterOutputs(ViewController)となっています。  
そしてこのサンプルアプリではPresenterInputsの責務として「セットアップ/下部ボタン・Alertのボタンのタップの処理」、PresenterOutputsの責務として「アラートの表示と動物写真の表示」があり、それぞれの定義はコードでは以下のようになされます。  
```
protocol HogePresenterInputs: AnyObject {
    func setup()
    func tryChangeAnimalPhotoAlbum()
    func changeAnimalAlbum()
    
    init(initialDisplayAnimal: Animal,
         dogPhotoData: [Data],
         catPhotoData: [Data],
         output: HogePresenterOutputs)
}
```

```
protocol HogePresenterOutputs: AnyObject {
    func confirmChangeAnimal(strategy: AlertStrategy<ConfirmChangeAnimalAction>)
    func showAninmalAlbum(album: AnimalAlbum)
}
```
##### Presenterの実体型
先ほども述べた通りPresenterInputsプロトコルの実体型がPresenterクラスとなるのですが、今回のアプリではPresenterの設計は主題にはなっていないためその詳細については触れません。  
とりあえずここではPresenterはViewControllerから入力イベントを受け取って、View側で「Alertを表示する」また「新しい動物の写真を表示する」ための処理をして表示に必要なデータをHogePresenterOutputs(ViewController)に出力していることだけ把握してもらえれば十分です。  

実際のコードは以下のようになっています。  
```
final class HogePresenter: HogePresenterInputs {
    
    private var displayingAnimal: Animal
    private let dogPhotoData: [Data]
    private let catPhotoData: [Data]
    private weak var output: HogePresenterOutputs!
    
    init(initialDisplayAnimal: Animal,
         dogPhotoData: [Data],
         catPhotoData: [Data],
         output: HogePresenterOutputs) {
        self.displayingAnimal = initialDisplayAnimal
        self.dogPhotoData = dogPhotoData
        self.catPhotoData = catPhotoData
        self.output = output
    }
    
    func setup() {
        self._showAnimal()
    }
    
    func tryChangeAnimalPhotoAlbum() {
        let to: Animal = self.displayingAnimal.nextAnimal
        self.output
            .confirmChangeAnimal(strategy: AlertStrategy(animal: to))
    }
    
    func changeAnimalAlbum() {
        displayingAnimal.toggle()
        self._showAnimal()
    }
    
    @inline(__always)
    func _showAnimal() {
        let data: [Data]
        switch self.displayingAnimal {
        case .cat:
            data = self.catPhotoData
        case .dog:
            data = self.dogPhotoData
        }
        self.output.showAninmalAlbum(album: AnimalAlbum(animal: displayingAnimal,
                                                        photoData: data))
    }
}
```
    
ちなみにPresenterクラス内ではAnimal型のdisplayingAnimalインスタンスが`nextAnimal`変数と`toggle()`メソッドにアクセスしていますが、これは先ほど紹介したAnimal型の定義は全体としては以下のようになっておりこれらのプロパティ、メソッドにアクセスしています。  
    
```
enum Animal: String, Equatable {
    case cat = "Cat"
    case dog = "Dog"
    
    var nextAnimal: Animal {
        switch self {
        case .cat:
            return .dog
        case .dog:
            return .cat
        }
    }
    
    mutating fileprivate func toggle() {
        switch self {
        case .cat:
            self = .dog
        case .dog:
            self = .cat
        }
    }
}
```
    
    
#### ViewController
そして、これらView、Alert、PresenterのハブとしてViewControllerが存在します。  
ここでは最初にViewControllerのコードをお見せした上で、その入力処理と出力処理がどのように行われているか中心に説明していきます。    
    
    
ViewControllerのコード
```
class HogeViewController<Presenter: HogePresenterInputs,
                         SelectAnimalAlertClient: AlertClientType>: ViewController<HogeRootView>, HogePresenterOutputs
                            where SelectAnimalAlertClient.Action == ConfirmChangeAnimalAction {
    
    private var presenter: Presenter!
    private var datasource: AnimalCollectionDataSourceWrapper!
    private var alertClient: SelectAnimalAlertClient!
    init(initialAnimal: Animal,
         dogPhotoData: [Data],
         catPhotoData: [Data]) {
        super.init()
        self.alertClient = .init(viewController: self)
        self.presenter = .init(initialDisplayAnimal: .dog,
                               dogPhotoData: dogPhotoData,
                               catPhotoData: catPhotoData,
                               output: self)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
        
    override func viewDidLoad() {
        super.viewDidLoad()
        self.datasource = .init(collectionView: self.rootView.collectionView)
        
        // MARK: Inputs
        
        self.presenter.setup()
        
        self.rootView
            .button
            .addAction(UIAction(handler: { [weak self] _ in
                            self?.presenter.tryChangeAnimalPhotoAlbum()
                            }),
                        for: .touchUpInside)
        
        _ = self.alertClient
            .register(handler: { [weak self] (action: ConfirmChangeAnimalAction) in
                switch action {
                case .change:
                    self?.presenter.changeAnimalAlbum()
                    return
                case .cancel:
                    return
                }
            })
    }
    
    // MARK: Outputs
    func confirmChangeAnimal(strategy: AlertStrategy<ConfirmChangeAnimalAction>) {
        self.alertClient
            .show(strategy: strategy,
                  animated: true,
                  completion: nil)
    }
    
    func showAninmalAlbum(album: AnimalAlbum) {
        self.rootView.update(animalType: album.animal)
        self.datasource.update(newItems: album.photoData)
    }
}

```
##### ViewControllerの入力処理
本アプリではViewControllerの入力処理はviewDidLoad()内で行われます。(実際のほとんどのアプリでも同様にviewDidLoad()内だと思います。)  
具体的に本アプリで行なっている入力処理はセットアップ、画面下部ボタンタップ時の登録、Alertボタンのタップ時の登録の3つです。

###### セットアップ処理
これは言い換えるとViewのロード完了イベントの処理で、本アプリではViewがロードされたタイミングでPresenterのセットアップ処理を呼び出しています。  
```
override func viewDidLoad() {
   ...
   self.presenter.setup()
   ...
}
```
###### 画面下部ボタンタップ時の登録
こちらも
```
    
```
こちらも`
こちらも`
    
##### ViewControllerの出力処理
