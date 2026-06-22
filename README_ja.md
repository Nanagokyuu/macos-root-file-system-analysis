# macOS Golden Gate（Apple Silicon）ルートファイルシステム・APFSボリュームグループ・ブートチェーン 深層解析

**言語：** [简体中文](README.md) [English](README_en.md) [日本語](README_ja.md)
**ドキュメントバージョン：** v1.0  
**解析対象：** MacBook Pro 14（macOS Golden Gate 27.0 beta 1）  
**解析方法：** 実機検証（diskutil、mount、firmlink、snapshot、APFS Volume Group、Simulator Runtime）

---

# 1. 概要

現代 macOS のストレージアーキテクチャは、表面上の見た目よりはるかに複雑だ。`/` の裏側で何が起きているかを本当に理解するには、Apple が長年にわたってシステムの階層設計に加えてきた進化を先に把握しておく必要がある。

macOS Catalina 以降、Apple はシステムを次の二つに分割した：

* System Volume（システムボリューム）
* Data Volume（データボリューム）

この変更の出発点は、読み取り専用のOS ファイルとユーザーが書き込み可能なデータを完全に分離し、後続のセキュリティ強化の基盤を築くことにある。

Big Sur からは、Apple はさらに踏み込んだセキュリティ機構を導入した：

* Signed System Volume (SSV)——システムボリュームへの暗号化署名検証
* APFS Snapshot Boot——ボリュームを直接マウントするのではなく、スナップショットから起動
* Firmlink Namespace——カーネルレベルでのクロスボリュームディレクトリマッピング

そして Apple Silicon プラットフォームでは、チップに内蔵された独立したセキュリティプロセッサとファームウェア信頼チェーンにより、ブートフローにさらにいくつかの新たな参加者が加わった：

* iSCPreboot——ローカル起動ポリシー（LocalPolicy）の保存と実行
* xART——信頼キャッシュとセキュアブートメタデータの保持
* Hardware——デバイスアクティベーション、工場出荷データ等のハードウェア関連情報を格納
* Secure Boot Policy——Apple Silicon 固有の多層セキュアブートポリシー体系

つまり、現代 macOS のルートディレクトリ `/` はもはや実際のディスクパーティション一つに対応するものではなく、複数のボリュームが共同で構築した統一名前空間（Namespace Union）なのだ。この点を理解しておくことが、本文のすべての内容を読み解く前提条件となる。

---

# 2. APFS コンテナ構造

APFS（Apple File System）は「コンテナ」（Container）を最上位のストレージ単位とし、その下に各独立した「ボリューム」（Volume）が存在する。本機のメイン APFS コンテナは `disk3` に位置し、内部には以下のボリュームが含まれる：

```text
disk3
├── disk3s1  System
├── disk3s2  Preboot
├── disk3s3  Recovery
├── disk3s5  Data
└── disk3s6  VM
```

そのうち、System ボリュームと Data ボリュームは論理的に同一の「ボリュームグループ」（Volume Group）に属する：

```text
disk3s1
Name: Macintosh HD
Role: System

disk3s5
Name: Data
Role: Data
```

これらは共同して以下に帰属する：

```text
APFS Volume Group
UUID:
937449E6-EFC1-4FF8-B7B9-10B984769757
```

Volume Group の概念は macOS Big Sur で導入されたものであり、その核心的な役割は System ボリュームと Data ボリュームを論理的にペアとして結びつけることだ。これにより Finder、diskutil、システムツールが両者を一つのまとまりとして管理でき、ユーザーは内部の分離構造を意識せずに済む。

---

# 3. 実際のブートボリューム

よくある誤解として、システムは `disk3s1`（System Volume）から起動するというものがある。実際はそうではない。

検証：

```bash
mount | grep ' / '
```

結果：

```text
/dev/disk3s1s1 on /
(apfs, sealed, read-only)
```

ここに三段階のデバイスノード `disk3s1s1` が登場するが、これは独立したパーティションではなく、`disk3s1` 上の **APFS Snapshot**（スナップショット）だ。完全なブート階層は以下のとおりだ：

```text
disk3s1
    ↓
Snapshot
    ↓
disk3s1s1
    ↓
/ へマウント
```

構造図：

```text
disk3s1
(System Volume)
    │
    ▼
Signed Snapshot
(disk3s1s1)
    │
    ▼
/
```

