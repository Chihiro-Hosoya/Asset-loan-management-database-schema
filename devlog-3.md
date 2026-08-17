# 資産管理リマインドアプリ 開発ログ #3(ログイン画面実装・動作確認編)

前回の開発ログでは、Repository層の実装とSpring Securityによる認証機能(CustomUserDetails / CustomUserDetailsService / SecurityConfig)の実装過程をまとめた。今回はログイン画面(Thymeleaf)の実装と、実際のログイン動作確認までの過程を記録する。今回は特にトラブルシューティングの比重が大きく、認証機能に関わるほぼ全ての層でエラーに直面した。

---

## 1. Controllerの実装

ログイン画面(`/login`)へのアクセスを受け付け、対応するHTMLを表示するためのControllerを作成した。

```java
package com.example.asset_management.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

/**
 * ログイン画面の表示を担当するコントローラー。
 */
@Controller
public class AuthController {

    /**
     * "/login" にGETリクエストが来た時、login.htmlを表示する。
     */
    @GetMapping("/login")
    public String loginPage() {
        return "login";
    }
}
```

`@Controller`はSpring Bootに「画面表示を担当するクラス」であることを伝えるアノテーション。`return "login";`という文字列を返すことで、Spring Bootが自動的に`templates/login.html`を探して表示する仕組みになっている。

## 2. ログイン画面(login.html)の実装

`src/main/resources/templates/login.html`に、Thymeleafを使ったログインフォームを実装した。

```html
<!DOCTYPE html>
<html lang="ja" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>ログイン | 資産管理アプリ</title>
</head>
<body>
<h1>ログイン</h1>

<div th:if="${param.error}">
    <p style="color:red;">ユーザー名またはパスワードが正しくありません。</p>
</div>

<form th:action="@{/login}" method="post">
    <div>
        <label for="username">ユーザー名:</label>
        <input type="text" id="username" name="username">
    </div>
    <div>
        <label for="password">パスワード:</label>
        <input type="password" id="password" name="password">
        <button type="submit">ログイン</button>
    </div>
</form>
</body>
</html>
```

**設計のポイント**

- `input`の`name`属性を`username` / `password`にすることで、Spring Securityが自動的にログイン処理として認識する(Controller側でログイン処理自体を書く必要がない)
- `th:if="${param.error}"`で、ログイン失敗時にSpring Securityが自動付与する`error`パラメータを検知し、エラーメッセージを出し分けている

---

## 3. トラブルシューティング

### 3-1. SecurityConfigの構文重複エラー

- **症状**: `変数 formはすでにメソッド filterChain(HttpSecurity)で定義されています`
- **原因**: `SecurityConfig.java`のコード整形時、`.formLogin(form -> form` の行が誤って2回連続で記述されており、閉じ括弧の数も合わなくなっていた
- **対処**: 重複した行を削除し、括弧の対応関係を1つずつ確認しながら修正

### 3-2. CustomUserDetailsServiceの一連の文法エラー

`Optional<User>`から`orElseThrow`を使って値を取り出す処理の実装中、複数の文法エラーが連鎖して発生した。

- `User user = UserRepository.findByUsername(username);` → クラス名(大文字始まり)を、インスタンス変数のように直接呼び出そうとしていた。正しくはコンストラクタで受け取った`userRepository`(小文字)を使う必要があった
- `.orElseThrow(...)`の前で誤って文が`;`で終わってしまい、メソッドチェーンが分断されていた
- `.orElseThrow() -> new UsernameNotFoundException(...)` のように、ラムダ式`() -> ...`が`orElseThrow`のカッコの外に出てしまっていた
- エラーメッセージの文字列の中に`+ username`ごと含めてしまい、変数ではなく文字列として扱われていた

これらが積み重なり、一時は`org.apache.catalina.User`という無関係なTomcatのクラスへの型不一致エラーにまで発展した。最終的にコードを一から書き写すことで解決した。

**最終的な正しい実装**

```java
@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("ユーザーが見つかりません: " + username));

    return new CustomUserDetails(user);
}
```

### 3-3. 無限リダイレクト(ERR_TOO_MANY_REDIRECTS)

**症状**: `/login`にアクセスすると、ブラウザで「リダイレクトが繰り返し行われました」というエラーが発生。シークレットウィンドウでも再現したため、Cookieが原因ではないことを確認。

**原因調査の過程**

1. `application.properties`に`logging.level.org.springframework.security=DEBUG`を追加し、Spring Securityの詳細ログを出力
2. ログを確認したところ、実際には`/login`ではなく`/error`へのアクセスが記録されていた
3. `/error`は`permitAll()`の対象外だったため、「ログインが必要」と判定されて`/login`に戻される
4. しかし`/login`ページの表示処理自体でもエラーが起きており、再び`/error`に転送される、という無限ループになっていた
5. コンソールをさらに調べたところ、`org.thymeleaf.TemplateEngine`のエラーを発見。原因は`login.html`内の`th:if="${param.error"`で、閉じ括弧`}`が抜けていたこと(`"..."`の外側に`}`が飛び出してしまっていた)
6. `th:if="${param.error}"`に修正し、一旦解消

