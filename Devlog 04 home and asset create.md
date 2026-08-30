開発ログ #4：トップページ実装・資産登録機能実装編
はじめに

前回の開発ログでは、Spring Securityによるログイン認証機能とログイン画面を実装し、ログイン成功後にトップページへ遷移できるところまで確認した。

今回は、ログイン後に表示するトップページを実装し、データベースに登録されている資産数を画面に表示する機能を追加した。その後、資産管理アプリの中心となるCRUD機能のうち、**Create（資産登録）**の実装に着手した。

1. トップページの実装

ログイン成功後に表示するトップページを作成した。

当初は / にアクセスしても画面が表示されなかったため、HomeController を作成し、/ へのGETリクエストを処理するようにした。

java
@GetMapping("/")
public String home(Model model) {
    long assetCount = homeService.getAssetCount();
    model.addAttribute("assetCount", assetCount);
    return "home";
}

HomeController から HomeService を呼び出し、データベースに登録されている資産数を取得する構成にした。処理の流れは以下のようになる。

ブラウザ
  ↓
HomeController
  ↓
HomeService
  ↓
AssetRepository
  ↓
assetsテーブル
  ↓
資産数を取得
  ↓
home.htmlに表示

実際に画面を確認し、「総資産数：0件」と表示されることを確認した。この時点では資産がまだ登録されていないため、0件となるのは正常な状態である。

2. HomeServiceの実装

トップページから資産数を取得できるようにするため、HomeService を作成した。

java
@Service
public class HomeService {
    private final AssetRepository assetRepository;

    public HomeService(AssetRepository assetRepository) {
        this.assetRepository = assetRepository;
    }

    public long getAssetCount() {
        return assetRepository.count();
    }
}

AssetRepository の count() を利用することで、assets テーブルに登録されている資産の総数を取得できるようにした。

3. 貸出可能・貸出中の件数表示

トップページで資産の状態も確認できるようにするため、AssetRepository に以下のメソッドを追加した。

java
long countByStatus(String status);

これにより、status の値を指定して資産数を取得できるようにした。今回のアプリでは、資産の状態を以下の2種類で管理することにした。

貸出可能
貸出中

HomeService には、それぞれの件数を取得する処理を追加した。

java
public long getAvailableAssetCount() {
    return assetRepository.countByStatus("貸出可能");
}

public long getLoanedAssetCount() {
    return assetRepository.countByStatus("貸出中");
}

実装途中で、一度 getAvailableAssetCount() の中に "貸出中" を指定してしまい、メソッド名と実際に取得するデータの内容が一致しない状態になった。確認した結果、以下のように修正した。

java
public long getAvailableAssetCount() {
    return assetRepository.countByStatus("貸出可能");
}

この経験から、メソッド名と実際の処理内容を一致させることの重要性を学んだ。

最終的にトップページには、

総資産数：0件
貸出可能：0件
貸出中：0件

と表示されることを確認した。

4. 資産登録機能の実装開始

トップページの実装後、CRUD機能のうち最初の**Create（登録）**の実装に着手した。資産登録処理を担当する AssetService を作成した。

java
@Service
public class AssetService {
    private final AssetRepository assetRepository;

    public AssetService(AssetRepository assetRepository) {
        this.assetRepository = assetRepository;
    }

    public Asset createAsset(Asset asset) {
        asset.setStatus("貸出可能");
        return assetRepository.save(asset);
    }
}

新しく登録された資産は、まだ誰にも貸し出されていないため、登録時に自動的に「貸出可能」を設定するようにした。これにより、ユーザーが登録画面でステータスを入力しなくても、アプリ側で初期状態を設定できるようにした。

5. AssetControllerの実装

資産登録画面の表示と、登録処理を行う AssetController を作成した。

java
@Controller
public class AssetController {
    private final AssetService assetService;

    public AssetController(AssetService assetService) {
        this.assetService = assetService;
    }

    @GetMapping("/assets/new")
    public String showCreateForm() {
        return "asset-form";
    }

    @PostMapping("/assets")
    public String createAsset(@ModelAttribute Asset asset) {
        assetService.createAsset(asset);
        return "redirect:/";
    }
}

これにより、GET /assets/new で資産登録画面を表示し、POST /assets で登録処理を実行後、return "redirect:/"; によってトップページへ戻るようにした。

6. 資産登録画面の実装

asset-form.html を作成し、以下の入力項目を用意した。

