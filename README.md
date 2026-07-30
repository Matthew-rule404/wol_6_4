# wol_6_4

IPv6 からは Web UI、IPv4 専用の管理端末からは ConoHa などを経由する制限済み
ProxyJump CLI を使い、宅内 Ubuntu サーバーで Wake on LAN を安全に実行する
アプリです。宅内への IPv6 着信が使えない場合は、ConoHa を IPv4 / IPv6 Web
入口にする代替構成も設計対象に含みます。

現在は設計段階です。稼働する Web アプリや WoL 実行スクリプトはまだ実装して
いません。

## 目標

- Web ブラウザーと公開入口の通信を TLS 1.3 のみに限定する
- CSV に登録された許可済みホストだけを起動対象にする
- PING、SSH、推定電源状態、最終確認時刻を Web 画面で確認する
- IPv6 では宅内 WoL サーバーへ直接 HTTPS 接続できるようにする
- IPv4 専用の管理端末では ConoHa などを ProxyJump とした制限 CLI を利用する
- Magic Packet は対象 PC と同じ宅内ネットワーク上の WoL サーバーから送信する

## 設計文書

- [アーキテクチャ図](docs/00_ARCHITECTURE_DIAGRAM.md)
- [アーキテクチャ](docs/01_ARCHITECTURE.md)
- [API とデータ設計](docs/02_API_AND_DATA.md)
- [セキュリティ設計](docs/03_SECURITY.md)
- [開発計画](docs/04_DEVELOPMENT_PLAN.md)
- [用語集](docs/99_GLOSSARY.md)

## 重要な前提

ConoHa 上で `wakeonlan` を実行しても、宅内ネットワークのブロードキャストへは
通常届きません。踏み台は SSH 接続の中継に限定し、次のコマンドは宅内 Ubuntu
WoL サーバー上で実行します。

```bash
wakeonlan 38:05:25:38:F0:C9
```
