# macOS Golden Gate（Apple Silicon）ルートファイルシステム・APFSボリュームグループ・ブートチェーン 詳細解説

**言語:** [简体中文](README.md) [English](README_en.md) [日本語](README_ja.md)  
**ドキュメントバージョン:** v1.0  
**対象機種:** MacBook Pro 14（macOS Golden Gate 27.0 beta 1）  
**解析方法:** 実機検証（diskutil、mount、firmlink、snapshot、APFS Volume Group、Simulator Runtime）

---

# 1. 概要

現代のmacOSのストレージアーキテクチャは、表面上見えている以上にはるかに複雑だ。`/` の裏側で何が起きているのかを本当に理解するには、まずAppleがこれまでどのように多層的なシステム設計を進化させてきたかを把握する必要がある。

macOS Catalinaから、Appleはシステムを以下の2つに分割した：

* System Volume（システムボリューム）
* Data Volume（データボリューム）

この変更の動機は、読み取り専用のOSファイルとユーザーが書き込み可能なデータを完全に分離し、その後のセキュリティ強化の基盤を築くことにある。

Big Surからは、Appleはさらに踏み込み、より強力なセキュリティ機構を導入した：

* Signed System Volume（SSV）— システムボリュームの暗号署名検証
* APFS Snapshot Boot — ボリュームを直接マウントするのではなく、スナップショットからブート
* Firmlink Namespace — カーネルレベルで実装されたクロスボリュームのディレクトリマッピング

Apple Siliconプラットフォームでは、チップに専用のセキュリティプロセッサとファームウェアトラストチェーンが内蔵されているため、ブートプロセスにいくつかの新たなコンポーネントが追加された：

* iSCPreboot — ローカルブートポリシー（LocalPolicy）の保存と適用
* xART — トラストキャッシュとセキュアブートメタデータの保持
* Hardware — デバイスアクティベーション、工場出荷時データ、その他ハードウェア関連情報の保存
* Secure Boot Policy — Apple Siliconの多段階セキュアブートポリシーフレームワーク

その結果、現代のmacOSのルートディレクトリ `/` は、もはや単一の物理ディスクパーティションに対応するものではない。複数のボリュームによって構成される統合ネームスペースだ。これを理解することが、本ドキュメントのすべての前提となる。

---

# 2. APFSコンテナ構造

APFS（Apple File System）は「コンテナ」を最上位のストレージ単位とし、その下に個々の「ボリューム」が存在する。この機種のメインAPFSコンテナは `disk3` に存在し、以下のボリュームを含む：

```text
disk3
├── disk3s1  System
├── disk3s2  Preboot
├── disk3s3  Recovery
├── disk3s5  Data
└── disk3s6  VM
```

SystemボリュームとDataボリュームは、論理的に同じ「Volume Group」に属する：

```text
disk3s1
Name: Macintosh HD
Role: System

disk3s5
Name: Data
Role: Data
```

両者が属するのは：

```text
APFS Volume Group
UUID:
937449E6-EFC1-4FF8-B7B9-10B984769757
```

Volume Groupという概念はmacOS Big Surで導入された。その主な目的は、SystemボリュームとDataボリュームを論理的にペアとして結びつけ、Finder・diskutil・システムユーティリティが単一の論理単位として管理できるようにすること——ユーザーが内部の分離構造を意識せずに済むようにすることだ。

---

# 3. 実際のブートボリューム

よくある誤解として、システムが `disk3s1`（System Volume）からブートしているというものがある。実際はそうではない。

検証：

```bash
mount | grep ' / '
```

結果：

```text
/dev/disk3s1s1 on /
(apfs, sealed, read-only)
```

ここで3階層のデバイスノード `disk3s1s1` が現れる。これは独立したパーティションではなく、`disk3s1` 上の **APFSスナップショット** だ。完全なブート階層は以下のとおり：

```text
disk3s1
    ↓
Snapshot
    ↓
disk3s1s1
    ↓
/ にマウント
```