資産コード
資産名
カテゴリ
レンタル開始日

フォームは以下のように実装した。

html
<form th:action="@{/assets}" method="post">
    <div>
        <label for="assetCode">資産コード：</label>
        <input type="text"
               id="assetCode"
               name="assetCode"
               required>
    </div>
    <div>
        <label for="name">資産名：</label>
        <input type="text"
               id="name"
               name="name"
               required>
    </div>
    <div>
        <label for="category">カテゴリ：</label>
        <input type="text"
               id="category"
               name="category"
               required>
    </div>
    <div>
        <label for="rentalStartDate">レンタル開始日</label>
        <input type="date"
               id="rentalStartDate"
               name="rentalStartDate"
               required>
    </div>
    <button type="submit">登録</button>
</form>
7. 「購入日」から「レンタル開始日」への変更

当初は資産の情報として「購入日」を設定していた。しかし、今回作成しているアプリはレンタルPCなどの資産管理を想定しているため、「購入日」ではなく「レンタル開始日」を管理する方が実際の用途に合っていると判断した。

そのため、Entity・フォーム・DBの項目を purchaseDate から rentalStartDate へ変更することにした。

8. Assetエンティティの変更

Asset エンティティに rentalStartDate を追加した。

java
@Column(nullable = false)
private LocalDate rentalStartDate;

また、フォームから値を受け取れるようにgetter / setterを追加した。

java
public LocalDate getRentalStartDate() {
    return rentalStartDate;
}

public void setRentalStartDate(LocalDate rentalStartDate) {
    this.rentalStartDate = rentalStartDate;
}

これにより、

asset-form.html
  ↓
rentalStartDate
  ↓
Asset.rentalStartDate

という形で、フォームからEntityへレンタル開始日を渡せる構成にした。

9. 資産登録時のエラー

実際に資産登録を試したところ、500エラーが発生した。エラー内容は以下のとおり。

not-null property references a null or transient value
for entity com.example.asset_management.entity.Asset.rentalStartDate

これは、Asset の rentalStartDate が null の状態でDBへ保存されようとしていることを示している。

10. エラーの原因調査

まず asset-form.html を確認した。

html
<input type="date"
       id="rentalStartDate"
       name="rentalStartDate"
       required>

となっており、フォーム側の name は正しく設定されていた。また Asset.java にも、

java
private LocalDate rentalStartDate;

public LocalDate getRentalStartDate() {
    return rentalStartDate;
}

public void setRentalStartDate(LocalDate rentalStartDate) {
    this.rentalStartDate = rentalStartDate;
}

が存在していることを確認した。そのため、フォーム側とEntity側だけでは原因を特定できず、Controller・Service・DBを含めてデータがどこで途切れているのかを確認しながら原因を切り分けることにした。

11. 現時点の進捗
項目	状態
要件定義・DB設計	完了
User / Asset / Loan Entity	完了
Repository層	完了
Spring Security認証	完了
ログイン画面	完了
トップページ	完了
総資産数表示	完了
貸出可能・貸出中の件数表示	完了
資産登録画面	完了
資産登録Controller	実装済み
資産登録Service	実装済み
資産登録DB保存	エラー調査中
資産一覧	未着手
資産編集	未着手
資産削除	未着手
貸出機能	未着手
返却機能	未着手
リマインド機能	未着手
ログアウト	追加予定
新規ユーザー登録	追加予定
12. 今回の開発で理解したこと

今回、トップページの実装を通して、Webアプリケーションにおける基本的なデータの流れを確認した。

ブラウザ
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
DB

トップページでは、

home.html
  ↓
HomeController
  ↓
HomeService
  ↓
AssetRepository
  ↓
MySQL

という流れで、DBから取得したデータを画面に表示する処理を実装した。

また、資産登録では、

asset-form.html
  ↓
AssetController
  ↓
AssetService
  ↓
AssetRepository
  ↓
MySQL

という、画面から入力したデータをDBへ保存する処理の実装に取り組んだ。

特に今回は、単にコードを書くことだけではなく、rentalStartDate の保存時エラーを通して、画面・Controller・Service・Entity・DBのそれぞれが正しくつながっている必要があることを実際のエラー調査を通して確認した。

次回

次回は、今回発生した rentalStartDate の保存エラーの原因を特定・修正し、資産登録処理を正常に完了させるところから再開する。その後、資産一覧・編集・削除などのCRUD機能へ進む予定。
