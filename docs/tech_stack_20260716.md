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
- データ管理: リアルタイムな状態遷移はFirebase（Firestore）に一任し、お題カードのキャッシュやユーザー設定（音量など）の永続化にのみ`AsyncStorage`を使用する。

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

## 10. お題カードのコンテンツ管理
- **将来対応: Firestoreの `promptCards` コレクションでリモート追加・更新を可能にする**
  - オンライン起動時にリモートの最新リストを取得してローカルにキャッシュ(`AsyncStorage`)
  - アプリアップデートなしでお題を追加・差し替えできる運用にする
- カードのデータ構造(例):
  ```ts
  type PromptCard = {
    id: string;
    text: string;          // 例: "オペラの一節"
    category: "opera" | "poem" | "speech" | ...;
    difficulty: 1 | 2 | 3;
    isSafeForAllAges: boolean; // NGワード/際どい内容の除外フラグ
  };
  ```

## 11. タイムアウト・強制公開の仕様
- 各ラウンドの評価入力に**制限時間(デフォルト60秒、設定で30〜120秒に変更可)**を設ける
- 制限時間終了時、未提出プレイヤーの扱いはホストが事前に選択:
  - **自動投入(デフォルト)**: 未提出者には一律デフォルト値(下手さ3・愛おしさ3)を自動投入して公開へ進む
  - **除外**: 未提出者はそのラウンドの集計から除外(得点計算の母数からも外す)
- ホストは制限時間を待たずに**「強制公開」ボタン**でいつでも公開へ進められる(全員急いで入力させたくない場合の救済)
- 通信不安定への対応(オンラインモードのみ):
  - Firestoreのオフライン永続化(`enableIndexedDbPersistence` 相当)を有効化し、一時切断中の入力はローカルにキューイング、再接続時に自動再送
  - 再接続できないままタイムアウトを迎えたプレイヤーは上記の未提出扱いルールに従う

## 12. 演出(アニメーション・効果音)
- **要否: 実装する。** 「矛盾した評価が同時に成立する」瞬間の盛り上がりがゲームの核なので、演出への投資対効果が高いと判断
- 公開シーケンス(案):
  1. カウントダウン(3・2・1)
  2. 全員のカードが同時にめくれる演出(スコア数値がフリップ/回転して表示)
  3. 「下手さ × 愛おしさ」の積が観客ごとにカウントアップ→合計値が加算されるスコアアニメーション
  4. 高得点(閾値超え)時は追加のファンファーレ演出
- 効果音: ドラムロール(集計中)、公開時のジングル、高得点時のファンファーレ。BGMは常時ではなくラウンド区切りのみ
- 設定: 音声・アニメーションはそれぞれオン/オフ切替可能(会場の状況やアクセシビリティ配慮)
- 実装ライブラリ: `expo-av`(効果音再生)、`react-native-reanimated` + `lottie-react-native`(アニメーション)

## 13. 今後の検討事項
- お題カードの多言語対応(日本語のみか、英語版も出すか)
- リモートお題カードのモデレーション・投稿フロー(ユーザー投稿を受け付けるかどうか)

