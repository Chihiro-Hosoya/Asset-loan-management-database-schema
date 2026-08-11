# 開発ログ・トラブルシューティング記録

資産管理リマインドアプリの開発過程で行った設計判断、発生したエラー、修正内容を時系列でまとめたものです。
「動くコードを書けたか」だけでなく、「なぜその設計にしたか」「エラーにどう向き合ったか」を記録しています。

要件定義は [README.md](./README.md) を参照してください。ここでは設計判断とエラー対応の過程を記録しています。

---

## 1. DB設計

### テーブル構成の考え方

エンティティ(登場人物・モノ)を洗い出し、関係性(1対多など)を整理してからテーブルに落とし込んだ。

- ユーザー:資産の貸出/返却を行う人
- 資産:PCや備品そのもの
- 貸出:誰が・何を・いつ借りたかの記録

### なぜ`loans`テーブルを独立させたか

もし`users`テーブルに直接貸出情報を持たせると、以下の問題が起きる。

- `users`テーブルは1人につき1行という構造のため、2回目・3回目と借りるたびに新しい貸出記録を追加する場所がない
- 返却後も履歴として残したいが、上書きすると過去の記録が消えてしまう

そのため、貸出という「出来事」を`loans`という別テーブルに切り出し、`user_id`・`asset_id`で紐付けることで、履歴が何度でも積み上がっていく形にした。

### なぜ`assets`に`department`カラムを持たせなかったか

実際の運用が「PCそのものの所属部署」ではなく「借りている人の部署」で検索する形だったため、`assets`側に部署情報を直接持たせず、`loans`テーブル経由で`users.department`を辿ることで間接的に部署を追える設計にした(正規化)。

```
assets(PC) --(loansで紐付け)--> users(借りた人) --- department(部署)
```

### 確定テーブル設計

**users**

| カラム名 | 型 | 説明 |
|---|---|---|
| id | BIGINT (PK) | |
| username | VARCHAR | ログインID(UNIQUE) |
| password | VARCHAR | ハッシュ化して保存 |
| role | VARCHAR | ADMIN / USER |
| department | VARCHAR | 部署 |

**assets**

| カラム名 | 型 | 説明 |
|---|---|---|
| id | BIGINT (PK) | |
| asset_code | VARCHAR (UNIQUE) | 資産番号 |
| name | VARCHAR | PC名・備品名 |
| category | VARCHAR | PC/モニタ/周辺機器など |
| status | VARCHAR | 在庫/貸出中/故障中 |
| purchased_date | DATE | 購入日 |

**loans**

| カラム名 | 型 | 説明 |
|---|---|---|
| id | BIGINT (PK) | |
| asset_id | BIGINT (FK → assets.id) | |
| user_id | BIGINT (FK → users.id) | |
| loaned_at | DATE | 貸出日 |
| due_date | DATE | 返却期限 |
| returned_at | DATE (NULL可) | 返却日。NULLなら貸出中 |
| status | VARCHAR | ACTIVE / OVERDUE / RETURNED |

### なぜ`asset_code`に`unique`制約をつけたか

`asset_code`は人間が画面上で見て資産を識別するための番号。重複していると見た目で区別できなくなり、操作ミス(例:間違った資産を返却済みにしてしまう)につながる。そのため一意性(unique)を持たせている。

---

## 2. 環境構築でつまずいたポイント

### 2-1. Packaging設定のミス(War → Jar)

**症状**
Spring Initializrで作成後、`ServletInitializer`という想定外のクラスが生成されていた。

**原因**
Packagingを「War」で作成していた。Warは外部Webサーバーへの後付けデプロイを前提とした形式で、今回のような単体起動アプリには「Jar」が適切。

**対処**
Packagingを「Jar」に選び直してプロジェクトを再作成した。

### 2-2. javaフォルダがソースルートとして未認識

**症状**
`entity`パッケージ配下で右クリックしても「Java クラス」が選択肢に出ず、「ファイル」しか作成できなかった。

**原因**
`src/main/java`フォルダがIntelliJに「ソースフォルダ」として正しく認識されていなかった(Project Structure上は登録されていたが反映されていない状態)。

**対処**
「File」→「Invalidate Caches...」→「破棄して再起動」でIntelliJのキャッシュを再構築し解決。

---

## 3. エンティティ実装でつまずいたポイント

### 3-1. Userエンティティ:クラスの二重定義

**修正前**

```java
public class User {

  @Entity
  @Table(name = "users")
  public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String Username;

    @Column(nullable = false)
    private String password;

    @Column(nullable = false)
    private String role;

    private String Department;
  }
}
```

