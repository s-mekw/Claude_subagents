Feature: 多要素認証（MFA）機能
  As a セキュリティ意識の高いユーザーとして
  I want to 多要素認証（MFA）を有効化し、ログイン時にパスワードに加えてワンタイムパスワードを入力したい
  So that 万が一パスワードが漏洩した場合でも、第三者による不正ログインを防ぎ、アカウントを安全に保ちたい

  Background:
    Given ユーザー "test.user@example.com" が登録済みである
    And パスワード "SecurePass123!" でログインできる状態である
    And スマートフォンにGoogle Authenticatorアプリがインストール済みである

  @smoke @critical
  Scenario: MFAの有効化から無効化までの完全なフロー
    Given ユーザーがログイン済みである
    When アカウント設定画面を開く
    And セキュリティ設定タブを選択する
    And "多要素認証（MFA）を有効化" ボタンをクリックする
    Then QRコード画面が表示される
    And "Google Authenticatorでスキャンしてください" のガイダンスが表示される
    
    When スマートフォンでQRコードをスキャンする
    And 生成された6桁のワンタイムパスワード "123456" を入力する
    And "有効化" ボタンをクリックする
    Then "MFAが正常に有効化されました" の成功メッセージが表示される
    And MFA設定有効化の確認メールが "test.user@example.com" に送信される
    And MFAステータスが "有効" と表示される

    When 一度ログアウトする
    And メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    Then "ワンタイムパスワードを入力してください" の画面が表示される
    
    When Google Authenticatorで生成された6桁のOTP "789012" を入力する
    And "ログイン" ボタンをクリックする
    Then ログインが成功し、ダッシュボード画面が表示される

    When アカウント設定画面を開く
    And セキュリティ設定タブを選択する
    And "MFAを無効化" ボタンをクリックする
    And 現在のパスワード "SecurePass123!" を入力する
    And 確認ダイアログで "無効化する" を選択する
    Then "MFAが正常に無効化されました" の成功メッセージが表示される
    And MFA設定無効化の確認メールが "test.user@example.com" に送信される
    And MFAステータスが "無効" と表示される

  @regression
  Scenario: 無効なOTPでのログイン失敗
    Given MFAが有効化済みのユーザーがいる
    When メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    And 無効なOTP "000000" を入力する
    And "ログイン" ボタンをクリックする
    Then "ワンタイムパスワードが正しくありません" のエラーメッセージが表示される
    And ログイン画面に留まる
    And ログイン失敗のログが記録される

  @regression
  Scenario: 期限切れOTPでのログイン失敗
    Given MFAが有効化済みのユーザーがいる
    When メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    And 30秒以上前に生成された期限切れOTP "456789" を入力する
    And "ログイン" ボタンをクリックする
    Then "ワンタイムパスワードの有効期限が切れています" のエラーメッセージが表示される
    And ログイン画面に留まる

  @regression
  Scenario: 使用済みOTPでのログイン失敗
    Given MFAが有効化済みのユーザーがいる
    And 過去に使用済みのOTP "654321" がある
    When メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    And 使用済みのOTP "654321" を再度入力する
    And "ログイン" ボタンをクリックする
    Then "このワンタイムパスワードは既に使用されています" のエラーメッセージが表示される
    And ログイン画面に留まる

  @security @critical
  Scenario: OTP入力の試行回数制限
    Given MFAが有効化済みのユーザーがいる
    When メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    And 無効なOTP "111111" を入力して "ログイン" ボタンをクリックする
    And 無効なOTP "222222" を入力して "ログイン" ボタンをクリックする
    And 無効なOTP "333333" を入力して "ログイン" ボタンをクリックする
    And 無効なOTP "444444" を入力して "ログイン" ボタンをクリックする
    And 無効なOTP "555555" を入力して "ログイン" ボタンをクリックする
    Then "試行回数が上限に達しました。しばらく時間をおいてから再度お試しください" のメッセージが表示される
    And アカウントが15分間ロックされる
    And セキュリティアラートメールが送信される

  @edge-case
  Scenario: QRコードの再生成
    Given ユーザーがログイン済みである
    When アカウント設定画面を開く
    And セキュリティ設定タブを選択する
    And "多要素認証（MFA）を有効化" ボタンをクリックする
    Then QRコード画面が表示される
    
    When "QRコードを再生成" ボタンをクリックする
    Then 新しいQRコードが表示される
    And 以前のQRコードとは異なるコードが生成される
    And "新しいQRコードが生成されました" の通知が表示される

  @edge-case
  Scenario: 複数デバイスでのMFA設定
    Given ユーザーがログイン済みである
    And 既に1台のデバイスでMFAが有効化済みである
    When 別のデバイスでGoogle Authenticatorに同じアカウントを追加しようとする
    Then 両方のデバイスで同じOTPが生成される
    And どちらのデバイスのOTPでもログインできる

  @edge-case
  Scenario: MFA有効化中のネットワークエラー
    Given ユーザーがログイン済みである
    When アカウント設定画面を開く
    And セキュリティ設定タブを選択する
    And "多要素認証（MFA）を有効化" ボタンをクリックする
    And QRコードをスキャンして6桁のOTP "123456" を入力する
    And ネットワーク接続が切断された状態で "有効化" ボタンをクリックする
    Then "ネットワークエラーが発生しました。再度お試しください" のエラーメッセージが表示される
    And MFAの有効化が完了していない状態を保つ

  @usability
  Scenario: MFA設定時のガイダンス表示
    Given ユーザーがログイン済みである
    When アカウント設定画面を開く
    And セキュリティ設定タブを選択する
    Then "多要素認証を有効化すると、ログイン時の安全性が向上します" のガイダンスが表示される
    
    When "多要素認証（MFA）を有効化" ボタンをクリックする
    Then QRコード画面が表示される
    And "手順1: スマートフォンにGoogle Authenticatorアプリをインストール" が表示される
    And "手順2: アプリでQRコードをスキャン" が表示される
    And "手順3: 表示された6桁の数字を入力" が表示される
    And Google Authenticatorアプリのダウンロードリンクが表示される

  @usability
  Scenario: OTP入力フィールドの操作性
    Given MFAが有効化済みのユーザーがいる
    When メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    Then OTP入力フィールドが表示される
    And 入力フィールドにフォーカスが自動的に設定される
    And プレースホルダーに "6桁のコードを入力" が表示される
    And 数字以外の文字は入力できない
    And 6文字入力すると自動的にフォーカスが次のボタンに移動する

  @security
  Scenario: MFA無効化時の確認フロー
    Given MFAが有効化済みのユーザーがいる
    And ユーザーがログイン済みである
    When アカウント設定画面を開く
    And セキュリティ設定タブを選択する
    And "MFAを無効化" ボタンをクリックする
    Then "本当にMFAを無効化しますか？セキュリティが低下します" の警告ダイアログが表示される
    And "パスワードを再入力してください" の入力フィールドが表示される
    
    When 間違ったパスワード "WrongPassword" を入力する
    And "無効化する" ボタンをクリックする
    Then "パスワードが正しくありません" のエラーメッセージが表示される
    And MFAが有効のまま変更されない

  @security @critical
  Scenario: セッション管理とMFAの関係
    Given MFAが有効化済みのユーザーがいる
    When メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    And 有効なOTP "123456" を入力してログインに成功する
    And 8時間後にページを更新する
    Then セッションが期限切れとなりログイン画面にリダイレクトされる
    And 再度メールアドレスとパスワードでログインを試行する
    Then OTP入力画面が再度表示される

  @regression
  Scenario Outline: 異なるOTP形式での入力検証
    Given MFAが有効化済みのユーザーがいる
    When メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    And OTP入力フィールドに "<input>" を入力する
    And "ログイン" ボタンをクリックする
    Then "<expected_message>" が表示される

    Examples:
      | input      | expected_message                          |
      | 12345      | 6桁の数字を入力してください                   |
      | 1234567    | 6桁の数字を入力してください                   |
      | abc123     | 数字のみ入力可能です                         |
      | 12 34 56   | スペースは使用できません                      |
      |            | ワンタイムパスワードを入力してください          |

  @notification
  Scenario: MFA設定変更時の通知メール内容
    Given ユーザーがログイン済みである
    When MFAを有効化する
    Then 確認メールが "test.user@example.com" に送信される
    And メールの件名が "【Chatwork】多要素認証が有効化されました" である
    And メールに有効化日時が記載されている
    And メールに使用したブラウザとIPアドレスが記載されている
    And メールに"心当たりがない場合はすぐにパスワードを変更してください"の注意書きがある

  @accessibility
  Scenario: MFA設定画面のアクセシビリティ
    Given ユーザーがログイン済みである
    When アカウント設定画面を開く
    And セキュリティ設定タブを選択する
    Then "多要素認証（MFA）を有効化" ボタンにaria-labelが設定されている
    And QRコード画像にalt属性が設定されている
    And エラーメッセージがスクリーンリーダーで読み上げ可能である
    And キーボードのみで全ての操作が完了できる

  @performance
  Scenario: OTP検証のレスポンス時間
    Given MFAが有効化済みのユーザーがいる
    When メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    And 有効なOTP "123456" を入力する
    And "ログイン" ボタンをクリックした時刻を記録する
    Then OTP検証とログイン完了までの時間が3秒以内である
    And ダッシュボード画面の読み込みが完了する

  @backup-codes
  Scenario: バックアップコードの生成と使用
    Given ユーザーがログイン済みである
    When MFAを有効化する
    And "バックアップコードを生成" ボタンをクリックする
    Then 10個のバックアップコード（各8桁）が表示される
    And "これらのコードを安全な場所に保存してください" の注意書きが表示される
    And "印刷" ボタンが表示される
    
    When 一度ログアウトする
    And メールアドレス "test.user@example.com" とパスワード "SecurePass123!" でログインを試行する
    And "バックアップコードを使用" リンクをクリックする
    And 生成されたバックアップコード "ABCD1234" を入力する
    And "ログイン" ボタンをクリックする
    Then ログインが成功する
    And 使用したバックアップコードが無効化される
    And "バックアップコードを1つ使用しました。残り9個です" の通知が表示される