図解：

```text
disk3s1
（System Volume）
    │
    ▼
Signed Snapshot
（disk3s1s1）
    │
    ▼
/
```

つまり、見えている `/System`・`/bin`・`/usr` などのディレクトリは、System Volume自体の現在の状態ではなく、不変のスナップショットから来ているということだ。

---

# 4. Signed System Volume（SSV）

SSVはmacOS Big Surで導入された中核となるセキュリティ機能の一つだ。その目的は、システムファイルが出荷後にroot権限を持つ者を含む誰によっても、密かに改ざんされないことを保証することにある。

検証：

```bash
diskutil info /
```

結果：

```text
Sealed: Yes
Volume Read-Only: Yes
```

現在のブートは：

```text
Signed System Volume
```

そのセキュリティ特性は複数のレベルで機能する：

* **読み取り専用**：書き込み試行はすべて拒否される
* **Apple署名**：リリース時にAppleがボリューム全体のコンテンツに署名
* **ハッシュツリー検証**：カーネルがアクセスするすべてのファイルのハッシュを検証し、改ざんがないことを確認
* **セキュアブート検証**：iBootがシステム読み込み前にSSV署名を検証

したがって、以下のパス以下にあるすべてのファイルは：

```text
/System
/bin
/sbin
/usr
```

次のものから来ている：

```text
disk3s1s1
```

——Dataボリュームではない。rootでさえも、SSVの整合性を破らずにこれらのパスのファイルを変更することはできない。

---

# 5. ルートネームスペースの統合

macOSを使うとき、ユーザーには `/` が単一の統合ファイルシステムとして見える。実際に裏側で起きているのは、2つの独立したボリュームによる「ネームスペースの統合」だ。

検証：

```bash
ls -ldO /
```

結果：

```text
sunlnk
```

そして：

```bash
ls -ldO /System/Volumes/Data
```

結果：

```text
sunlnk
```

`sunlnk` フラグ（Sun Link）は、macOSカーネルがNamespace Unionに参加しているディレクトリノードをマークするために使用するもので、システムが以下のメカニズムを使っていることを示す：

```text
Namespace Union
```

このメカニズムの論理構造は以下のとおり：

```text
System Snapshot
        +
Data Volume
        │
        ▼
統合ルート /
```

ユーザーが最終的に見るのはマージされたビューだ——System Snapshotからの読み取り専用のシステムファイルと、Data Volumeからの書き込み可能なユーザーデータが同じディレクトリツリーに共存し、両者の境界はユーザーからは意識されない。

---

# 6. Firmlinkの仕組み

Namespace Unionを実現する鍵となる技術の一つが **Firmlink** だ。FirmlinkはAppleがAPFS専用に設計したクロスボリュームのディレクトリマッピング機構だ。シンボリックリンク（Symlink）に似た動作をするが、実装レベルでは根本的に異なる。

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

これらのパスはシステムルートに存在しているように見えるが、実際にはすべて次の場所を指している：

```text
/System/Volumes/Data
```

例：

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

Firmlinkとシンボリックリンクとの中核となる違いを以下にまとめる：

| 特性       | Firmlink         | Symlink          |
| -------------- | ---------------- | ---------------- |
| カーネルレベル   | あり             | なし             |
| ユーザーから見える | なし           | あり             |
| クロスボリューム | あり             | なし             |
| Finderの表示   | 通常のディレクトリ | リンク           |

Firmlinkはカーネルレベルで動作し、ユーザーからは見えないため、`/Applications` や `/Users` にアクセスするほぼすべてのアプリケーションは、Data Volumeのコンテンツにアクセスしていることをまったく知らないまま、ボリュームの境界を静かに越えている。

---

# 7. ルートディレクトリ構造

Firmlinkの仕組みを踏まえると、ルートディレクトリ内のパスが実際にどこに存在するかをより明確に理解できる。

## 7.1 Dataボリューム上に物理的に存在するディレクトリ

