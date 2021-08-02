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
最初にデフォルトAlertの開発を踏まえながら、今回のAlert設計に当たって解決すべき問題点を確認します。  

### デフォルトAlertの問題点
私はアプリ設計の観点からデフォルトAlertには以下3点の問題点があると考えています。  
1. 表示するために必要な設定データが多くプログラムが命令的
2. アプリの機能との連携が見えづらい
3. データフローが複雑になる

なんのことを言っているか大体検討がついている点もあるかと思いますが、以下ではそれぞれについて簡単に説明します。  
#### 1.表示するために必要な設定が多くプログラムが命令的
これは一般的に認識されている問題なので、想像するのは難しくないでしょう。    
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
本記事ではViewControllerの代理としてAlertの表示を行うコンポーネントはAlertClientとします。  
その基本的な構造はRouterと同じで、AlertClientにViewControllerを渡してそちら側でAlert表示の実装を行います。  
これはAlertの表示が技術的にはViewControllerの`present(_:animated:completion:)`メソッド、すなわち遷移処理によって実行されていることを考えれば当然のことです。  

しかし、技術的には同類でもやはりサービスの観点からいうと「遷移」と「アラートの表示」は異なります。  
そのためAlertの表示を行うコンポーネントをAlertClientとしてRouterと区別しているわけですが、本記事が提案する設計ではコンポーネントがAlertClientである条件としてAlertClientTypeというプロトコルに準拠しているものとします。  
以下ではAlertClientTypeプロトコルのAlertの表示に関する定義のみ紹介します。(AlertClientTypeプロトコルの定義全体は後ほど示します。)　　

```
protocol AlertClientType: NSObject {
    associatedtype Action: AlertActionType
    init(viewController: UIViewController)
    
    func show(strategy: AlertStrategy<Action>,
              animated: Bool,
              completion: (() -> Void)?)
   ...
```


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

しかし通常のラッパーオブジェクトと異なる点としてはAlertStrategyでジェネリクスとして宣言している`<Action: AlertActionType>`型によってアプリ機能に対応したAlertのモジュール化を可能にしています。  
AlertActionTypeについても後ほど詳しく説明します。  

ちなみにUIKitに｀UIAlertController.Style｀型があるにも関わらず、わざわざ自作でAlertStyle型を定義したのはAlertStrategy型は性質上ViewModel等でも利用するためUIKitに依存した設計にしたくなかったからであって深い理由はありません。  
AlertのUI側では以下のように`UIAlertController.Style`に拡張的初期化処理を定義してAlertStyle型から生成できるようにしています。  
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

また`extension AlertStrategy: Error {}`とAlertStrategy型をErrorプロトコルに準拠させているのは、性質上Result型のFailure型として扱われる場合があるためです。  

#### 「2.アプリ機能との連携が見えづらい」問題の解決
既に述べた通り、今回の設計ではAlertActionTypeプロトコルを利用することでアプリ固有の機能に合わせたAlertのモジュール化を実現しています。  
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
このAlertActionTypeに準拠する実体型では対応するモジュール名を型名として定義します。  
これによってAlertの開発時には利用しているAlertActionTypeの実体型からそのコンテクストが一目わかるようになります。  
例えば今回のサンプルプロジェクトでは写真アイテムの取得失敗時にアラートを表示する仕様になっているのですが、それに対応したAlertActionTypeとしてFetchPhotoErrorAction型を定義・実装しています。  
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
そして、「写真アイテムの取得失敗」に関連するAlertを表示する際にはAlertを表示するクライアントは`AlertClient<FetchPhotoErrorAction>`、また表示時にそれに渡すAlertStrategyも`AlertStrategy<FetchPhotoErrorAction>`という型名になるため一眼でそれが写真取得失敗時に関するアラートであることがわかります。(AlertClient型については後ほど詳しく説明します。)  
ちなみに基本的にAlertActionTypeの実体型は上記のようにEnumで定義し、ユーザーがAlertに対して取りうる手段をcaseとして宣言していきます。  
まあいうまでもないと思いますが、AlertActionTypeの`title:String`はAlertボタンに表示される文言、`style: AlertActionStyle`は`UIAlertAction.Style`と同じです。  
`UIAlertAction.Style`を使わずにわざわざAlertActionStyle型を自作で定義しているのは先程のAlertStyleと同じ理由です。  

#### 「3.データフローが複雑になる」問題の解決
デフォルトのAlertでは出力(Alertの表示)と入力(Alertボタンタップ時の処理)を切り離せないため、ViewControllerのデータフローが複雑になってしまうと先程説明しました。  
ここではAlertClient(Alertの表示を担うコンポーネント)を工夫してAlertの出力と入力を切り離す方法を説明します。  
ただ「出力と入力の切り離し」の前にAlertClientの基本的な性質から確認していきます。  
AlertClientは抽象レベルではAlertClientTypeに準拠するようになっており、これは既に説明した通りViewControllerの代理としてAlertの表示を行うコンポーネントです。    
先程もAlertClientTypeの一部を示しましたが、その全体の定義は以下のようになっています。  
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

protocol AlertClientType: NSObject {
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

AlertClientの基本的な役割でAlertの表示は上記の`func show(strategy: AlertStrategy<Action>, animated: Bool, completion: (() -> Void)?)`によって行われます。  
そしてここで主題となっているAlertの入力と出力処理を切り離すのはregisterメソッドによって実現されています。  
定義を見てわかると思いますが、このregisterメソッドではクロージャをパラメーターとして渡して登録しており、Alertボタンタップ時にはここで登録したクロージャが呼び出されるようになっています。  
例えばAlertClientが先程の`FetchPhotoErrorAction`をAction型として指定してる場合には以下のようにクロージャを渡すことでAlertボタンタップ時の処理を登録しています。  
```
//　alertClientはAction型に「FetchPhotoErrorAction」を指定したAlertClientTypeの実体型インスタンス

　　　alertClient
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
デフォルトのAlert設計よりもこちらの方がAlertのボタンタップ時の処理を直感的に登録できていると思います。  
ちなみに`func register(on action: Action,_ handler: @escaping (Action) -> Void) -> RegistryKey`と`func register(on actions: [Action],_ handler: @escaping (Action) -> Void) -> RegistryKey`は何か特定のActionが発生した場合のみ呼び出したい処理を登録する場合に利用します。  
またRegistryKey型はAlertに登録した処理を管理するのに利用するキーであり、もし登録した処理が呼びされるのをやめた場合は該当の処理を登録時返り値として受け取ったRegistryKeyを`func unregister(key: RegistryKey) -> Void?`に渡すことで登録を解除できます。  


このAlertClientTypeの実体型として私は以下のAlertClient型を定義しました。
```
final class AlertClient<Action: AlertActionType>: NSObject {
    private weak var vc: UIViewController!
    var handlers: [RegistryKey: (Action) -> Void] = [:]
    
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
