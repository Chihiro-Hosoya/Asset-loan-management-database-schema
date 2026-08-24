## 6. 資産登録機能の実装

ログイン後に表示する資産管理機能の第一段階として、資産登録機能を実装した。`AssetController`・`AssetService`・`AssetRepository`・`Asset` Entityを通じて、画面から入力した資産情報をMySQLの`assets`テーブルへ登録し、登録時にステータスを「貸出可能」に設定する処理を組んだ。

### 6-1. Asset Entityの整理

今回のアプリでは購入日を管理対象としない方針としたため、`Asset` Entityから購入日に関する項目を削除した。

```java
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
```

現時点で管理している資産情報は以下の5項目。

| 項目 | 内容 |
|---|---|
| id | 資産ID |
| assetCode | 資産番号 |
| name | 資産名 |
| category | カテゴリ |
| status | 貸出状態 |

### 6-2. AssetRepositoryの実装

```java
public interface AssetRepository extends JpaRepository<Asset, Long> {

  long countByStatus(String status);

}
```

`JpaRepository<Asset, Long>`を継承することで、基本的なCRUD操作を自分でSQLを書くことなく利用できる(例:`assetRepository.save(asset);`)。

また、`countByStatus(String status)`を実装し、ステータスごとの資産数(総資産数/貸出可能/貸出中)を集計できるようにした。

### 6-3. AssetServiceの実装

```java
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
```

ポイントは`asset.setStatus("貸出可能");`。資産登録画面では利用者がステータスを入力するのではなく、新しく登録された資産は必ず「貸出可能」から始まるようにした。そのため、Controllerから渡された`Asset`に対してService層でステータスを設定してから、Repositoryを通してDBへ保存する設計とした。

処理の流れ:

```
資産登録画面 → AssetController → AssetService（statusを「貸出可能」に設定） → AssetRepository → MySQL
```

### 6-4. AssetControllerの実装

```java
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
```

`@GetMapping("/assets/new")`では資産登録画面を表示し、`@PostMapping("/assets")`ではフォームから送信されたデータを`@ModelAttribute Asset asset`によってAssetオブジェクトに受け取り、Service層へ渡している。登録処理完了後は`redirect:/`でトップページへリダイレクトする。

---

## 7. DBとEntityの構造不一致によるエラー対応

### 症状

資産登録を実行したところ、以下のエラーが発生した。

```
Field 'purchase_dates' doesn't have a default value
```

Hibernateが実行しようとしていたSQLを確認すると、

```sql
insert into assets (asset_code,category,name,rental_start_date,status) values (?,?,?,?,?)
```

となっていた。Java側の`Asset` Entityでは`purchase_dates`を使用していないため、**Java側のEntityとMySQL側の`assets`テーブルの構造が一致していない**ことが原因と判断した。

### 原因調査:MySQLの接続先を確認

まず`SHOW CREATE TABLE assets;`や`DESCRIBE assets;`を実行したところ、`ERROR 1046 (3D000): No database selected`が発生。`SELECT DATABASE();`を実行すると`NULL`が返り、使用するデータベースが選択されていない状態だと分かった。

```sql
SHOW DATABASES;
-- asset_management / enemy_down / information_schema / mysql / performance_schema / spigot_server / sys

USE asset_management;
SELECT DATABASE();
-- asset_management が選択されていることを確認
```

### 原因調査:assetsテーブルの構造を確認

`asset_management`を選択した状態で`DESCRIBE assets;`を実行すると、Java側では使用していない`purchase_dates`・`purchase_date`・`username`・`rental_start_date`などのカラムがDB側に残っていることが判明した。特に`purchase_dates`はNOT NULLかつデフォルト値が未設定だったため、HibernateのINSERT時に値が渡されず`Field 'purchase_dates' doesn't have a default value`というエラーが発生していた。

### 対処

購入日を管理しない方針に合わせて、DB側の不要なカラムを整理し、Java側のEntityと構造を一致させた。最終的な`assets`テーブルの構成は以下の通り。

```
id / asset_code / name / category / status
```

再度資産登録を実行したところ、正常に完了。トップページにも「総資産数:1件 / 貸出可能:1件 / 貸出中:0件」と表示され、登録した資産が正しくDBに保存されステータスが反映されていることを確認した。

---

## 8. 今回理解したこと

- Spring Bootでは、Javaの`Entity`を修正するだけでなく、**Entityと実際のDBテーブル構造が一致している必要がある**ことを再確認した
- エラーの追い方として、`Whitelabel Error Page` → `500 Internal Server Error` → `DataIntegrityViolationException` → 具体的なエラーメッセージ(`Field 'purchase_dates' doesn't have a default value`)というように、階層を1つずつ掘り下げることで原因を特定できた
- `SELECT DATABASE();`で現在選択中のDBを確認し、`DESCRIBE テーブル名;`で実際のテーブル構造を確認するという、**DB側から原因を切り分ける方法**を経験できた

---

## 9. 現時点の進捗状況

| 項目 | 状態 |
|---|---|
| 要件定義・DB設計 | 完了 |
| エンティティ実装(User / Asset / Loan) | 完了 |
| Repository層 | 完了 |
| Spring Security(認証の仕組み) | 完了 |
| ログイン画面・動作確認 | 完了 |
| トップページ | 完了 |
| 資産登録機能 | 完了 |
| 資産一覧 | 未着手 |
| 資産編集・削除 | 未着手 |
| 貸出機能 | 未着手 |
| 返却機能 | 未着手 |
| リマインド機能 | 未着手 |

次回は、登録した資産を一覧表示する資産一覧機能の実装に着手する。
