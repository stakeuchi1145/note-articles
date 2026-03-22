---
title: "無人型カーシェアアプリ NeoDriver を Kotlin Multiplatform + Jetpack Compose で作った"
emoji: "🚗"
type: "tech"
topics: ["Android", "KotlinMultiplatform", "JetpackCompose", "MVVM", "個人開発"]
published: false
---

今日は個人プロジェクト **NeoDriver** のAndroidアプリ開発をガッツリ進めました。無人型のレンタカー・カーシェアサービスを想定したアプリで、Kotlin Multiplatform (KMP) と Jetpack Compose を組み合わせて作っています。

---

## NeoDriver とはどんなアプリか

「無人で借りられるカーシェア」というコンセプトで作っています。鍵の受け渡しなしに、スマホのBluetoothで車を解錠して乗り出せるイメージです。

主な機能は以下の通りです。

- 近くの車両を探してブラウズする
- 日時・保護プランを選んで予約する
- 予約の確認・変更・キャンセル
- Bluetooth/IoT経由での車両アンロック（実装予定）

---

## 画面紹介

### ホーム画面

![ホーム画面](/images/neodriverapp/neodriver_home.png)

ログイン後に表示されるメイン画面です。アクティブな予約がある場合は上部にカードで表示されます。「AVAILABLE NEARBY」セクションでは近くの空き車両を横スクロールで確認できます。

### 予約詳細画面

![予約詳細画面](/images/neodriverapp/neodriver_reservation.png)

車両情報・レンタル期間・保護プランの選択・料金サマリーを一画面にまとめています。Basic / Standard / Elite のプランを選ぶと料金がリアルタイムで更新されます。

---

## 今日やったこと

### ホーム画面の実装

ログイン後に飛ぶホーム画面を一から作りました。「近くの車両」と「おすすめ車両」の2セクション構成で、車両カードをタップすると予約画面へ遷移する動線も実装しています。

車両カードは途中でデザインを作り直しました。最初は「航続距離」「バッテリー残量」を表示していたのですが、ホーム画面では「料金」と「場所」の方が直感的と判断して変更しています。

### ログイン情報の保持とログアウト

認証トークンを **Multiplatform Settings** に保存してセッションを維持するようにしました。KMPのライブラリなのでAndroid・iOS両方で同じコードが使えます。

ログアウト時はトークンをクリアしてログイン画面に戻ります。ViewModelのStateFlowを介してナビゲーションをトリガーする部分が地味に悩みどころでした。

```kotlin
fun logout() {
    viewModelScope.launch {
        authRepository.clearToken()
        _uiState.update { it.copy(isLoggedOut = true) }
    }
}
```

### UI/UX の細かい改善

画面遷移やボタン操作にアニメーションとハプティクスを追加しました。地味な変更ですが、実機で触ったときの体感がかなり変わります。セーフエリア対応も `WindowInsets` で実施しています。

### Bluetooth権限の追加

Android 12以降は `BLUETOOTH_SCAN` と `BLUETOOTH_CONNECT` を個別に宣言する必要があります。実装中に漏れに気づいたのでまとめて追加しました。

```xml
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

### Android Gradle Plugin を 9.0.1 へアップグレード

Gradle周りのバージョンを上げ、出ていた警告も解消しています。

---

## アーキテクチャ

MVVM + クリーンアーキテクチャで構成しています。

```
UI層 (Jetpack Compose)
  ↓
ViewModel (StateFlow)
  ↓
UseCase (ビジネスロジック)
  ↓
Repository
  ↓
DataSource (モックデータ / SQLDelight)
```

`shared` モジュールにViewModelとUseCaseをまとめているため、将来iOSにも同じロジックを流用できる構成です。データベースには **SQLDelight** を採用しています。型安全なSQLが書けてKMPとの相性がいいのが気に入っています。

---

## 今日の感想

Jetpack Compose は StateFlow との組み合わせが素直で、状態変化の追跡がしやすいと感じています。KMPはViewModel層まで共通化していますが、Android固有の処理が混ざりそうになる場面があり、共通化の境界線をどこに引くかは引き続き考えどころです。

---

## 次にやること

- Bluetooth経由での車両アンロック実装
- Google Maps APIを使った地図表示
- モックデータから実APIへの切り替え
- iOSアプリの対応
