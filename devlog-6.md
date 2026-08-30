# 開発ログ #6：資産CRUD機能実装編

## はじめに

前回の開発ログでは、Spring Securityによる認証機能とログイン画面を実装し、ログイン動作まで確認した。
今回は、ログイン後に利用する資産管理機能の実装に着手した。

実装した機能は以下のとおり。

- 資産登録
- 資産一覧表示
- 資産編集画面
- 資産更新
- 資産削除
- レンタル開始日の追加

開発途中では、データベースとEntityの定義が一致していないことによるエラーや、Thymeleafの記述ミスによるエラーにも直面した。本記事では、実装内容とあわせてこれらのトラブルシューティングも記録する。

---

## 1. Asset Entityの整理

資産情報を管理する `Asset` エンティティを整理した。最終的に、資産には以下の情報を持たせることにした。

```java
private Long id;
private String assetCode;
private String name;
private String category;
private String status;
private LocalDate rentalStartDate;
```

当初は購入日などの項目も存在していたが、今回のアプリでは不要と判断し削除した。

| 項目 | 内容 |
|---|---|
| id | 資産ID |
| assetCode | 資産コード |
| name | 資産名 |
| category | カテゴリ |
| status | 貸出状態 |
| rentalStartDate | レンタル開始日 |

`status` には「貸出可能」「貸出中」などの状態を保持する設計とした。

---

## 2. データベースの整理

MySQLで `asset_management` データベースを使用していることを確認した。

```sql
USE asset_management;
```

`SHOW TABLES;` を実行し、以下のテーブルが存在することを確認した。

- assets
- loans
- users

### assetsテーブル

最終的に `assets` テーブルは以下の構成となった。

```
+-------------------+
| Field             |
+-------------------+
| id                |
| asset_code        |
| category          |
| status            |
| name              |
| rental_start_date |
+-------------------+
```

### rental_start_date 追加時のトラブル

Javaの `Asset` クラスに `rentalStartDate` を追加した際、データベース側にカラムが存在していなかったため、以下のエラーが発生した。

```
Unknown column 'a1_0.rental_start_date' in 'field list'
```

原因は、Java側のEntityには `rentalStartDate` が存在するが、MySQLの `assets` テーブルには対応するカラムが存在しないことだった。MySQL側に以下のカラムを追加することで対応した。

```sql
ALTER TABLE assets
ADD COLUMN rental_start_date DATE NULL;
```

※途中で `ALTER` を `AKTER` と入力してしまい、SQLの構文エラーも発生した。

---

## 3. 資産登録機能

`AssetRepository` にはSpring Data JPAの `JpaRepository` を利用した。

```java
public interface AssetRepository extends JpaRepository<Asset, Long> {
    long countByStatus(String status);
}
```

これにより、基本的な登録・検索・更新・削除処理をRepository側で利用できるようにした。

---

## 4. AssetServiceの実装

資産に関する処理を `AssetService` にまとめた。

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

    public List<Asset> findAllAssets() {
        return assetRepository.findAll();
    }

    public Asset findAssetById(Long id) {
        return assetRepository.findById(id).orElseThrow();
    }

    public Asset updateAsset(Asset asset) {
        asset.setStatus("貸出可能");
        return assetRepository.save(asset);
    }

    public void deleteAsset(Long id) {
        assetRepository.deleteById(id);
    }
}
```

### 登録時のステータス設定

資産登録時には `asset.setStatus("貸出可能");` を実行することで、登録した資産の初期状態を「貸出可能」に設定した。

---

## 5. 資産一覧表示機能

`AssetController` に資産一覧を表示する処理を追加した。

```java
@GetMapping("/assets")
public String listAssets(Model model) {
    model.addAttribute("assets", assetService.findAllAssets());
    return "assets";
}
```

`AssetService` から資産一覧を取得し、`model.addAttribute("assets", ...)` によってThymeleafの画面に渡す仕組みとした。ブラウザから `http://localhost:8080/assets` へアクセスすることで資産一覧を表示できることを確認した。

---

## 6. 資産一覧画面の実装

`assets.html` を作成し、登録されている資産を一覧表示できるようにした。

```html
<tr th:each="asset : ${assets}">
    <td th:text="${asset.assetCode}"></td>
    <td th:text="${asset.name}"></td>
    <td th:text="${asset.category}"></td>
    <td th:text="${asset.rentalStartDate}"></td>
    <td th:text="${asset.status}"></td>
</tr>
```

`th:each` を利用することで、Controllerから渡された資産のリストを1件ずつ取り出して表示している。

---

## 7. 資産編集機能

一覧画面に「編集」リンクを追加した。

```html
<a th:href="@{/assets/{id}/edit(id=${asset.id})}">
    編集
</a>
```

例えばIDが `1` の場合、`/assets/1/edit` というURLになる。Controllerでは以下のようにして、対象IDの資産情報を取得し編集画面に渡すようにした。

```java
@GetMapping("/assets/{id}/edit")
public String showEditForm(@PathVariable Long id, Model model) {
    Asset asset = assetService.findAssetById(id);
    model.addAttribute("asset", asset);
    return "asset-edit";
}
```

ブラウザから編集を押して、対象の資産情報が編集画面に表示されることを確認した。

---

## 8. 資産更新機能

編集画面から変更した資産情報を更新できるようにした。Controllerに以下の処理を追加した。

```java
@PostMapping("/assets/{id}")
public String updateAsset(
        @PathVariable Long id,
        @ModelAttribute Asset asset) {
    asset.setId(id);
    assetService.updateAsset(asset);
    return "redirect:/assets";
}
```

