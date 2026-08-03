---
title: "macOS 26 TahoeでTime Machineが必ず失敗する原因は、日本語のボリューム名だった"
emoji: "🍎"
type: "tech"
topics: ["macos", "timemachine", "nas", "smb", "troubleshooting"]
published: true
---

## 結論

NAS（SMB）へのTime Machineバックアップが毎回失敗する場合、**バックアップ先ボリュームの名前をASCIIにリネームする**と直ることがあります。

```bash
# ディスクイメージがアタッチされている状態で
sudo diskutil rename /dev/disk38s1 TMBackup
```

日本語環境のmacOSは、バックアップ先を `<Mac名>のバックアップ` と自動命名します。この名前が **NFD（濁点が結合文字として分離した形）** でマウントパスに現れ、`backupd` の内部処理がマウントテーブルと照合できず「ボリュームが存在しない」と誤判定します。

以下は、そこに辿り着くまでに **4つの仮説を立てて全部外した** 記録です。同じ症状で消耗している人の時間を節約できればと思います。

## 環境

| | |
|---|---|
| Mac | macOS 26.2 (Tahoe) |
| バックアップ先 | UGREEN DXP2800（SMB共有） |
| 接続 | Wi-Fi 5GHz (802.11ax, ch36, -48dBm) |
| データ量 | 約1.7TB（初回フルバックアップ） |

半年以上にわたり、**一度も成功していませんでした**。

## 症状

`tmutil startbackup` を実行すると `MountingDiskImage` フェーズで止まり、しばらくして停止します。

```
$ defaults read /Library/Preferences/com.apple.TimeMachine.plist Destinations
RESULT = 70;
```

`70` は `BACKUP_FAILED_DISCONNECTED_DISK_IMAGE` です。「ディスクイメージが切断された」という意味になります。

Wi-Fi経由なので「無線が不安定なのだろう」と考えるのが自然です。**これが最初の落とし穴でした。**

## ハズレだった仮説 4つ

### ① 中断ファイル（`.incomplete`）の残骸

NAS側に `<Mac名> <UUID>.incomplete` というディレクトリが残っていました。過去の失敗した作成処理の残骸です。macOSがこれを削除しようとして「Resource busy」で失敗しているのではないか、と考えました。

SMB越しでは消せなかったので、NAS側にSSHで入って直接削除しました。

**結果**: 次の試行で少し先へ進んだものの、また同じ `.incomplete` が生成されて同じエラー。**残骸は結果であって原因ではありませんでした。**

### ② `nsmb.conf` の soft mount

`/etc/nsmb.conf` に以下を設定していました。

```ini
[default]
signing_required=yes
streams=yes
soft=yes
protocol_vers_map=6
```

`soft=yes` は「SMBがタイムアウトしたらリトライせずエラーを返す」設定です。ディスクイメージ利用時は一瞬の詰まりで即「切断」扱いになりうるため、これが `DISCONNECTED_DISK_IMAGE` の直接原因ではないかと考えました。

`soft=no`（ハードマウント）に変更。

**結果**: 同じ7秒で同じエラー70。しかも今回は `.incomplete` すら作られず、sparsebundleも1バイトも増えませんでした。

### ③ バックアップ先UUIDの不一致

途中で sparsebundle を作り直したところ、エラーコードが `1008`（`BACKUP_FAILED_TARGETVOL_NOT_RECOGNIZED`）に変わりました。ログにはこう出ます。

```
Identity check failed: current volume UUID does not match trusted UUIDs
```

sparsebundle を削除したことでボリュームUUIDが変わり、Time Machineが保持する「信頼済みUUIDリスト」と食い違ったためです。システム設定からバックアップ先を登録し直して解決しました。

**結果**: エラーは `1008` から **`70` に戻っただけ**。前進ではありましたが、本丸ではありません。

### ④ 暗号化状態の不一致 ← 最も紛らわしかった

設定を読むと、こうなっていました。

```
LastKnownEncryptionState = Encrypted;      ← Time Machineは「暗号化済み」と記録
DiskImageKeychainUUID = 4672BB9F-...;      ← 鍵のUUIDも保持している
```

一方、実際のボリュームは暗号化されていません。

```
$ diskutil apfs list
FileVault:  No
```

さらに、**失敗の直前に `SecKeychainFindGenericPassword` が2回呼ばれている**ことがログから分かりました。System keychain には該当エントリが実際に残っています。

「暗号化ボリュームを削除したのに、記録と鍵だけが残った。この不整合で `backupd` がボリュームを切り離している」——**非常にもっともらしく見えたので、確信を持って断定してしまいました。**

宛先の登録解除・古い鍵の削除・sparsebundle の削除まで行い、暗号化を有効にして作り直し。

**結果**: 同じ7秒で、同じ `70`。**完全に外れでした。**