つまり、あなたが目にしている `/System`、`/bin`、`/usr` などのディレクトリは、実際には System Volume そのもののリアルタイム状態ではなく、不変のスナップショットから来ているということだ。

---

# 4. Signed System Volume（SSV）

SSV は macOS Big Sur で導入されたコアセキュリティ機能の一つであり、その目標はシステムファイルが出荷後に誰によっても（root ユーザーも含む）こっそり改ざんされないことを保証することだ。

検証：

```bash
diskutil info /
```

結果：

```text
Sealed: Yes
Volume Read-Only: Yes
```

現在起動しているのは：

```text
Signed System Volume
```

そのセキュリティ特性は複数の層に現れている：

* **読み取り専用（Read-Only）**：すべての書き込み操作は拒否される
* **Apple 署名**：ボリューム全体の内容は Apple がリリース時に署名を完了している
* **Hash Tree 検証**：カーネルがすべてのファイルにアクセスする際にハッシュ値を検証し、ファイルが改ざんされていないことを確認する
* **Secure Boot 検証**：iBoot はシステムをロードする前に SSV の署名が有効かどうかを検証する

したがって、以下のパスに存在するすべての内容：

```text
/System
/bin
/sbin
/usr
```

はすべて以下から来ている：

```text
disk3s1s1
```

Data ボリュームからではない。root ユーザーであっても、SSV の完全性を損なわずにこれらのパス内のファイルを変更することはできない。

---

# 5. Root Namespace Union

ユーザーが macOS を使うとき、`/` はひとつの完全で統一されたファイルシステムのように感じられる。しかし裏側で実際に起きているのは、二つの独立したボリュームの「名前空間の統合」（Namespace Union）だ。

検証：

```bash
ls -ldO /
```

結果：

```text
sunlnk
```

同時に：

```bash
ls -ldO /System/Volumes/Data
```

結果：

```text
sunlnk
```

`sunlnk` フラグ（Sun Link）は macOS カーネルが Namespace Union に参加するディレクトリノードをマークするためのフラグビットであり、システムが以下を使用していることを示す：

```text
Namespace Union
```

このメカニズムの論理的な構造は以下のとおりだ：

```text
System Snapshot
        +
Data Volume
        │
        ▼
統一ルートディレクトリ /
```

ユーザーが最終的に見るのは統合されたビューだ——System Snapshot からの読み取り専用のシステムファイルと、Data Volume からの書き込み可能なユーザーデータが、同じディレクトリツリーの中に共存し、その間の境界はユーザーには完全に透明になっている。

---

# 6. Firmlink メカニズム

Namespace Union が実現できる鍵となる技術の一つが、**Firmlink**（固態リンク）だ。Firmlink は Apple が APFS のために設計したクロスボリューム・ディレクトリマッピング機構であり、その動作はシンボリックリンク（Symlink）に似ているが、実装レベルでは根本的に異なる。

検証：

```bash
cat /usr/share/firmlinks
```

結果：

```text
/Applications
/Users
/Library
/private
/opt
/cores
/pkg
/usr/local
...
```

上記のパスはユーザーの目にはシステムルートディレクトリの下にあるように見えるが、実際にはすべて以下を指している：

```text
/System/Volumes/Data
```

例えば：

```text
/Applications
        │
        ▼
/System/Volumes/Data/Applications
```

```text
/Users
        │
        ▼
/System/Volumes/Data/Users
```

```text
/Library
        │
        ▼
/System/Volumes/Data/Library
```

Firmlink と従来のシンボリックリンクの主な違いは以下の表のとおりだ：

| 特性             | Firmlink | Symlink  |
| -------------- | -------- | -------- |
| カーネルレベル        | あり       | なし       |
| ユーザーから見える      | なし       | あり       |
| クロスボリュームマッピング  | あり       | なし       |
| Finder での表示    | 通常のディレクトリ | リンク      |

Firmlink がカーネルレベルで動作し、ユーザーには不可視であるために、大多数のアプリケーションは `/Applications` や `/Users` にアクセスする際、ボリュームの境界を完全に無意識に越えて、実際には Data Volume のコンテンツにアクセスしている。

---

# 7. ルートディレクトリ構造

Firmlink メカニズムを理解した上で、ルートディレクトリ内の各パスの実際の帰属をより明確に理解できる。