以下のパスはルートディレクトリ以下に見えるが、実際にはFirmlinkを通じてDataボリュームにマッピングされている：

```text
/Applications
/Users
/Library
/private
/opt
/cores
/pkg
```

物理的に存在する場所：

```text
/System/Volumes/Data
```

これらはユーザーが書き込み可能なディレクトリ——システムの「変更可能」な部分だ。

---

## 7.2 シンボリックリンク

Firmlinkに加え、ルートディレクトリには従来のシンボリックリンクも多数存在する。主に後方互換性のパスを維持するためのものだ。

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

`/private` はSystem Snapshot内に実際に存在するディレクトリであり、`/etc`・`/tmp`・`/var` はそのサブディレクトリへのシンボリックリンクだ。この設計は歴史的な遺産でもある——macOSの初期バージョンでは、これらのパスは従来のUnixシステムと一貫した動作をしていた。

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

`/home` は他のパスとは少し異なる動作をする。実際には以下によって動的に管理されている：

```text
autofs
```

——オンデマンドでネットワークホームディレクトリを自動マウントする役割を持つ（Open DirectoryやLDAP経由でバインドされたユーザーディレクトリなど）。これはオンデマンドマウントの仕組みだ。

---

# 8. Volumesディレクトリ

`/Volumes` はmacOSの伝統的なマウントポイントディレクトリで、すべての可視ディスクボリューム（外部ストレージを含む）は通常ここにマウントされる。ここには人々を混乱させがちな設計上の細部がある。

検証：

```bash
ls -ld /Volumes/"Macintosh HD"
```

結果：

```text
Macintosh HD -> /
```

つまり：

```text
/Volumes/Macintosh HD
```

は実際には：

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

この設計は主に、古いソフトウェアとの互換性（`/Volumes/Macintosh HD/...` 形式のパスを参照するアプリケーションが存在する）とFinderのナビゲーションロジックのためだ——Finderのサイドバーでは「Macintosh HD」は独立したディスクアイコンとして表示され、クリックするとルートディレクトリ `/` にアクセスする。

---

# 9. Dataボリューム

Dataボリュームは日常的な使用において最も頻繁に書き込まれるボリュームであり、システムのすべての変更可能なランタイムデータを担っている。

マウント先：

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

その主な役割はいくつかの領域をカバーする：

* **ユーザーデータ**：ホームディレクトリ、書類、ダウンロード、すべての個人ファイル
* **サードパーティアプリケーション**：App Storeやその他のソースからインストールされたアプリ
* **ホームディレクトリ**：すべてのローカルユーザーのホームディレクトリがこのボリュームに存在する
* **書き込み可能なシステムデータ**：一部のシステムコンポーネントがランタイムに書き込む設定やキャッシュ

読み取り専用のSystem Volumeとは異なり、DataボリュームはRead/Write操作をサポートし、FileVaultによる暗号化の主要な対象だ。

---

# 10. Prebootボリューム

Prebootボリュームはブートプロセスの重要な構成要素であり、カーネルの読み込みに必要なサポート資材を供給する役割を担う。

ボリューム：

```text
disk3s2
```

マウント先：

```text
/System/Volumes/Preboot
```

ここに格納されているコンテンツ：

```text
Kernel Collections   —— カーネルとカーネル拡張コレクション（kextcacheによりビルド）
Boot Manifest        —— 現在のブート設定を記述したブートマニフェスト
Boot Support Files   —— iBootや初期ブートステージが必要とする補助ファイル
```

Prebootボリュームは通常、ブート後にユーザーが直接アクセスすることはないが、OSアップデート時やカーネルキャッシュの再構築時にシステムが書き込む。

---

# 11. Recoveryボリューム

Recoveryボリュームはほとんどのときは休眠状態で存在し、システムリカバリが必要なときにのみアクティブになってマウントされる。

ボリューム：

```text
disk3s3
```

ロール：