:::message
「もっともらしい不一致」は真因とは限りません。この4つ目は特に、証拠が揃って見えるぶん危険でした。実験で否定されるまで、私は解決したと思い込んでいました。
:::

## 決め手になった3つの切り分け

### ① 失敗後もディスクイメージとSMB共有は生きている

エラーは「ディスクイメージが切断された」です。であれば切断されているはずなので、確認しました。

```bash
$ diskutil list
/dev/disk37 (disk image):
   0:                  +16.0 TB    disk37          ← 生きている
/dev/disk38 (synthesized):
   0:  APFS Container Scheme       disk38          ← 生きている
       Physical Store disk37

$ mount | grep TimeMachine
//user@NAS._smb._tcp.local./TimeMachine on /Volumes/.timemachine/... (smbfs)
                                                             ← 繋がったまま
```

**ディスクイメージもSMB共有も無傷でした。**つまりネットワーク断ではありません。この時点でWi-Fi起因説が消えます。

### ② 手動でマウントすると消えない

Time Machineを介さず、同じボリュームを手でマウントして放置しました。

```bash
$ diskutil mount disk38s1
Volume ... on disk38s1 mounted

# 30秒間、1秒ごとに確認 → 一度も消えない
```

**ボリュームもSMBも健全です。**壊れるのはTime Machine経由のときだけ、と確定しました。

### ③ 失敗判定の「後」にアンマウントしている

`backupd` のログを時系列に並べると、決定的な矛盾が見つかりました。

```
07:25:02.619  Mounted disk38s1 at '/Volumes/<Mac名>のバックアップ'
07:25:05.191  Checking identity of target volume '/Volumes/<Mac名>のバックアップ'   ← 成功
07:25:05.192  Checking role of target volume '/Volumes/<Mac名>のバックアップ'       ← 成功
07:25:06.465  Set quota for '/Volumes/<Mac名>のバックアップ' to none               ← 成功
07:25:07.619  '/Volumes/<Mac名>のバックアップ' - no volume mounted at this path    ← ❌
07:25:07.633  Failing backup with BACKUP_FAILED_DISCONNECTED_DISK_IMAGE (70)
07:25:09.160  Unmounted '/Volumes/<Mac名>のバックアップ'                            ← 1.5秒【後】
```

**失敗を宣言した1.5秒後に「アンマウントした」とログが出ています。**

判定の瞬間、ボリュームは**実際にはマウントされていた**わけです。これは本当の切断ではなく、**パスの照合に失敗している**だけでした。

さらに注目すべきは、**同じパスに対する `Checking identity` と `Checking role` は成功している**ことです。同一の文字列を扱っているはずなのに、APIによって成否が分かれています。

## 真因: マウントパスがNFDだった

ログに出るURLエンコード形式のパスに、答えが書いてありました。

```
file:///Volumes/mymac%E3%81%AE%E3%83%8F%E3%82%99%E3%83%83%E3%82%AF%E3%82%A2%E3%83%83%E3%83%95%E3%82%9A/
```

デコードするとこうなります。

| バイト列 | 文字 |
|---|---|
| `E3 81 AE` | の |
| `E3 83 8F` | **ハ** |
| `E3 82 99` | **゙**（結合用濁点 U+3099） |
| `E3 83 83` | ッ |
| `E3 82 AF` | ク |
| `E3 82 A2` | ア |
| `E3 83 83` | ッ |
| `E3 83 95` | **フ** |
| `E3 82 9A` | **゚**（結合用半濁点 U+309A） |

つまり `バ` が `ハ + ゙`、`プ` が `フ + ゚` に**分解された NFD 形式**です。文字数を数えると差が出ます。

```python
>>> len(s)                                    # NFD
29
>>> len(unicodedata.normalize('NFC', s))      # NFC
27
```

`backupd` の `MountPointValidity` はマウントテーブルとパス文字列を照合しますが、ここで正規化形式が食い違うと一致しません。一方、`Checking identity` / `Checking role` はNSURL系のAPIを使っており、そちらは正規化して比較するため成功します。**ログが矛盾して見えたのはこのためです。**

照合に失敗すると `UnmountChecker` が「ディスクイメージが切断された」とみなし、`BACKUP_FAILED_DISCONNECTED_DISK_IMAGE (70)` を返します。

## 対処

ボリューム名をASCIIに変えるだけです。

```bash
# sparsebundleをアタッチ（/Volumes/.timemachine 配下はroot専用）
sudo hdiutil attach -nobrowse -noverify \
  "/Volumes/.timemachine/<NAS名>._smb._tcp.local./<UUID>/TimeMachine/<Mac名>.sparsebundle"

# ボリューム名をASCIIへ
sudo diskutil rename disk38s1 TMBackup

# 切り離してからバックアップ開始
sudo diskutil unmount disk38s1
sudo hdiutil detach disk37
sudo tmutil startbackup
```

