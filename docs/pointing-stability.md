# LiNEA40 と比較した Zonkey のポインティング設定

比較元は Zonkey `57d6a81cc4439791d1f76e140767938e7e8725a1` と
LiNEA40 `43d3ac4b988189fad51ac9cb3a4dffaebc641e97`。
実機での遅延測定はまだ行っていないため、以下は設定と実装から判断した改善候補です。

| 項目 | LiNEA40 | 変更前の Zonkey | この変更 |
| --- | --- | --- | --- |
| ZMK | v0.2.1 | v0.3 | v0.3 |
| PMW3610 ドライバー | inorichi/main | kureyakey/main | kureyakey の確認済みコミットに固定 |
| ポーリング | 125 Hz、ソフトウェアで2サンプルを合算 | 250 Hz | センサー側で125 Hz |
| 自動マウスレイヤー | 無効（`automouse-layer` 未指定） | レイヤー6、タイムアウト0 ms | 無効（`automouse-layer` 未指定） |
| 手動マウスレイヤー | `&lt 5 LANG2` によるレイヤー5 | `&lt 6 LANGUAGE_2` によるレイヤー6 | 既存のレイヤー6を使用 |
| CPI / 除数 | 800 / 1 | 1600 / 1 | 1600 / 1 |
| スクロール閾値 | 2 | 32 | 32 |
| センサー常時 RUN | 無効 | 有効 | 有効 |
| ホスト向け BLE 接続間隔の希望値 | ZMK既定値 | 7.5〜15 ms | 7.5〜15 ms |
| ホスト向け BLE peripheral latency | ZMK既定値 | 0 | 0 |
| ホスト向け BLE supervision timeout | ZMK既定値 | 1秒 | ZMK v0.3 既定の4秒 |

## 変更の理由

### レイヤー切り替えの繰り返しを取り除く

ドライバーは `automouse-layer > 0` で自動切り替え処理を組み込みます。
移動時にレイヤーを有効化して、`CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS` 後に
タイマーで解除する実装です。`0` は無効化の指定ではありません。
従来は閾値を超える移動のたびに有効化と直後の解除が発生し得る設定で、
クリック用キーマップとレイヤー色の切り替えを繰り返す原因になります。
解除処理はタイマー割り込みからレイヤー変更イベントを発行しています。

LiNEA40 と同様に `automouse-layer` を省略し、ドライバーの自動切り替えを
コンパイル対象から外します。クリックは既存の `&lt 6 LANGUAGE_2` を保持して行います。
タイマーが手動レイヤーを解除することもなくなります。
V キーの長押しによるレイヤー5のトラックボールスクロールも引き続き使用できます。

### BLE に渡すレポートの頻度を抑える

Zonkey の250 Hzは4 msごとの設定です。ホスト向けBLE接続間隔の希望値は
7.5〜15 msのため、1接続イベントまでに複数のレポートが発生し得ます。
ZMK v0.3 はマウスレポートをキューに積み、満杯なら最大100 ms待機する実装です。
実際のキュー詰まりは未計測ですが、125 Hzへ下げることで送信負荷を減らします。
1接続イベントで送れるパケット数やホストの判断にも依存するため、
250 Hzだけを遅延の確定原因とはしていません。

LiNEA40 の `125_SW` はセンサーを250 Hzで動かし、2サンプルを合算してから
送信します。停止直前などに1サンプルだけ残ると、それを送るタイマーがありません。
この挙動まで取り込まず、Zonkey ではドライバーが提供するセンサー側125 Hzを選択します。
これは8 msの設定であり、実際のホストへの更新間隔はBLE接続にも依存します。
感度、軸方向、GPIO、スクロール閾値は機種固有のためコピーしていません。

### 一時的な電波状態の悪化による切断を減らす

`CONFIG_BT_PERIPHERAL_PREF_TIMEOUT` の単位は10 msです。
100（1秒）からZMK v0.3の既定値400（4秒）へ戻します。
これは切断判定の猶予であり、通常の入力を4秒遅延させる設定ではありません。
接続間隔の希望値と peripheral latency 0 は維持します。
実際の接続パラメーターはホストとの交渉で決まります。
左右間接続のパラメーターとは別の設定です。

### ドライバーを固定して再現可能にする

`west.yml` で確認済みの `9f5b7ac0509ac353826c9634826814056a4be168` に固定します。
ビルドのたびに外部ドライバーの `main` の変更が混ざることを防ぎます。
センサー常時 RUN は維持しているため、LiNEA40 と同じ電池持ちになるとは限りません。

## 実機での確認

1. 同じビルドの `zonkey_R rgbled_adapter` と `zonkey_L rgbled_adapter` を左右へ書き込みます。
   `settings_reset` はペアリング情報を消すため、通常の更新では書き込みません。
2. まず右手側をUSB接続して、ゆっくりした移動、素早い往復、細かな位置合わせを試します。
   動作を止めた後にカーソルが流れ続けないかも確認します。
3. 同じホスト・距離・OSのポインター設定でBLE接続でも同じ操作を試し、
   連続操作中にカクつきが蓄積しないか確認します。
4. 左手側の `&lt 6 LANGUAGE_2` を保持し、J / K / L の左・中・右クリックを試します。
   ボールを止めて1秒以上保持してもクリック用レイヤーが解除されないか確認します。
   離すと通常の文字入力へ戻り、ボールだけを動かしても自動切り替えしないことを確認します。
5. V を保持して縦横スクロールし、通常移動との往復、左手側のエンコーダー、
   マウス操作中の左右のキー入力も確認します。
6. 数分放置してからの操作再開、30分以上のBLE連続使用、電源再投入後の再接続を確認します。
   USBでも問題が残る場合はセンサー・ボール・電源を、BLEだけの場合は接続条件も切り分けます。

## 確認したソース

- [LiNEA40 のセンサー設定](https://github.com/1Q3dayo/zmk-config-LiNEA40/blob/43d3ac4b988189fad51ac9cb3a4dffaebc641e97/config/boards/shields/LiNEA40/LiNEA40_right.conf)
- [LiNEA40 のオーバーレイ](https://github.com/1Q3dayo/zmk-config-LiNEA40/blob/43d3ac4b988189fad51ac9cb3a4dffaebc641e97/config/boards/shields/LiNEA40/LiNEA40_right.overlay)
- [PMW3610 の設定定義](https://github.com/kureyakey/zmk-pmw3610-driver/blob/9f5b7ac0509ac353826c9634826814056a4be168/Kconfig)
- [PMW3610 のポーリング・レイヤー処理](https://github.com/kureyakey/zmk-pmw3610-driver/blob/9f5b7ac0509ac353826c9634826814056a4be168/src/pmw3610.c#L555-L749)
- [PMW3610 のセンサー設定値](https://github.com/kureyakey/zmk-pmw3610-driver/blob/9f5b7ac0509ac353826c9634826814056a4be168/src/pmw3610.h#L106-L120)
- [ZMK v0.3 のBLE既定値](https://github.com/zmkfirmware/zmk/blob/edf5c0814fd3ea202e43aad2d68fd32e882a518c/app/Kconfig#L208-L235)
- [ZMK v0.3 のマウス送信キュー](https://github.com/zmkfirmware/zmk/blob/edf5c0814fd3ea202e43aad2d68fd32e882a518c/app/src/hog.c#L413-L460)
- [Zephyr 3.5 のタイマー](https://docs.zephyrproject.org/3.5.0/kernel/services/timing/timers.html)