```text
Recovery
```

通常動作中はマウントされず、ユーザーは直接アクセスできない。

システムが正常にブートできない場合、またはユーザーが明示的にリカバリモードに入る場合（Apple Siliconでは電源ボタン長押し、IntelではCmd+R長押し）、このボリュームが読み込まれ、以下を提供する：

```text
macOS Recovery       —— 完全なリカバリ環境
Restore System       —— Time Machineまたはインターネットから復元
Disk Utility         —— 応急処置、パーティション、消去など
Terminal             —— 高度なトラブルシューティング向けコマンドラインアクセス
Reinstall macOS      —— インターネットからmacOSをダウンロードして再インストール
```

---

# 12. VMボリューム

VMボリュームは仮想メモリ管理専用のボリュームで、メモリ圧迫時にシステムがメモリをページアウトする場所だ。

ボリューム：

```text
disk3s6
```

マウント先：

```text
/System/Volumes/VM
```

主な機能：

```text
swapfile          —— メモリをディスクにページングするスワップファイル
memory pressure   —— システムのメモリ圧力管理と連携
Virtual Memory    —— すべてのプロセスの統合仮想アドレス空間を支える
```

このボリュームのマウントパラメータには以下が含まれる：

```text
noexec
```

——つまりこのボリュームから実行ファイルを実行できず、潜在的なセキュリティ悪用を防止している（悪意あるコードをスワップに書き込んで実行しようとする攻撃など）。

---

# 13. Updateボリューム

Updateボリュームはシステムアップデートプロセスにおける一時的な作業スペースとして機能し、ライブシステムに反映される前のデータをステージングする。

ボリューム：

```text
disk3s4
```

マウント先：

```text
/System/Volumes/Update
```

その機能：

```text
System Update Cache     —— ダウンロード済みだが未適用のアップデートパッケージを保存
SSV Construction        —— 新しいSystem Volumeスナップショットをビルドする作業スペース
Update Intermediate State —— アップデートの進捗を記録し、中断後の再開を可能にする
```

この設計により、macOSは実行中のシステムを中断させることなくバックグラウンドで新しいシステムスナップショットをビルドし、次回の再起動時に新しいSigned Snapshotへ切り替えることができる。

---

# 14. Apple Silicon 補助ブートコンテナ

Apple Siliconのセキュアブートチェーンは、Intelの時代よりも大幅に複雑だ。メインの `disk3` コンテナに加えて、システムにはもう一つAPFSコンテナが存在する：

```text
disk1
```

このコンテナはApple SiliconのSecure Enclaveとブート補助チップ（SoC）の制御下で動作する。その構造は：

```text
disk1
├── iSCPreboot
├── xART
├── Hardware
└── Recovery
```

これら4つのボリュームが合わさって、Apple Silicon固有の「セキュアブート補助レイヤー」を形成し、iBootが正式にmacOSカーネルを読み込む前に重要な役割を果たす。

---

# 15. iSCPreboot

iSCPreboot（iSCは「Internal SoC」の略）は、Apple Siliconのブートポリシーの中核となるストレージボリュームだ。

ボリューム：

```text
disk1s1
```

マウント先：

```text
/System/Volumes/iSCPreboot
```

その内容を調べると：

```text
937449E6-EFC1-...
```

このディレクトリ名は：

```text
APFS Volume Group UUID
```

と完全に一致する。これは偶然ではない——iSCPrebootのコンテンツを特定のVolume Groupに明示的に結びつけているのだ。登録された各macOSインストールに対してこのようなディレクトリが一つ対応する。

そのディレクトリの中には：

```text
LocalPolicy/
```

LocalPolicyはApple Siliconのセキュアブートフレームワークの中心概念の一つだ。この機種の現在のmacOSインストールに対して設定されたブートポリシーを記録しており、以下をカバーする：