## 7.1 Data Volume に実際に格納されているディレクトリ

以下のパスは表面上はルートディレクトリに位置しているが、実際には Firmlink を通じて Data Volume にマッピングされている：

```text
/Applications
/Users
/Library
/private
/opt
/cores
/pkg
```

実際の場所：

```text
/System/Volumes/Data
```

これらはすべてユーザーが書き込み可能なディレクトリであり、システムの「可変」部分に属する。

---

## 7.2 シンボリックリンク

Firmlink の他にも、ルートディレクトリには従来のシンボリックリンクがいくつか存在しており、主に歴史的なパスの互換性維持のために使われている。

### etc

```text
/etc
    ↓
private/etc
```

---

### tmp

```text
/tmp
    ↓
private/tmp
```

---

### var

```text
/var
    ↓
private/var
```

`/private` は実際には System Snapshot の中に存在する実ディレクトリであり、`/etc`、`/tmp`、`/var` はそのサブディレクトリを指すシンボリックリンクだ。この設計もまた歴史的な経緯による——macOS の初期において、これらのパスの挙動は従来の Unix システムと一致するものだった。

---

### home

検証：

```bash
mount | grep home
```

結果：

```text
auto_home
```

構造：

```text
/home
    ↓
/System/Volumes/Data/home
```

`/home` の挙動は他のパスとは少し異なる。実際にはこれは：

```text
autofs
```

によって動的に管理されており、必要に応じてネットワークホームディレクトリ（Open Directory や LDAP バインドによるユーザーディレクトリなど）を自動マウントするオンデマンドマウント（on-demand mount）の仕組みだ。

---

# 8. Volumes ディレクトリ

`/Volumes` は macOS の従来のマウントポイントディレクトリであり、外部ストレージを含むすべての可視のディスクボリュームは通常ここにマウントされる。注目すべき点として、ここには混乱を招きやすい設計が存在する。

検証：

```bash
ls -ld /Volumes/"Macintosh HD"
```

結果：

```text
Macintosh HD -> /
```

したがって：

```text
/Volumes/Macintosh HD
```

これは実際には：

```text
シンボリックリンク
```

であり、以下を指している：

```text
/
```

循環構造を形成している：

```text
/Volumes/Macintosh HD
      │
      ▼
      /
```

この設計は主に古いソフトウェアとの互換性のためだ（一部のアプリケーションが `/Volumes/Macintosh HD/...` の形式でパスを参照している）。また Finder のナビゲーションロジックのためでもある——Finder のサイドバーでは「Macintosh HD」が独立したディスクアイコンとして表示され、クリックすると実際にはルートディレクトリ `/` にアクセスする。

---

# 9. Data Volume

Data Volume はユーザーの日常操作において最も頻繁に書き込みが行われるボリュームであり、システム実行時のすべての可変データを担っている。

マウント：

```text
disk3s5
```

パス：

```text
/System/Volumes/Data
```

検証：

```bash
mount
```

結果：

```text
protect
root data
```

このボリュームが以下であることを示している：

```text
Root Data Volume
```

その主な役割は以下の几つの側面をカバーしている：

* **ユーザーデータ**：ホームディレクトリ、ドキュメント、ダウンロードなどすべての個人ファイル
* **サードパーティアプリ**：App Store や他のチャネルからインストールされたアプリケーション
* **Home Directory**：すべてのローカルユーザーのホームディレクトリはこのボリューム上にある
* **書き込み可能なシステムデータ**：一部のシステムコンポーネントが実行時に書き込む必要がある設定とキャッシュ

System Volume の読み取り専用特性とは異なり、Data Volume は読み書き操作をサポートしており、FileVault 暗号化の主要な保護対象でもある。

---

# 10. Preboot Volume

Preboot Volume はブートプロセスの重要な参加者であり、カーネルロードの補助材料を提供する責務を担っている。

ボリューム：

```text
disk3s2
```

マウント：

```text
/System/Volumes/Preboot
```

格納されている内容は以下を含む：

```text
Kernel Collections   —— カーネルおよびカーネル拡張のコレクション（kextcache でビルド）
Boot Manifest        —— 起動マニフェスト（現在の起動設定を記述）
Boot Support Files   —— iBoot と初期起動フェーズに必要な補助ファイル
```

