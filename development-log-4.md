資産管理リマインドアプリ 開発ログ #4

資産登録機能実装・DB不整合の修正・動作確認編
前回の開発ログでは、Spring Securityによる認証機能とログイン画面を実装し、ログイン処理までの動作確認を行った。
今回は、ログイン後に表示する資産管理機能の第一段階として、資産登録機能の実装に着手した。
AssetController、AssetService、AssetRepository、Asset Entityを利用して、画面から入力した資産情報をMySQLのassetsテーブルへ登録し、登録時にはステータスを「貸出可能」に設定する処理を実装した。
また、実装途中でJava側のEntityと既存のDBテーブルの構造が一致していないことによるエラーが発生したため、MySQLのテーブル構造を確認しながら原因を切り分けた。

1. Asset Entityの整理
資産情報を管理するAsset Entityを整理した。

@Entity
@Table(name = "assets")
public class Asset {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, unique = true)
  private String assetCode;

  @Column(nullable = false)
  private String name;

  @Column(nullable = false)
  private String category;

  @Column(nullable = false)
  private String status;

  // getter / setter
}

今回の資産管理アプリでは、購入日は管理対象としない方針としたため、購入日に関する項目をAsset Entityから削除した。
また、資産の貸出状態はstatusで管理することとし、新規登録された資産は「貸出可能」とする設計にした。
現時点では、資産情報として以下を管理している。

項目

内容

id

資産ID

assetCode

資産番号

name

資産名

category

カテゴリ

status

貸出状態

2. AssetRepositoryの実装

AssetRepositoryでは、Spring Data JPAのJpaRepositoryを継承している。

public interface AssetRepository extends JpaRepository<Asset, Long> {

  long countByStatus(String status);

}

JpaRepository<Asset, Long>を継承することで、基本的なCRUD操作を自分でSQLを書くことなく利用できる。

例えば、

assetRepository.save(asset);

とすることで、Asset Entityの情報をデータベースへ保存できる。
また、既に実装していた、
long countByStatus(String status);
によって、ステータスごとの資産数を取得できるようにしている。
総資産数
貸出可能
貸出中
の集計に利用している。


3. AssetServiceの実装
資産登録処理をService層に実装した。

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

今回のポイントは、asset.setStatus("貸出可能");である。
資産登録画面では、利用者がステータスを入力するのではなく、新しく登録された資産は必ず「貸出可能」から始まるようにした。
そのため、Controllerから渡されたAssetに対してService層でステータスを設定してから、Repositoryを通してDBへ保存する設計とした。

処理の流れは以下のようになる。
資産登録画面
    ↓
AssetController
    ↓
AssetService
    ↓
statusを「貸出可能」に設定
    ↓
AssetRepository
    ↓
MySQL


4. AssetControllerの実装
資産登録画面の表示と、フォームから送信された資産情報の登録処理をAssetControllerに実装した。

@Controller
public class AssetController {

  private final AssetService assetService;

  public AssetController(AssetService assetService) {
    this.assetService = assetService;
  }

  // 資産登録画面を表示
  @GetMapping("/assets/new")
  public String showCreateForm() {
    return "asset-form";
  }

  // 資産を登録
  @PostMapping("/assets")
  public String createAsset(@ModelAttribute Asset asset) {
    assetService.createAsset(asset);
    return "redirect:/";
  }
}

@GetMapping("/assets/new")では資産登録画面を表示する。
一方、@PostMapping("/assets")ではフォームから送信されたデータを@ModelAttribute Asset assetによってAssetオブジェクトに受け取り、Service層へ渡している。

登録処理が完了した後は、return "redirect:/";　によってトップページへリダイレクトする。

5. 資産登録時にDBエラーが発生
実際に資産登録を行ったところ、以下のエラーが発生した。
Field 'purchase_dates' doesn't have a default value

さらに、Hibernateが実行しようとしていたSQLを確認すると、
insert into assets
(asset_code,category,name,rental_start_date,status)
values (?,?,?,?,?)
となっていた。

一方、Java側のAsset Entityではpurchase_datesを使用していなかった。
つまり、Java側のEntityと、MySQL側のassetsテーブルの構造が一致していないことが原因だと判断した。

6. MySQLの接続先を確認

最初に、SHOW CREATE TABLE assets;やDESCRIBE assets;を実行したが、ERROR 1046 (3D000): No database selectedというエラーが発生した。
SELECT DATABASE();を実行したところ、
+------------+
| DATABASE() |
+------------+
| NULL       |
+------------+
となっており、使用するデータベースが選択されていないことが分かった。
そこで、SHOW DATABASES;を実行し作成済みのデータベースを実行した。
asset_management
enemy_down
information_schema
mysql
performance_schema
spigot_server
sys
今回のアプリで使用しているasset_managementを選択した。

USE asset_management;その後、SELECT DATABASE();を実行し、asset_managementが選択されていることを確認した。


7. assetsテーブルの構造を確認

asset_managementを選択した状態で、DESCRIBE assets; 実行した。
すると、Java側では使用していない、
purchase_dates
purchase_date
username
rental_start_date
などのカラムがDB側に残っていることが分かった。
特に、purchase_datesはNOT NULLかつデフォルト値が設定されていなかった。
そのため、HibernateがINSERTを実行した際にpurchase_datesへ値が渡されず、
Field 'purchase_dates' doesn't have a default value というエラーが発生していた。


8. DBとEntityの構造を整理

今回のアプリでは購入日を管理しない方針としたため、Java側のEntityとDB側のテーブルについて不要な項目を整理した。
最終的にassetsテーブルは、資産管理に必要な項目を中心に構成した。
id
asset_code
name
category
status
これにより、Java側のAsset Entityとデータベース側の構造を合わせることができた。

9. 資産登録処理の動作確認

DB構造を整理した後、再度資産登録を実行した。その結果、登録処理が正常に完了した。
トップページにも、
総資産数：1件

貸出可能：1件

貸出中：0件
と表示され、登録した資産が正しくDBに保存され、ステータスが「貸出可能」として反映されていることを確認した。

10. 今回理解したこと

今回の実装を通して、Spring Bootでは単にJavaのEntityを修正するだけではなく、Entityと実際のデータベースのテーブル構造が一致している必要があることを確認した。
また、エラーが発生した際に、
Whitelabel Error Page
        ↓
500 Internal Server Error
        ↓
DataIntegrityViolationException
        ↓
Field 'purchase_dates' doesn't have a default value
        ↓
MySQLのassetsテーブルを確認
というように、エラーの下層まで確認することで原因を特定できた。特に今回、SELECT DATABASE();で現在選択されているデータベースを確認し、
DESCRIBE assets;で実際のテーブル構造を確認するという、DB側から原因を確認する方法を経験できた。

11. 現時点の進捗状況
項目

状態

要件定義・DB設計

完了

エンティティ実装(User / Asset / Loan)

完了

Repository層

完了

Spring Security(認証の仕組み)

完了

ログイン画面・動作確認

完了

トップページ

完了

資産登録機能

完了

資産一覧

未着手

資産編集・削除

未着手

貸出機能

未着手

返却機能

未着手

リマインド機能

未着手

次回は、登録した資産を一覧表示する資産一覧機能の実装に着手する。