適用後、初めて `MountingDiskImage` を越えて `Copying` に到達しました。

```
   6s Running=1 BackupPhase = PreparingSourceVolumes
  36s Running=1 BackupPhase = FindingChanges
  42s Running=1 BackupPhase = Copying          ← 到達
 300s Running=1 BackupPhase = Copying          ← 継続
```

### その後: 初回フルバックアップが完走しました

念のため追記します。**約25時間後に完走しました。**

```
$ tmutil listbackups
/Volumes/.timemachine/.../2026-08-04-075644.backup

$ defaults read /Library/Preferences/com.apple.TimeMachine.plist Destinations
RESULT = 0;          ← 半年ぶりに 70 から 0 へ
```

| | |
|---|---|
| 所要時間 | **約25時間**（Wi-Fi 5GHz 経由） |
| 実データ量 | **1.07 TB** |
| NAS上のサイズ | 1.1 TB / 609バンド |

**リネーム以外は何も変えていません。** 途中で試した `nsmb.conf` の調整も、暗号化の設定も、結局は関係ありませんでした。

なお速度は**ファイル数に律速**されていました。約100ファイル/秒で頭打ちになるので、大きいファイルの区間は一気に進み、小さいファイルの区間で遅くなります。**2回目以降は差分なので数分で終わります。**

:::message alert
**sparsebundleを作り直すと日本語名に戻ります。**作成のたびにリネームが必要です。
:::

## ログを読むときの注意

調査中、ログには大量の `Permission denied` が出ます。

```
[TimeMachine:TMDisk] attrVolumeWithMountPoint 'file:///Volumes/.timemachine/.../TimeMachine/'
  failed, error: Error Domain=NSPOSIXErrorDomain Code=13 "Permission denied"
```

一見すると権限問題ですが、**これはノイズです。**出力元は `TimeMachineSettings` と `SystemUIServer` で、いずれも一般ユーザー権限のプロセスです。`/Volumes/.timemachine/` 配下はroot専用マウントなので、これらが弾かれるのは正常な挙動です。

実際にバックアップを行う `backupd` はrootで動くため、この権限エラーとは無関係です。**私はしばらくこれに引っ張られて時間を溶かしました。**

プロセス名で絞ると本質だけが見えます。

```bash
sudo log show --predicate 'process == "backupd"' \
  --start "2026-08-02 07:25:00" --end "2026-08-02 07:25:10" \
  --info --debug --style compact | grep -v "com.apple.xpc:connection"
```

## 同じ症状に当たったときの切り分け手順

1. **失敗後に `diskutil list` と `mount` を確認する**
   ディスクイメージとSMB共有が生きていれば、ネットワーク断ではない
2. **`diskutil mount <dev>` で手動マウントして放置する**
   消えなければボリュームとSMBは無実。Time Machine固有の問題と確定できる
3. **`backupd` のログを時系列で並べる**
   失敗判定の後にアンマウントが出ていれば、切断ではなく照合の失敗
4. **マウントパスに非ASCII文字が含まれていないか確認する**
   含まれていればリネームを試す

「毎回きっちり同じ秒数で、同じエラーコードで落ちる」場合、環境要因（ネットワークやディスク）ではなく**決定論的な処理の問題**を疑うのが近道です。

## おわりに

エラーメッセージは「ディスクイメージが切断された」でしたが、実際には何も切断されていませんでした。エラーメッセージが指し示す方向に原因が無いことは、珍しくありません。

この件で私は4回誤診しました。特に暗号化状態の不一致は、証拠が揃って見えるだけに厄介でした。**「もっともらしい不一致を見つけた」ことと「原因を特定した」ことは違う**、というのが一番の教訓です。

否定できる実験を先に組むこと。それだけで、4回のうち3回は避けられたはずです。

---

## この手の失敗を55件ぶん書きました

この記事で書いた「**もっともらしい不一致を見つけた ≠ 原因を特定した**」という失敗は、私の自宅ラボ環境で繰り返し起きています。

同じような地雷を55件、**症状から逆引きできる形**で本にまとめました。

https://zenn.dev/nomunomu0504/books/homelab-proxmox-zero-to-100

Proxmox + LXC 13台の環境を組むなかで実際に踏んだものだけを載せています。この記事と同じく、**私が誤診して撤回した過程まで含めて**書きました。

本の付録では、55件を8つの「型」に整理しています。

| 型 | 例 |
|---|---|
| **設定済みに見えるのに機能しない** | `startup: order=` があるのに `onboot: 1` が無い |
| **エラーにならないのに動いていない** | APIが `errorCode: 0` で0件を返す |
| **半分だけ動く** | 内部名は引けるが外部名だけタイムアウト |
| **検証方法そのものが間違っていた** | ← **この記事はここに該当します** |

**第0章（設計の5原則）と更新履歴は無料**で読めるので、合うかどうかはそこで判断できます。