更新処理では、URLから取得したIDをEntityに設定した上で、Serviceの更新処理を呼び出している。更新後は `return "redirect:/assets";` によって資産一覧画面へ戻るようにした。

---

## 9. 更新時のエラー

更新ボタンを押した際、以下のエラーが発生した。

```
not-null property references a null or transient value
for entity com.example.asset_management.entity.Asset.status
```

原因は、編集フォームから送信された `Asset` に `status` が含まれておらず、更新時に `status` が `null` になっていたこと。一方 `Asset` では `@Column(nullable = false) private String status;` となっているため、データベースに `NULL` を保存することができなかった。

そこで `updateAsset()` でも登録時と同じように `asset.setStatus("貸出可能");` を設定してから保存することで解決した。

```java
public Asset updateAsset(Asset asset) {
    asset.setStatus("貸出可能");
    return assetRepository.save(asset);
}
```

これにより、編集画面から資産情報を変更して更新できることを確認した。

---

## 10. レンタル開始日の追加

資産情報として「レンタル開始日」を追加した。

Java側では `private LocalDate rentalStartDate;` を追加し、getter / setterも実装した。

HTML側では以下のようにした。

```html
<label for="rentalStartDate">レンタル開始日</label>
<input type="date"
       id="rentalStartDate"
       name="rentalStartDate"
       th:field="*{rentalStartDate}"
       required>
```

`type="date"` を利用することで、ブラウザ上で日付入力用のUIを利用できるようにした。

---

## 11. レンタル開始日の入力で発生したエラー

更新時に、以下のエラーが発生した。

```
Failed to convert from type [java.lang.String]
to type [java.time.LocalDate]
```

原因は、送信された日付が `2026/8/27` という形式になっていたのに対し、`LocalDate` への変換に適した形式と一致していなかったこと。HTMLの `input type="date"` と `th:field` を正しく設定することで対応した。

最終的にはレンタル開始日を登録・表示・更新できる状態になった。

---

## 12. 資産削除機能

CRUDの最後の機能として、資産削除機能を実装した。

Serviceには以下を追加した。

```java
public void deleteAsset(Long id) {
    assetRepository.deleteById(id);
}
```

Controllerには以下を追加した。

```java
@PostMapping("/assets/{id}/delete")
public String deleteAsset(@PathVariable Long id) {
    assetService.deleteAsset(id);
    return "redirect:/assets";
}
```

一覧画面には「削除」ボタンを追加した。

```html
<form th:action="@{/assets/{id}/delete(id=${asset.id})}"
      method="post"
      style="display:inline;">
    <button type="submit">削除</button>
</form>
```

`style="display:inline;"` は、編集リンクと削除ボタンを横並びに表示するための指定であり、削除処理そのものには関係しない。

---

## 13. Thymeleafの記述ミス

削除機能実装時、資産一覧画面を開くと `TemplateInputException` が発生した。ログを確認すると、以下のように表示されていた。

```
Could not parse as expression:
"@{/assets/{id/delete(id=${asset.id})}"
```

原因は、URLの `{id}` の閉じ括弧が抜けていたことだった。

```
誤: @{/assets/{id/delete(id=${asset.id})}
正: @{/assets/{id}/delete(id=${asset.id})}
```

修正後、`/assets` へ正常にアクセスできるようになった。その後、削除ボタンを押して対象の資産が一覧から消えることを確認し、削除機能の動作確認も完了した。

---

## 14. 現時点の進捗状況

| 項目 | 状態 |
|---|---|
| 要件定義・DB設計 | 完了 |
| Entity実装 | 完了 |
| Repository層 | 完了 |
| Spring Security | 完了 |
| ログイン画面 | 完了 |
| 資産登録 | 完了 |
| 資産一覧 | 完了 |
| 資産編集 | 完了 |
| 資産更新 | 完了 |
| 資産削除 | 完了 |
| レンタル開始日 | 完了 |
| 貸出機能 | 未着手 |
| 返却機能 | 未着手 |
| リマインド機能 | 未着手 |
| CSS・画面整理 | 未着手 |

---

## 15. 振り返り

今回は、資産管理アプリの基本的なCRUD機能を一通り実装した。特に、

- JavaのEntity
- Repository
- Service
- Controller
- Thymeleaf
- MySQL

という複数の層を連携させ、画面から入力したデータをデータベースへ保存し、一覧表示・編集・更新・削除まで行う一連の流れを実装できた。

また、開発中には以下のようなエラーが発生した。

- JavaのEntityとDBのカラム不一致
- `LocalDate` へのデータ変換エラー
- `status` が `null` になることによるDB制約エラー
- ThymeleafのURL式の記述ミス

エラーが発生した際には、ブラウザのエラー画面だけでなく、Spring Bootのコンソールログを確認することで、原因となっているファイルや行を特定して修正した。特に今回、`template: "assets" - line 34, col 13` というログから `assets.html` の該当箇所を確認し、`{id}` の閉じ括弧が抜けていることを特定できた。

単にコードを完成させるだけでなく、エラーログから原因を切り分けて修正する経験を積むことができた。

---

## 16. 次回の予定

次回は、今回作成した `Loan` エンティティを利用して、資産の貸出・返却機能の実装に着手する。想定している流れは以下のとおり。

```
資産一覧
  ↓
貸出
  ↓
誰に貸したか・貸出日・返却期限を登録
  ↓
ステータスを「貸出中」に変更
  ↓
返却
  ↓
返却日を登録
  ↓
ステータスを「貸出可能」に変更
```

その後、リマインド機能を実装し、最後にCSSや画面レイアウトを整理する予定。