Preboot Volume は通常、起動完了後にユーザーが直接アクセスすることはないが、システムアップデートやカーネルキャッシュの再構築時にはシステムが書き込みを行う。

---

# 11. Recovery Volume

Recovery Volume は普段「休眠」状態にあり、システムリカバリーが必要なときにのみ起動しマウントされる。

ボリューム：

```text
disk3s3
```

Role：

```text
Recovery
```

通常の実行時にはマウントされず、ユーザーは直接アクセスできない。

システムが正常に起動できない場合、またはユーザーがリカバリーモードに能動的に入る場合（Apple Silicon では電源ボタン長押し、Intel では Cmd+R を押し続ける）に、このボリュームがロードされ、以下の機能を提供する：

```text
macOS Recovery       —— 完全なリカバリー環境
システム復元          —— Time Machine またはインターネットからシステムを復元
ディスクユーティリティ  —— First Aid、パーティション、消去などの操作
ターミナル            —— コマンドラインアクセス（高度なトラブルシューティング用）
システム再インストール  —— macOS をオンラインでダウンロードして再インストール
```

---

# 12. VM Volume

VM Volume は仮想メモリ管理専用であり、メモリ圧力下でシステムがページングを行う場所だ。

ボリューム：

```text
disk3s6
```

マウント：

```text
/System/Volumes/VM
```

その主な機能：

```text
swapfile          —— スワップファイル（メモリページをディスクにページアウト）
memory pressure   —— システムのメモリ圧力管理メカニズムとの連携
仮想メモリ         —— すべてのプロセスに統一された仮想アドレス空間を提供
```

このボリュームのマウントパラメータには以下が含まれる：

```text
noexec
```

すなわち、このボリューム上での実行ファイルの実行を禁止することで、潜在的なセキュリティエクスプロイト（swap に悪意あるコードを書き込んで実行を試みるもの）を防いでいる。

---

# 13. Update Volume

Update Volume はシステムアップデートフロー中の一時ワークスペースであり、更新内容が正式に適用される前に関連データを一時保存する役割を担う。

ボリューム：

```text
disk3s4
```

マウント：

```text
/System/Volumes/Update
```

その役割には以下が含まれる：

```text
システムアップデートキャッシュ  —— ダウンロード済みだが未適用のアップデートパッケージを格納
SSV 構築                      —— 新しい System Volume スナップショットの構築ワークスペース
アップデート中間状態            —— アップデートの進行状況を記録し、中断からの再開をサポート
```

この設計により macOS は現在実行中のシステムを中断することなく、バックグラウンドで新しいシステムスナップショットの構築を完了し、次回の再起動時に新しい Signed Snapshot に切り替えることができる。

---

# 14. Apple Silicon 補助ブートコンテナ

Apple Silicon のセキュアブートチェーンは Intel 時代よりはるかに複雑だ。メインストレージの `disk3` コンテナに加えて、システムにはもう一つの APFS コンテナが存在する：

```text
disk1
```

このコンテナは Apple Silicon のセキュアエンクレーブ（Secure Enclave）とブート補助チップ（SoC）の制御下で動作しており、その構造は以下のとおりだ：

```text
disk1
├── iSCPreboot
├── xART
├── Hardware
└── Recovery
```

これら四つのボリュームが共同して、Apple Silicon プラットフォーム固有の「セキュアブート補助層」を構成しており、iBoot が正式に macOS カーネルをロードする前の段階で重要な役割を果たす。

---

# 15. iSCPreboot

iSCPreboot（iSC は「Internal SoC」、すなわちシステムオンチップ内部を表す）は、Apple Silicon のブートポリシーのコアストレージボリュームだ。

ボリューム：

```text
disk1s1
```

マウント：

```text
/System/Volumes/iSCPreboot
```

その内容を探索すると、以下が見つかる：

```text
937449E6-EFC1-...
```

このディレクトリ名は：

```text
APFS Volume Group UUID
```

と完全に一致する。これは偶然ではなく、iSCPreboot の内容が特定の Volume Group に紐付いていることを明示している。登録された各 macOS インストールはその中の一つのディレクトリに対応する。

そのディレクトリの内部にはさらに以下が含まれる：

```text
LocalPolicy/
```

LocalPolicy は Apple Silicon セキュアブート体系のコアコンセプトの一つであり、本機の現在の macOS インストールに対して設定されたブートポリシーを記録している。具体的には以下を網羅している：