**問題点**
- クラスの中にもう一つ同名の`class User`が入れ子になっていた(コピー時のミス)
- 変数名が大文字始まり(`Username`, `Department`)になっていた。Javaでは変数名は小文字始まり(キャメルケース)が慣習
- `department`に`@Column(nullable = false)`がついていなかった(部署は検索機能のmust要件があるため必須にすべき)

**修正後**

```java
package com.example.asset_management.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false)
    private String password;

    @Column(nullable = false)
    private String role;

    @Column(nullable = false)
    private String department;

    // getter/setterは省略
}
```

**設計判断メモ:なぜpasswordとroleにはunique = trueをつけないか**

`unique = true`は「その値を見ただけで1人の人を特定できる情報」につけるもの。

- `username`:ログインIDとして個人を一意に特定する必要があるためunique
- `password`:複数人が同じパスワードを設定できて当然のため、unique不可(むしろuniqueにすると「他人と同じパスワードを使えない」という不便な仕様になってしまう)
- `role`:ADMIN/USERは複数人が同じ値を持つのが前提のため、unique不可

なりすまし防止は「username + passwordの組み合わせ」で担保されており、unique制約ではなくハッシュ化などのセキュリティ対策で守るべき領域だと整理した。

### 3-2. Assetエンティティ:複数の設計ミス

**1回目(修正前)**

```java
package com.example.asset_management.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

@Entity
@Table(name = "assets")
public class Asset {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  @Column(nullable = false)
  private Long id;

  @Column(nullable = false)
  private String asset_code;

  @Column(nullable = false)
  private String username;

  @Column(nullable = false)
  private String category;

  @Column(nullable = false)
  private String status;

  @Column(nullable = false)
  private int purchase_dates;
}
```

**問題点**
- `id`に`@Column(nullable = false)`は不要(`@Id`の時点で自動的にNULL不可・重複不可になるため冗長)
- `asset_code`に`unique = true`がついていなかった(資産識別番号として一意である必要がある)
- `username`という不要なフィールドが混入していた(Userエンティティからのコピーミス。資産に借りている人の名前を直接持たせる設計ではない)
- `purchase_dates`が`int`型になっていた(日付なので`LocalDate`型が正しい)
- 変数名がスネークケース(`asset_code`)になっていた。Javaの慣習ではキャメルケース(`assetCode`)

**2回目(修正版、購入日以外を修正)**

```java
package com.example.asset_management.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

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

  @Column(nullable = false)
  private int purchaseDate;  // ← このタイミングではまだint型のまま
}
```

**最終版(purchaseDateをLocalDateに修正)**

```java
package com.example.asset_management.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.LocalDate;

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

  @Column(nullable = false)
  private LocalDate purchaseDate;

  // getter/setterは省略
}
```

---

## 4. MySQL接続でつまずいたポイント

### 4-1. Access denied エラー

**症状**

```
[28000][1045] Access denied for user 'root'@'localhost' (using password: YES)
```

**原因**
IntelliJのDatabase接続設定で入力したパスワードが誤っていた。

**対処**
ターミナルから`mysql -u root -p`で正しいパスワードを確認し、再入力して解決。

### 4-2. Unknown database エラー

**症状**

```
[42000][1049] Unknown database 'asset_management'.
```

**原因**
`asset_management`データベース自体をMySQL上にまだ作成していなかった。

**対処**

```sql
CREATE DATABASE asset_management;
```

をターミナルで実行して解決。

### 4-3. Spring Boot起動エラー:Failed to determine a suitable driver class

**症状**

```
Error creating bean with name 'dataSource' ...
Failed to instantiate [com.zaxxer.hikari.HikariDataSource]:
Factory method 'dataSource' threw exception with message:
Failed to determine a suitable driver class
```

**原因**
`application.properties`にMySQL接続設定を書き忘れており、Spring Initializrが自動生成した`spring.application.name`の1行しか存在していなかった。依存関係(MySQL Driver)自体は正しく入っていたが、接続先の情報がゼロだったためドライバを判断できなかった。

**対処**

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/asset_management
spring.datasource.username=root
spring.datasource.password=(パスワード)
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

を追記し、`Started AssetManagementApplication`のログが出て起動成功。DatabaseパネルでMySQLの`asset_management`スキーマ配下に`users`テーブルが自動生成されていることを確認した。

---

## 5. この開発ログから得られた学び

- テーブル設計は「なぜこのテーブルを分けたか」「なぜこの制約をつけたか」を1つずつ言語化することで、初めて自分の設計として説明できるようになる
- エラーメッセージは長く見えても、末尾のExceptionメッセージ(例:`Failed to determine a suitable driver class`)に原因のヒントが集約されていることが多い
- 環境構築段階のつまずき(War/Jar、ソースルート未認識)は地味だが、放置すると以降の作業全体がブロックされるため、早期に切り分けて解決することが重要