* **起動セキュリティ**：完全なセキュリティ、セキュリティを下げる
* **外部ブートポリシー**：外部ストレージからのブートを許可するかどうか
* **カスタムカーネル拡張**：サードパーティのカーネル拡張の読み込みを許可するかどうか
* **MDM登録**関連のポリシー

iSCPrebootの全体的なディレクトリ構造は：

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

xARTはApple Siliconのブートチェーンの中で最も不透明なコンポーネントの一つだ。フルネームも内部実装も、Appleによって公式に文書化されていない。

ボリューム：

```text
disk1s2
```

マウント先：

```text
/System/Volumes/xarts
```

限られた調査で判明したのは以下の通り：

```text
A375F60A-....gl
```

物理的な特徴：

* サイズは約6MB——極めて小さい
* SIP（システム整合性保護）によって厳重に保護されている
* rootでさえもコンテンツを読み取れない

利用可能なリバースエンジニアリングの知見とAppleのセキュリティホワイトペーパーからの間接的なヒントに基づいた推定機能：

```text
Trust Cache           —— システムが信頼する実行可能ファイルのハッシュキャッシュ
Boot Artifacts        —— ブートプロセス中に必要な中間成果物
Secure Boot Metadata  —— SSV検証をサポートするメタデータ
IMG4 / IM4P data      —— Appleのファームウェア署名形式に関連するデータ
```

xARTについてAppleは詳細なドキュメントを公開しておらず、その内部フォーマットとアクセスプロトコルは外部の研究者にとってブラックボックスのままだ。

---

# 17. Hardwareボリューム

Hardwareボリュームはデバイスのライフタイムを通じて蓄積された各種ハードウェア関連データを格納する——実質的にデバイスのアイデンティティと健全性ステータスの「書庫」だ。

ボリューム：

```text
disk1s3
```

マウント先：

```text
/System/Volumes/Hardware
```

その内容を調べると、以下のディレクトリとファイルが見つかる：

```text
FactoryData          —— 工場で書き込まれたデバイスのベースラインデータ
MobileActivation     —— デバイスアクティベーション情報（Apple IDとiCloudに関連）
ProductDocuments     —— 製品ドキュメントと認証資料
dramecc              —— DRAM ECC（エラー訂正符号）ステータスの記録
recoverylogd         —— リカバリプロセスのログ
srvo                 —— 目的が非公開のハードウェアサービスデータ
```

このデータはデバイスの「工場出荷時の固有属性」を表しており、通常の使用では変更されないが、デバイスアクティベーション、ハードウェア交換、または特定の修理シナリオでシステムが読み取りまたは更新することがある。

---

# 18. Simulator Runtime

macOS上のiOS/iPadOSシミュレータは純粋なソフトウェアレイヤーではない——そのランタイムはスタンドアロンのAPFSボリュームとしてマウントされる。この機種では2つの異なるランタイムメカニズムが動作しており、Appleが積極的に推進しているアーキテクチャの移行が反映されている。

## 18.1 従来のランタイム

```text
disk5s1
```

マウント先：

```text
/Library/Developer/CoreSimulator/Volumes/iOS_23E254a
```

対応：

```text
iOS 26.4.1 Simulator
```

これは従来のシミュレータランタイムのマウント方法だ：各iOSバージョンのランタイムが独立したAPFSボリュームとして存在し、`/Library/Developer/CoreSimulator/Volumes/` 以下の対応するパスに読み取り専用でマウントされる。

---

## 18.2 Cryptexランタイム

```text
disk7s1
```

マウント先：

```text
/private/var/run/com.apple.security.cryptexd/mnt/...
```

対応：

```text
iOS 27.0 Simulator
```

新しいシミュレータランタイムは以下のメカニズムを採用し始めている：

```text
Cryptex
```

動的マウントのための仕組みだ。CryptexはAppleがmacOS MontereyとiOS 15で導入したセキュアコンテナ形式で、元々はmacOSのセキュリティアップデート（Rapid Security Response）のために開発され、その後シミュレータランタイムの配布とマウントにも拡張された。より柔軟な署名検証とオンデマンドローディングを提供する。この移行は、将来のシミュレータランタイムがすべてCryptexチャネルを経由するようになることを示唆しており、従来の静的ボリュームマウント方式に取って代わる。