* **Startup Security**（起動セキュリティ）：完全なセキュリティ（Full Security）、セキュリティ低下（Reduced Security）
* **External Boot Policy**（外部ブートポリシー）：外部ストレージデバイスからの起動を許可するかどうか
* **Custom Kernel Extensions**：サードパーティのカーネル拡張のロードを許可するかどうか
* **MDM Enrollment**（モバイルデバイス管理）関連ポリシー

iSCPreboot の全体的なディレクトリ構造は以下のとおりだ：

```text
iSCPreboot
├── VolumeGroupUUID
│   └── LocalPolicy
├── SystemRecovery
├── WiFi
└── SFR
```

---

# 16. xART

xART は Apple Silicon ブートチェーンの中で最も機密性の高いコンポーネントの一つであり、その正式名称と内部実装は Apple によって公開されていない。

ボリューム：

```text
disk1s2
```

マウント：

```text
/System/Volumes/xarts
```

限定的な探索を行うと、以下が発見された：

```text
A375F60A-....gl
```

その物理的な特徴：

* サイズは約 6 MB と非常に小さい
* SIP（System Integrity Protection）による厳重な保護
* root ユーザーでさえその内容を読み取れない

現存するリバースエンジニアリング資料と Apple のセキュリティホワイトペーパーの間接的な手がかりから、その用途は以下と推測される：

```text
Trust Cache           —— システムが信頼する実行ファイルのハッシュキャッシュ
Boot Artifacts        —— ブートプロセスに必要な中間成果物
Secure Boot Metadata  —— SSV 検証を支えるメタデータ
IMG4 / IM4P 関連データ —— Apple ファームウェア署名フォーマットの関連データ
```

Apple は今に至るまで xART の詳細なドキュメントを公開しておらず、その内部フォーマットとアクセスプロトコルは外部の研究者にとって依然としてブラックボックスのままだ。

---

# 17. Hardware Volume

Hardware Volume はデバイスのライフサイクル中に蓄積された各種ハードウェア関連データを格納しており、デバイスのアイデンティティと健康状態の「アーカイブ室」だ。

ボリューム：

```text
disk1s3
```

マウント：

```text
/System/Volumes/Hardware
```

その内容を探索すると、以下のディレクトリとファイルが発見された：

```text
FactoryData          —— 工場出荷時に書き込まれたデバイスの基本データ
MobileActivation     —— デバイスのアクティベーション情報（Apple ID および iCloud 関連）
ProductDocuments     —— 製品ドキュメントと認証資料
dramecc              —— DRAM ECC（エラー訂正符号）状態の記録
recoverylogd         —— リカバリーフローのログ
srvo                 —— 未公開用途のハードウェアサービスデータ
```

これらのデータはデバイスの「出荷時固有属性」に属しており、通常の使用中には変更されないが、デバイスのアクティベーション、ハードウェア交換、または特定の修理シナリオ下ではシステムによって読み取られるか更新される可能性がある。

---

# 18. Simulator Runtime

macOS 上の iOS/iPadOS シミュレータは単純なソフトウェアレイヤーではなく、そのランタイム（Runtime）は独立した APFS ボリュームの形でマウントされる。本機には二種類の異なるランタイムメカニズムが存在しており、Apple が推進しているアーキテクチャの進化を体現している。

## 18.1 従来の Runtime

```text
disk5s1
```

マウント：

```text
/Library/Developer/CoreSimulator/Volumes/iOS_23E254a
```

対応：

```text
iOS 26.4.1 Simulator
```

これは従来のシミュレータランタイムのマウント方式だ：各 iOS バージョンのランタイムが独立した APFS ボリュームとして存在し、`/Library/Developer/CoreSimulator/Volumes/` 以下の対応するパスに読み取り専用でマウントされる。

---

## 18.2 Cryptex Runtime

```text
disk7s1
```

マウント：

```text
/private/var/run/com.apple.security.cryptexd/mnt/...
```

対応：

```text
iOS 27.0 Simulator
```

新バージョンのシミュレータランタイムは以下を採用し始めている：

```text
Cryptex
```

