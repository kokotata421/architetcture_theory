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
従来のiOSアプリの設計におけるViewControllerとViewの関係があまりに近しいため、前回のViewController設計の記事でViewの設計についても基本的なことはほとんど書いてしまいました。  
ただ今回の設計においてViewはViewContorllerと明確に区別され独立した重要な機構です。  
そのため内容は前回記事と重複する箇所もありますが再度Viewの設計について書いていきます。    
前回の記事を読んだ人はAlertの説明箇所まで読み飛ばしてもらって大丈夫です。    

## Viewの設計
さて、既にお伝えした通り今回の設計ではViewをViewControllerとは明確に切り離しており、実装においてViewController毎の独自RootViewクラスの指定を必須としています。  
ここでは前回の記事の例でも出したHogeRootViewを参考にその独自のRootViewクラスをどのように定義、実装していけば良いか説明します。  
なお冒頭の前提でも述べた通り、UIはInterface Builderを利用せずコードを使って生成しています。   


### HogeRootViewの例
さっそくですが、HogeRootViewはHogeViewControllerのRootViewであり画面の構成は以下のようになっています。    
<img src="https://github.com/kokotata421/architetcture_theory/blob/main/Chapter5(View%7CAlert)/Images/RootViewの構成.png" alt="RootViewの構成" width=60% > 

実装も以下に記載している通りです。  
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
1点目は実装コードの冒頭に書かれているAppViewプロトコルについてです。  
このAppViewは各ViewControllerのRootViewであることを明示するためのプロトコルであり、RootViewとなるViewはこのプロトコルに準拠している必要があります。  
そして各RootViewでセットアップ処理を行いたい場合はこのAppViewプロトコルのsetup()メソッドにその処理を実装します。  
今回の例ではs
  
2点目はこのHogeRootViewのsetColorMode(lightMode: Bool)についてです。  
この画面ではhogeViewColorChangeButtonにタップすることで画面全体の色を変えられる仕様になっており、setColorMode(lightMode: Bool)はその画面の色を変更するメソッドとなります。  

そして最後に


## Alertの設計