---

# 19. APFS暗号化ユーザー

APFSはマルチユーザー暗号化をサポートしており、そのユーザーモデルはmacOSのログインアカウントと一対一で対応しない。FileVaultを正しく扱うには、この違いを理解することが不可欠だ。

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

強調しておきたいのは：**APFSユーザーはmacOSのログインユーザーとは同じではない**。

APFSユーザーとは、ボリューム暗号化キー（VEK）をアンロックする権限を持つエンティティだ。現在このボリュームにはOwnerレベルのAPFSユーザーが2つある：

**Local Open Directory User**：ローカルmacOSアカウントの暗号化キー保持者を表す。ユーザーがログインパスワードを入力すると、このユーザーに対応するキープロテクターがアンロックされ、それがボリューム全体を復号化する。

**Personal Recovery User**：以下のシナリオで使用される：

* ユーザーがFileVaultパスワードを忘れた場合に、パーソナルリカバリキーでボリュームをアンロックする
* リカバリ環境内での認証
* Apple IDと連携したiCloud管理のリカバリフロー

---

# 20. 完全なブートチェーン

上記のすべてのコンポーネントを合わせると、Apple Silicon上のmacOSの完全なブートチェーンは以下のようになる。各ステップが次のステップの信頼の基盤となり、どこかのポイントで検証が失敗すると、ブートプロセス全体が中止される：

```text
Boot ROM
    │  （SoCにハードワイヤードされ、Apple署名済み、不変）
    ▼
LLB
    │  （Low-Level Bootloader、第1段階ブートローダー）
    ▼
iBoot
    │  （第2段階ブートローダー、後続コンポーネントの検証を担当）
    ▼
iSCPreboot
    │
    └── LocalPolicy
            （ブートポリシーの読み取りと適用：セキュリティレベル、外部ブート許可など）
    │
    ▼
xART
    │
    └── Trust Cache
        Boot Artifacts
        （後から読み込まれるコンポーネントの署名の正当性を検証）
    │
    ▼
disk3s1
（System Volume）
    │
    ▼
Signed Snapshot
（disk3s1s1）
    │  （SSV署名を検証し、ハッシュツリーの整合性を確認）
    ▼
Namespace Union
    │
    ├── System Snapshot    （読み取り専用、disk3s1s1から）
    └── Data Volume        （読み書き可能、disk3s5から）
    │
    ▼
最終的なルート /
```

---

# 21. まとめ

各コンポーネントの分析を通じて、以上から次のことが分かる：

現代のApple Silicon macOSの `/` は単一のファイルシステムではない。以下に挙げる複数の要素によって構成される論理ビューだ：

```text
Boot Policy
（iSCPreboot）
        +
Secure Boot Artifacts
（xART）
        +
Signed System Snapshot
（disk3s1s1）
        +
Root Data Volume
（disk3s5）
        +
Firmlink Namespace
        +
Symlink Layer
        +
Autofs
        │
        ▼
エンドユーザーが見る /
```

システム実装の観点から言えば、macOSのルートディレクトリは実のところ、**APFSスナップショット・Volume Group・Firmlink・セキュアブート・Namespace Union** によって構成された論理ファイルシステムビューであり、従来の意味での単一ディスクパーティションではない。

この設計の中核となる価値は、ユーザー体験を犠牲にすることなく3つのことを同時に実現している点にある：システムファイルの暗号署名保護、ユーザーデータの読み書き柔軟性、そしてファームウェアからオペレーティングシステムまでの一切途切れのないトラストチェーン。これらすべてが普通の `/` に見えるものの裏側に共存している——そしてそれこそがApple Siliconセキュリティアーキテクチャの最もエレガントな側面の一つだ。