動的マウントのメカニズム。Cryptex は Apple が macOS Monterey と iOS 15 で導入したセキュリティコンテナフォーマットであり、当初は macOS のセキュリティアップデート（Rapid Security Response）に使用されていたが、現在はシミュレータランタイムの配布とマウントシナリオにも拡張され、より柔軟な署名検証とオンデマンドロード能力を備えている。この転換は、将来の Simulator Runtime が従来の静的ボリュームマウント方式ではなく、Cryptex チャネルに統一されることを意味している。

---

# 19. APFS 暗号化ユーザー

APFS はマルチユーザー暗号化（Multi-User Encryption）をサポートしており、そのユーザーモデルは macOS のログインアカウント体系と必ずしも一致しない。この点を正しく理解することは、FileVault を適切に使用する上で不可欠だ。

検証：

```bash
diskutil apfs listUsers /
```

結果：

```text
Local Open Directory User
Volume Owner: Yes

Personal Recovery User
Volume Owner: Yes
```

ここで強調すべき点は：**APFS ユーザーは macOS ログインユーザーと等しくない。**

APFS ユーザーはボリューム暗号化キー（Volume Encryption Key, VEK）の解錠権限を持つエンティティだ。現在のボリュームには Owner レベルの APFS ユーザーが二つ存在する：

**Local Open Directory User**：ローカル macOS アカウントの暗号化キー保有者を表す。ユーザーがログインパスワードを入力すると、このユーザーに対応するキープロテクター（Key Protector）がアンロックされ、ボリューム全体が復号される。

**Personal Recovery User**：以下のシナリオで使用される：

* ユーザーが FileVault パスワードを忘れた際に、個人用回復キー（Personal Recovery Key）でボリュームをアンロック
* Recovery 環境での認証
* Apple ID と連携した iCloud 管理の回復フロー

---

# 20. 完全なブートチェーン

上述したすべてのコンポーネントをつなぎ合わせると、macOS Apple Silicon の完全なブートチェーンは以下のとおりだ。各環が前の環の信頼の基盤であり、どこか一箇所での検証失敗がブートフロー全体を中断させる：

```text
Boot ROM
    │  （SoC に固化、Apple 署名、変更不可）
    ▼
LLB
    │  （Low-Level Bootloader、第一段階ブートローダー）
    ▼
iBoot
    │  （第二段階ブートローダー、後続コンポーネントの検証を担当）
    ▼
iSCPreboot
    │
    └── LocalPolicy
            （ブートポリシーを読み込んで適用：セキュリティレベル、外部ブート権限等）
    │
    ▼
xART
    │
    └── Trust Cache
        Boot Artifacts
        （後続でロードされるコンポーネントの署名の正当性を検証）
    │
    ▼
disk3s1
(System Volume)
    │
    ▼
Signed Snapshot
(disk3s1s1)
    │  （SSV 署名を検証、Hash Tree の完全性を確認）
    ▼
Namespace Union
    │
    ├── System Snapshot    （読み取り専用、disk3s1s1 から）
    └── Data Volume        （読み書き可能、disk3s5 から）
    │
    ▼
最終ルートディレクトリ /
```

---

# 21. 最終結論

以上の各コンポーネントへの逐一の分析を通じて、明確な結論を導き出すことができる：

現代の Apple Silicon macOS の `/` は単一ファイルシステムではなく、以下のコンポーネントが共同して構築した論理的なビューだ：

```text
Boot Policy
(iSCPreboot)
        +
Secure Boot Artifacts
(xART)
        +
Signed System Snapshot
(disk3s1s1)
        +
Root Data Volume
(disk3s5)
        +
Firmlink Namespace
        +
Symlink Layer
        +
Autofs
        │
        ▼
最終ユーザーが見る /
```

システム実装の観点から見ると、macOS のルートディレクトリは実際には **APFS Snapshot、Volume Group、Firmlink、Secure Boot、および Namespace Union** によって共同構成された論理ファイルシステムビューであり、従来の意味での単一ディスクパーティションではない。

この設計の核心的な価値は、ユーザー体験を犠牲にすることなく、システムファイルへの暗号化署名保護、ユーザーデータの読み書きの柔軟性、そしてファームウェアからオペレーティングシステムまでの完全な信頼チェーン——この三つが、一見普通に見える同じ `/` の裏側で和諧共存していることを実現した点にある。これこそが Apple Silicon セキュリティアーキテクチャの最も精妙な部分の一つだ。
