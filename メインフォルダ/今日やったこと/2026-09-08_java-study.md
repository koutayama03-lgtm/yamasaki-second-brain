---
tags: [java, weekend-study]
date: 2026-09-08
---

# 2026-09-08 Java学習メモ

## 1. Proxy（プロキシ）とは
- 本来呼び出したい処理の代わりに、間に入って処理をする「代理人オブジェクト」
- 種類
  - **静的プロキシ**：手書きでプロキシクラスを作る（例：`HogeHogeProxy.java`のような命名規則が典型）
  - **動的プロキシ**：`java.lang.reflect.Proxy` + `InvocationHandler` で実行時に自動生成
- Spring等のフレームワークでは `@Transactional` や AOP がプロキシで実現されている
  → 「メソッドを呼んでいるのにトランザクションが効かない」等のバグの原因になりうる

## 2. RPC（Remote Procedure Call）とProxyの関係
- クライアント側の「スタブ（プロキシ）」がリモートのメソッド呼び出しをローカル呼び出しのように見せる仕組み
- 内部でやっていること：引数のシリアライズ → ネットワーク送信 → サーバー側で実行 → 結果のデシリアライズ
- Javaの古典例：RMI（`java.rmi`）も動的プロキシを利用

## 3. RowSet
- `ResultSet` を拡張した、JavaBeans仕様準拠のデータ格納インターフェース
- `ResultSet`の弱点：コネクションを開いたままでないと使えない／シリアライズできない
- 種類
  - `ConnectedRowSet`（例：`JdbcRowSet`）：接続を維持
  - `DisconnectedRowSet`（例：`CachedRowSet`）：**接続を切ってもデータを保持・編集可能**（`acceptChanges()`で反映）
- 「接続を張りっぱなしにしたくない」設計の古いアプリでよく使われる

## 4. abstract（抽象）
- クラスにも**メソッド**にも付けられる
  - クラス → インスタンス化不可、継承前提
  - メソッド → 中身なし、サブクラスに実装を強制

## 5. 継承（extends）とインターフェース実装（implements）
- クラスの継承：`extends`（**単一継承のみ**）
- インターフェースの実装：`implements`（**複数可、カンマ区切り**）
- 両方同時に使える：`class Dog extends Animal implements Runnable`

### 継承・実装は多段階でチェーンする
```java
interface Animal { void cry(); }
abstract class Mammal implements Animal { }  // ここには実装なくてOK
class Dog extends Mammal {
    @Override
    void cry() { System.out.println("ワン"); }
}
```
- `Dog.java`だけを見ても`implements Animal`が書かれていないことがある
  → 親クラス（`Mammal`）まで遡って`implements`を探す必要がある
- IDEの「継承階層を表示」機能を使うと早い

### import と implements は別物
- `import`：クラスの所在地をコンパイラに教えるだけ（継承・実装とは無関係）
- `implements`：インターフェースの実装を宣言する、振る舞いに関わるもの

## 6. Override（オーバーライド）と Overload（オーバーロード）
- **Override**：親／インターフェースの同じシグネチャ（メソッド名・引数が同じ）のメソッドを子クラスで書き換える
- **Overload**：同じクラス内で、同じメソッド名だが引数の型・数が違うメソッドを複数定義する
- クラスが違えば、同じメソッド名でも無関係な別メソッドとして共存できる（メソッド名の衝突は普通に許される）

## 7. 実践ケーススタディ：hoge/huga/piyo問題
**状況**
- `Hoge`（インターフェース）に抽象メソッド `doHoge()`（引数なし）
- `Huga`（クラス）が `Piyo` を`extends`
- `Huga`に`doHoge()`（引数なし）が実装されているが、`@Override`なし
- `Piyo`にも`Huga`にも`implements Hoge`は無い

**結論**
→ `Huga.doHoge()`と`Hoge.doHoge()`は**無関係な別メソッド**（名前が同じだけ）

**判断根拠**
1. 継承チェーン（`Huga → Piyo`）のどこにも`implements Hoge`が存在しない
2. `@Override`が付いていない
   - 本当にオーバーライドしているなら`@Override`を付けてコンパイルが通るはず
   - 試しに付けてエラーが出れば「無関係」の確定的な証拠になる

**教訓**
- メソッド名が同じでも、継承・実装のチェーンが繋がっていなければ無関係
- `@Override`の有無は、意図した実装かどうかを判断する重要な手がかりになる
- 命名の偶然の一致か、あるいは実装し忘れのバグかは、コードの背景を知る人に確認する必要がある
