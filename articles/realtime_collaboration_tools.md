---
title: "リアルタイムコラボレーションツールまとめ"
emoji: "👫"
type: "idea"
topics: ["OT","CRDT","Tools"]
published: true
---

# リアルタイムコラボレーションツールまとめ

WIP; コントリビューション歓迎

### まずレベル定義

1. 排他 (悲観ロック;楽観ロック)
2. 準同期 (相手の編集結果は見えないが、取り込み操作が可能)
3. 共同編集 (相手の編集結果が見える)
4. 同時編集 (相手の存在と編集中状態が見える)

### 歴史

1. OT; Operational Transformation; 1980頃
2. CRDT; Conflict-free Replicated Data Type; 2011年

# ツール

## Office

- Microsoft 365 w/Sharepoint (Word, Excel, Powerpoint)
- Notion
- Google Workspace (Document, Spreadsheet, Slide)

## ホワイトボード

- miro (realtime board) (ホワイトボード) 有料プラン
- FigJam (ホワイトボード) 有料プラン
- Lucidspark (ホワイトボード) 3枚まで無料

## UIデザインツール/ダイアグラム

- draw.io (diagrams.net) xml形式
- Figma (UIデザインツール)
- AdobeXD (UIデザインツール)
- coggle (マインドマップ)

## テキスト

- Scrapbox (アウトライナー/タグ型)
- HackMD/HudgeDOC/CodiMD (markdown)
- Dynalist (アウトライナー/階層型)
- workflowy (アウトライナー/階層型)
- simplenote (テキストエディタ)
- Overleaf (TeXエディタ)
- BOX notes (リッチテキスト)
- Dropbox Paper
- evernote

## IDE

- VS Code + Live share
- JupyterLab
- cloud9 --collab
- ATOM Teletype

## ライブラリ/部品

- yjs
- ot.js
- CKEditor

## データストア

### (CAP定理; Consistency/Availability/Partition-tolerance; 一貫性/可用性/分断耐性 のうちAとPを実現しているもの)

- AWS DynamoDB (KVS)
- Apache Cassandra (KVS;ASL2)
- riak/CRDT (Leaderless KVS;ASL2)
- roshi (time-series event storage;BSD2 by soundcloud)
- Akamai/EdgeKV
- Akka/Distributed Data (EIP用ツールキット)
- Open/R/KvStore (分散アプリ用ルーティングライブラリ; MIT by facebook)
- SQLite+CRDT拡張

### 違うもの

- Cloudflare R2 (オブジェクトストレージ;一貫性)
- Cloudflare D1 (RDB;)
- Google Spanner (RDB;2PC)
