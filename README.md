# Laravel テンプレートプロジェクト

このプロジェクトは Laravel + Sail を使用した開発環境テンプレートです。

## セットアップ

### 1. 開発環境の起動
```bash
./vendor/bin/sail up
```
Docker環境を起動します。初回起動時は時間がかかる場合があります。

### 2. データベースマイグレーション
```bash
./vendor/bin/sail artisan migrate
```
データベースのテーブルを作成します。初回セットアップ時に必要です。

## 開発コマンド

### コードフォーマッター（PHP CS Fixer / Pint）

#### 自動フォーマット実行
```bash
./vendor/bin/sail php ./vendor/bin/pint
```
すべてのPHPファイルを Laravel のコーディング規約に従って自動フォーマットします。

#### フォーマットチェック（テストモード）
```bash
./vendor/bin/sail php ./vendor/bin/pint --test
```
ファイルを変更せずに、フォーマットが必要な箇所を確認します。CIで使用することが多いです。

#### 変更ファイルのみフォーマット
```bash
./vendor/bin/sail php ./vendor/bin/pint --dirty
```
Gitで変更されたファイルのみをフォーマットします。効率的な開発に便利です。

## その他の便利なコマンド

### アプリケーション関連
```bash
# キャッシュクリア
./vendor/bin/sail artisan cache:clear

# ルート一覧表示
./vendor/bin/sail artisan route:list

# Tinkerコンソール起動
./vendor/bin/sail artisan tinker
```

### テスト実行
```bash
# 全テスト実行
./vendor/bin/sail test

# 特定のテストファイル実行
./vendor/bin/sail test tests/Feature/ExampleTest.php
```
