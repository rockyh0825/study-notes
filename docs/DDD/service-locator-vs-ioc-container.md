# Service LocatorパターンとIoC Containerパターン

## 依存関係逆転の原則（DIP）

依存関係逆転の原則（Dependency Inversion Principle）はSOLID原則の一つで、次の2点を要求する。

- 上位モジュールは下位モジュールに依存してはならない。両者とも抽象（インターフェース）に依存すべき
- 抽象は詳細に依存してはならない。詳細（実装）が抽象に依存すべき

DDDでいえば「リポジトリのインターフェースをドメイン層に置き、実装をインフラ層に置く」構成がまさにこの原則の適用例で、ドメイン層（上位）はインフラ層（下位）の実装詳細を知らずに、インターフェースという抽象だけに依存する。

この原則を実現する＝実際に依存オブジェクトを「どう調達するか」を解決する手段として、代表的なものにService LocatorパターンとIoC Container（DIコンテナ）パターンがある。

## Service Locatorパターン

Service Locatorパターンは、依存オブジェクトを一元管理する「ロケータ」を用意し、必要なクラスがそのロケータに問い合わせて依存オブジェクトを取得する方式。

```java
public class OrderService {
    public void placeOrder(Order order) {
        // 自分でロケータに問い合わせて依存を取得しにいく
        PaymentGateway gateway = ServiceLocator.get(PaymentGateway.class);
        gateway.charge(order);
    }
}
```

特徴:

- クラス自身が能動的に依存を「取りに行く」（pull型）
- コンストラクタやメソッドのシグネチャに依存関係が現れないため、外から見て何に依存しているか分かりにくい
- `ServiceLocator` 自体への依存がすべてのクラスに広がってしまう
- テスト時にロケータへ差し替え用の実装を登録する手間がかかり、実装漏れがあると実行時まで気づけない

## IoC Container（DIコンテナ）パターン

IoC Container（Inversion of Control Container、いわゆるDIコンテナ）は、フレームワーク側がオブジェクトの生成と依存の注入を肩代わりする方式。Spring Bootの `ApplicationContext` が代表例。

```java
@Service
public class OrderService {
    private final PaymentGateway gateway;

    // コンテナがコンストラクタ経由で依存を注入してくれる（push型）
    public OrderService(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    public void placeOrder(Order order) {
        gateway.charge(order);
    }
}
```

特徴:

- クラス自身は依存を取得しにいかず、コンテナから「渡される」（push型）
- コンストラクタのシグネチャを見るだけで、そのクラスが何に依存しているか一目で分かる
- クラスはコンテナの存在自体を知らなくてよい（コンテナへの依存がコード中に現れない）
- テスト時はコンストラクタにモックを直接渡すだけでよく、DIコンテナなしでも単体テストが書きやすい

## 2つのパターンの違い

| 観点 | Service Locator | IoC Container |
|---|---|---|
| 依存の取得方法 | クラスが能動的に問い合わせる（pull） | コンテナが自動的に渡す（push） |
| 依存の可視性 | シグネチャに現れず隠れやすい | コンストラクタ引数として明示される |
| ロケータ/コンテナへの結合 | 利用側コードに残る | 利用側コードには残らない |
| テストのしやすさ | ロケータへの登録が必要で煩雑になりがち | コンストラクタにモックを渡すだけで済む |

どちらもDIP（依存関係逆転の原則）を満たすための実現手段だが、依存関係が明示的でテストしやすいIoC Container（DIコンテナ）方式の方が現代的な設計では好まれる傾向にある。Spring BootのDI機能はこのIoC Containerパターンの実装にあたる。
