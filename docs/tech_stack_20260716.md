---
generated_at: 2026-07-16
based_on: idea_20260716_005412.md
target_platforms: [iOS, Android]
---

# 技術スタック: アンビバレント・オペラ アプリ版

## 1. 前提・要件整理
- 対応OS: iOS / Android(クロスプラットフォーム、単一コードベース希望)
- ゲームの性質:
  - 1ラウンドにつき1人がパフォーマー役、残りが観客役
  - 観客全員が「下手さ」「愛おしさ」(各1〜5点)を**同時に・伏せて**入力する必要がある
  - → 各プレイヤーが自分のスマホを持ち、同一ルームに接続するリアルタイム多人数構成が前提になる(Jackbox Party Pack方式に近い)
- プレイ人数: 4〜8人

## 2. アーキテクチャ概要
- ルームベースのリアルタイムマルチプレイヤー
  1. 誰か1人がルームを作成 → ルームコード(4桁英数字等)が発行される
  2. 他プレイヤーはルームコードを入力して入室、ニックネーム登録
  3. ホスト(または進行役)がラウンドを開始 → 現在のパフォーマーとお題が全員の画面に表示
  4. 観客役の画面では「下手さ」「愛おしさ」スコア入力UIを表示、送信するとサーバー側で「提出済み」状態のみ他者に見える(値は隠す)
  5. 全員の提出が揃ったら一斉公開・積の合計を計算・演出表示
  6. 全員がパフォーマーを一巡したら総合得点を表示して終了

## 3. フロントエンド(モバイルアプリ)
- **フレームワーク: React Native + Expo**
  - 理由: 単一コードベースでiOS/Android両対応。Expoにより実機テスト・ビルド(EAS Build)・配布が容易で、個人〜小規模開発に向く
  - 言語: TypeScript
  - ナビゲーション: React Navigation
  - 状態管理: Zustand(小規模アプリのため軽量なもので十分)
  - UI: NativeWind(Tailwind系) or Tamagui など、パーティーゲームらしいポップなUIを作りやすいライブラリ
- 代替案: Flutter(Dart) — アニメーションやUI表現力を優先する場合の選択肢。ただしTypeScript統一を重視するならReact Native優位

## 4. バックエンド / リアルタイム通信
スコアの同時提出・一斉公開・ルーム管理にリアルタイム同期が必須。

- **候補A: Firebase(Firestore + Realtime Database + Auth)**
  - サーバーレスで運用負荷が低く、小規模パーティーゲームの無料枠で十分まかなえる
  - Firestoreのリアルタイムリスナーでルーム状態・提出状況を同期
  - Firebase Anonymous Authでアカウント登録不要のニックネームプレイを実現
- **候補B: Supabase(PostgreSQL + Realtime + Auth)**
  - OSS志向、SQLベースでデータ構造の見通しが良い。Realtime機能でFirebaseと同等のことが可能
- 推奨: まずは **Firebase** で着手(ドキュメント・事例が多く立ち上げが速い)

## 5. 認証
- 匿名認証(Firebase Anonymous Auth)+ ニックネーム入力のみ。アカウント登録不要でパーティーゲームの「その場で始める」体験を優先

## 6. 配布・ビルド
- **Expo Application Services (EAS Build)** でiOS(.ipa)/Android(.apk・.aab)を単一設定からビルド
- ストア公開:
  - Apple App Store: Apple Developer Program登録(年額 $99)が必要
  - Google Play Store: Google Play Developer登録(一回 $25)が必要
- 開発中の配布・テスト:
  - Expo Go または EAS Development Build(実機デバッグ)
  - TestFlight(iOS)/ Google Play内部テスト(Android)でベータ配布

## 7. 開発ツール・CI
- 言語: TypeScript
- パッケージ管理: pnpm
- Lint/Format: ESLint + Prettier
- テスト: Jest + React Native Testing Library
- CI/CD: GitHub Actions(lint・test自動実行)+ EAS Buildのクラウドビルド連携

## 8. 想定ディレクトリ構成(例)
```
app/
  (screens)/
    Home.tsx
    CreateRoom.tsx
    JoinRoom.tsx
    Lobby.tsx
    PerformerView.tsx
    AudienceScoreInput.tsx
    RoundResult.tsx
    FinalResult.tsx
  components/
  stores/          # Zustand
  services/
    firebase.ts
    roomService.ts
    scoreService.ts
  types/
docs/
  idea_20260716_005412.md
  tech_stack_20260716.md
```

