# WoL Web アプリ アーキテクチャ図

## 1. 統合構成

Phase 5A の宅内 IPv6 直接公開、Phase 5B の ConoHa HTTPS 入口、IPv4
ProxyJump 管理経路を 1 枚に統合する。
図中には代替経路を併記しているが、Phase 5A と Phase 5B は宅内 IPv6 の
着信可否に応じて配備時にいずれか一方を選択する。
図は利用者から対象 PC まで、上から下へ進む順序で配置する。

```mermaid
flowchart TB
    subgraph Clients["利用者"]
        direction LR
        Browser["Web Browser<br/>IPv4 / IPv6"]
        Operator["管理端末<br/>IPv4"]
    end

    subgraph Cloud["ConoHa などの dual-stack server"]
        direction TB
        CloudCaddy["公開 Caddy<br/>TLS 1.3"]
        Bastion["OpenSSH Bastion<br/>ProxyJump"]
        CloudWG["ConoHa WireGuard peer"]
    end

    subgraph HomeEdge["宅内公開境界"]
        RouterFW["宅内 router<br/>IPv6 firewall"]
    end

    subgraph HomeServer["宅内 Ubuntu WoL サーバー"]
        direction TB
        HomeWG["宅内 WireGuard peer"]
        UbuntuFW["Ubuntu firewall<br/>送信元・port 制限"]
        HomeCaddy["宅内 Caddy<br/>公開 HTTPS または TCP 8080 origin"]
        SSHD["OpenSSH<br/>強制コマンド専用"]
        Dispatcher["wol-dispatch<br/>固定コマンド解析"]
        Web["Django Web<br/>認証・UI・Wake 受付"]
        Agent["WoL Agent<br/>送信・監視・状態遷移"]
        Catalog["root 所有 CSV<br/>許可ターゲット"]
        Identity[("identity.sqlite3<br/>認証・権限・session")]
        Operations[("operations.sqlite3<br/>操作・状態・監査")]
        Script["wol-send.sh<br/>固定 argv"]
    end

    subgraph HomeLAN["宅内 LAN"]
        direction TB
        Broadcast["IPv4 broadcast<br/>UDP port 9"]
        Target["対象 PC<br/>38:05:25:38:F0:C9"]
    end

    Browser -->|"Phase 5A（択一）<br/>DNS AAAA<br/>HTTPS / TLS 1.3 / IPv6"| RouterFW
    Browser -->|"Phase 5B（択一）<br/>HTTPS / TLS 1.3<br/>IPv4・IPv6"| CloudCaddy
    Operator -->|"SSH / IPv4"| Bastion

    CloudCaddy -->|"Phase 5B<br/>HTTP / TCP 8080"| CloudWG
    Bastion -->|"Phase 5B<br/>SSH / TCP 22"| CloudWG
    CloudWG <-->|"WireGuard 暗号化<br/>宅内側から接続開始"| HomeWG

    Bastion -->|"Phase 5A<br/>SSH / IPv6 / TCP 22"| RouterFW
    RouterFW -->|"TCP 443 / 22"| UbuntuFW
    HomeWG -->|"TCP 8080 / 22"| UbuntuFW
    UbuntuFW -->|"TCP 443 または 8080"| HomeCaddy
    UbuntuFW -->|"TCP 22"| SSHD

    HomeCaddy -->|"Unix socket"| Web
    SSHD -->|"SSH_ORIGINAL_COMMAND"| Dispatcher
    Dispatcher -->|"Unix socket<br/>固定 JSON"| Agent

    Web -->|"read only"| Catalog
    Web --> Identity
    Web --> Operations
    Agent -->|"実行直前に再検証"| Catalog
    Agent --> Operations
    Agent -->|"固定 3 引数"| Script
    Script -->|"Magic Packet"| Broadcast
    Broadcast --> Target
    Agent -->|"PING / TCP 22"| Target

    classDef client fill:#eff6ff,stroke:#2563eb,color:#172554
    classDef edge fill:#fff7ed,stroke:#ea580c,color:#431407
    classDef service fill:#f0fdf4,stroke:#16a34a,color:#052e16
    classDef data fill:#faf5ff,stroke:#9333ea,color:#3b0764
    classDef target fill:#fef2f2,stroke:#dc2626,color:#450a0a

    class Browser,Operator client
    class CloudCaddy,Bastion,CloudWG,RouterFW,HomeWG,UbuntuFW,HomeCaddy,SSHD,Dispatcher edge
    class Web,Agent,Script service
    class Catalog,Identity,Operations data
    class Broadcast,Target target
```

## 2. 経路の使い分け

| 構成・経路 | 公開 Web 入口 | 宅内 Ubuntu WoL サーバーまで | 用途 |
|---|---|---|---|
| Phase 5A | 宅内 Caddy の IPv6 TCP 443 | Web はIPv6で直接接続、管理CLIはConoHaからSSH IPv6 TCP 22 | 宅内 IPv6 着信が可能な場合 |
| Phase 5B | ConoHa Caddy の IPv4 / IPv6 TCP 443 | WebはWireGuard内TCP 8080、管理CLIはTCP 22 | 宅内 IPv6 着信が使えない場合 |
| ProxyJump CLI | ConoHa OpenSSH の IPv4 TCP 22 | Phase 5A は IPv6、Phase 5B は WireGuard | 制限済み管理 CLI |

## 3. 信頼境界

1. 公開 HTTPS 入口では TLS 1.3 だけを許可し、ProxyJump は SSH の暗号化で保護する。
2. Web から MAC、ブロードキャスト、UDP port、Shell command を受け取らない。
3. CSV は root 所有の許可リストとし、Web と Agent は読取専用とする。
4. Web は Wake 操作を `operations.sqlite3` に登録し、Shell Script を実行しない。
5. Agent は実行直前に CSV と `target_revision` を再検証する。
6. ProxyJump では任意 Shell を提供せず、UUID 付き固定 Wake command だけを許可する。
7. Magic Packet は ConoHa ではなく宅内 Ubuntu WoL サーバーから送信する。

## 4. Wake 操作の主な状態遷移

```text
Web / SSH request
  -> authenticated principal + target_id + idempotency key
  -> operations.sqlite3: ACCEPTED
  -> WoL Agent: CSV and revision revalidation
     -> expired before claim: EXPIRED / NOT_APPLICABLE
     -> invalid target or revision: REJECTED / NOT_APPLICABLE
     -> process not started: FAILED / NOT_APPLICABLE
     -> result uncertain after process start: OUTCOME_UNKNOWN / NOT_APPLICABLE
     -> /usr/local/libexec/wol-send.sh
        -> /usr/bin/wakeonlan -i BROADCAST -p 9 MAC
        -> PACKET_SENT
        -> PING / TCP 22 verification
        -> REACHABLE, NO_RESPONSE, or INTERRUPTED
```

`PACKET_SENT` は Magic Packet の送信処理が成功したことを示し、対象 PC の
起動完了を示さない。起動状態は後続の PING または TCP 22 で確認する。

詳細は [アーキテクチャ設計](01_ARCHITECTURE.md)、
[API とデータ設計](02_API_AND_DATA.md)、
[セキュリティ設計](03_SECURITY.md)を参照する。
