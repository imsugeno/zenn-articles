# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## リポジトリ概要

Zenn CLIで管理する技術記事・書籍のコンテンツリポジトリ。GitHub連携によりmasterブランチへのpushでZennに自動デプロイされる。

## 構成

- `articles/` - 記事のmarkdownファイル（ファイル名がスラッグになる）
- `books/` - 書籍のmarkdownファイル

## 記事のフロントマター形式

```yaml
---
title: "記事タイトル"
emoji: "🐕"
type: "tech"  # tech: 技術記事 / idea: アイデア
topics: ["topic1", "topic2"]  # タグ（最大5つ）
published: true  # true で公開、false で下書き
publication_name: "org_name"  # Organization投稿時のみ
---
```

## コマンド

```bash
npx zenn preview        # ローカルプレビュー（http://localhost:8000）
npx zenn new:article    # 新規記事の作成
npx zenn new:book       # 新規書籍の作成
```

## 注意事項

- 日本語で記事を執筆する
- masterブランチへのpushが本番デプロイとなるため、公開前の記事は `published: false` にしておく
