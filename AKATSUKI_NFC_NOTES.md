# akatsuki (SO-01K 系 / SDM845) DOCOMO版 NFC 調査メモ

LineageOS 22.2 (`android_kernel_sony_sdm845` lineage-22.2) 上での、CXD224x (carillon) NFC の調査記録。
状態: **カーネル側は動作。通常 NFC はユーザー空間(HAL)で未解決。FeliCa は対象外。**

## 結論の要約

| 層 | 状態 |
|---|---|
| カーネルドライバ (`CONFIG_NFC_CARILLON`) | OK |
| Device tree / dtbo | OK(boot・vendor と同じビルドの dtbo.img を書き込めば解決) |
| `/dev/cxd224x-i2c` | 生成される |
| NXP HAL (`android.hardware.nfc-service.nxp`) | 起動するが、NFCC から応答なし(Read Timeout) |
| NFC トグル | ON にしても OFF に戻る |

## 経緯と確認事項

1. **当初の症状**: `cxd224x-i2c 0-0029: prop pinctrl-0 index 0 invalid phandle` / `probe ... failed with error -22`。
2. **ソース側の確認** (lineage-22.2 の `arch/arm64/boot/dts/somc/`)
   - `somc-tama-nfc_carillon.dtsi`: GPIO63 = INT, GPIO62 = WAKE, `sdm_gpio_63` を参照。
   - `sdm_gpio_63` は `sdm845-tama-common.dtsi` で定義され、`sdm845-tama-akatsuki_jp-common.dtsi` で pull-up に上書き。
   - `sdm845-tama-akatsuki_dcm.dtsi` / `_kddi.dtsi` は末尾で carillon の dtsi を include 済み。
   - `tama_akatsuki_kddi_defconfig` に `CONFIG_NFC_CARILLON=y`。DCM 側 defconfig は未確認。
   - したがって SO-02K パッチ(MSM8998 の GPIO92/93、clk3)を移植する必要はなかった。`felica_clk` が無い (`XTAL should be used`) のは想定どおり。
3. **根本原因**: dtbo.img を自前ビルドせず、古い dtbo が残ったまま新しい boot/vendor を使っていた。dtbo.img を書き込むと probe が成功した。
   ```
   cxd224x_probe, irq_gpio(63) / wake_gpio(62) / irq(320)
   cxd224x_probe, probing cxd224x driver exited successfully
   /sys/firmware/devicetree/base/__symbols__/sdm_gpio_63 が存在
   ```
4. **残る問題 (HAL)**
   - 起動する HAL は NXP 用 (`nfc-service-nxp.rc`, `INxpNfc`)。
   - `/vendor/etc/libnfc-nxp.conf`: `NXP_NFC_DEV_NODE="/dev/pn553"`。
   - `/dev/pn553` から `/dev/cxd224x-i2c` へのシンボリックリンクで open はできた。SELinux・権限は原因ではない。
   - その後 `NxpTml` が `Read requested` → 2秒後 `Read Timeout` / `NFCC - Error in Read` を繰り返す。`NxpHal` の CORE_RESET 等は出ない。`NfcService: NFC is not initialized yet`。
   - dmesg に `cxd224x` の read/write/ioctl の行は出ていない(ドライバのデバッグ出力が無効なだけの可能性もある)。

## 未確認の仮説(原因候補)

1. NXP HAL は open 後に PN5xx 系 ioctl で VEN を制御して NFCC を起動するが、CXD224x ドライバにはその ioctl が無い。DT でも `rst_gpio(0)` / `ven_gpio(0)` で該当 GPIO が無い。
2. CXD224x の起動に Sony 独自の手順・コマンドが必要。
3. 初期化に TA 領域の鍵など Sony 独自のセキュアな仕組みが必要。これだと BL アンロック後は無理。

## 重要な注意

- 既存の `0001-akatsuki-Fix-NFC-FeliCa-support-for-SO-01J-DCM.patch` は、存在しない defconfig 名 (`tama_akatsuki_dcm_defconfig` を BoardConfig.mk に設定) に書き換えるだけで、**動作する修正ではない**。適用しないこと。
- カーネルへの追加パッチは現時点で不要(SO-02K パッチからの移植対象は無かった)。
- FeliCa(おサイフケータイ)は BL アンロックで TA が消えるため、目標外。
- LineageOS で NFC が不可とされる点についての一般論(Web 検索の要約のみ。一次情報は未確認): 標準 NFC は動く場合があるが FeliCa は不可。

## 次にやること(再開時)

1. ストック国内版の `/vendor` から NFC 関連を確認する。
   ```
   ls -l /vendor/bin/hw /vendor/lib64 /vendor/etc | grep -i -E "nfc|felica|cxd|nxp"
   ```
   特に `/dev/cxd224x-i2c` を開く Sony 製ライブラリ/HAL、init rc、`libnfc` 設定を探す。
2. 見つかれば、通常 NFC 用に Sony 製 HAL を流用できるか検討する。
3. 見つからない場合の小実験: carillon ドライバに NXP HAL 互換の ioctl を追加して Read の応答が出るか確認(効果は不明)。
4. 実現性が低いと判断できたら調査を終了する。

## 実機で使った確認コマンド

```
dmesg | grep -i -E "cxd|carillon|bd7602|felica"
ls /sys/firmware/devicetree/base/__symbols__ | grep 63
ls -l /dev | grep -i -E "cxd|felica|nfc"
grep DEV_NODE /vendor/etc/libnfc-nxp.conf
logcat -b all -d | grep -i -E "NxpTml|NxpHal|NfcService"   # NFC を OFF→ON した直後
```