**原因1: Thymeleafの構文エラー**
`${ }`(Thymeleafの式)と`" "`(HTML属性の引用符)が入れ子になっている構文で、`}`の位置がずれていたことが根本原因だった。

### 3-4. 修正後も無限リダイレクトが再発

Thymeleafのエラーを直したにも関わらず、同じ`ERR_TOO_MANY_REDIRECTS`が再発。再度DEBUGログを確認したところ、起動ログの中に以下の記述を発見した。

```
Using generated security password: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Global AuthenticationManager configured with UserDetailsService bean with name inMemoryUserDetailsManager
```

**原因2: CustomUserDetailsServiceがBeanとして認識されていなかった**

自作の`CustomUserDetailsService`が正しくSpring Bootにスキャンされておらず、代わりにSpring Bootの開発用デフォルト機能(仮のログインユーザーを自動生成する機能)が起動してしまっていた。コード自体(`@Service`アノテーション、パッケージ宣言)に問題は見当たらなかったため、**ビルドキャッシュが古い状態のまま使われていた可能性**が高いと判断。

**対処**: ターミナルから以下を実行し、クリーンビルドを行った。

```bash
./gradlew clean build -x test
```

再ビルド後、起動ログの表示が

```
Global AuthenticationManager configured with UserDetailsService bean with name customUserDetailsService
```

に変わったことを確認し、自作のサービスが正しく認識されるようになったことを確認した。

### 3-5. パスワード未ハッシュ化による認証失敗

`CustomUserDetailsService`が正しく動くようになった後、ログイン画面自体は表示されたが、正しいユーザー名・パスワードを入力しても「ユーザー名またはパスワードが正しくありません」と表示された。

**原因**: 動作確認用に作成した`TestDataRunner`で、パスワードをハッシュ化せず平文のまま保存していた。ログイン時はSpring Securityが`BCryptPasswordEncoder`でハッシュ化した値と比較するため、平文で保存されたパスワードとは一致しない。

**対処**: `TestDataRunner`に`PasswordEncoder`をコンストラクタで受け取り、保存時にハッシュ化するよう修正した。

```java
@Component
public class TestDataRunner implements CommandLineRunner {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public TestDataRunner(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public void run(String... args) throws Exception {
        User user = new User();
        user.setUsername("test_tanaka2");
        user.setPassword(passwordEncoder.encode("password123"));
        user.setRole("USER");
        user.setDepartment("経理部");

        userRepository.save(user);
    }
}
```

### 3-6. 重複データによる起動エラー

`TestDataRunner`は`@Component`かつ`CommandLineRunner`のため、アプリ起動のたびに自動実行される。1回目の起動でユーザー保存に成功した後、2回目の起動でも同じユーザーを保存しようとして、以下のエラーが発生した。

```
Duplicate entry 'test_tanaka2' for key 'users.UKr43af9ap4edm43mmtq01oddj6'
```

これは`username`に設定していた`UNIQUE`制約が正しく機能している証拠でもある。動作確認用のデータは既に保存済みだったため、`TestDataRunner`は削除して対応した。

---

## 4. ログイン成功の確認

`test_tanaka2` / `password123`でログインを実行したところ、以下のエラーページが表示された。

```
There was an unexpected error (type=Not Found, status=404).
No static resource for request '/'.
```

これは`SecurityConfig`で設定した`defaultSuccessUrl("/", true)`が正しく機能し、ログイン成功後にトップページ(`/`)へリダイレクトされた結果である。`/`に対応する画面をまだ実装していないためのエラーであり、**認証処理自体は正常に完了している**ことを意味する。

---

## 5. 振り返り

今回は、無限リダイレクトという一つの症状の裏に、以下のように複数の異なる原因が重なっていた。

1. Thymeleafテンプレートの構文エラー(`}`の位置)
2. ビルドキャッシュにより自作のBeanが認識されていなかった
3. パスワードの未ハッシュ化

一見同じ症状(無限リダイレクト、ログイン失敗)でも、Spring Securityのデバッグログを有効にし、`/error`への転送やThymeleafの例外メッセージまで遡って確認することで、一つひとつ原因を切り分けることができた。特に「起動ログに残る一見無害な情報(generated security passwordの表示)」が、実は重大な設定ミスのシグナルだったという点は、今後同様の問題に直面した際に役立つ知見となった。

---

## 6. 現時点の進捗状況

| 項目 | 状態 |
|---|---|
| 要件定義・DB設計 | 完了 |
| エンティティ実装(User / Asset / Loan) | 完了 |
| Repository層 | 完了 |
| Spring Security(認証の仕組み) | 完了 |
| ログイン画面(HTML)・動作確認 | 完了 |
| トップページ・CRUD機能(資産登録・貸出・返却) | 未着手 |
| リマインド機能 | 未着手 |

*次回はトップページの実装と、資産管理のCRUD機能の実装に着手する予定。*
