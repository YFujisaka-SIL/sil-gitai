# GITAI-01 物理ARMボタン・駆動許可・緊急停止の設計案 v0.1

- 作成：チャッピー（ハード担当）、2026-11-05
- 対象：現行 GITAI-01（ESP32 / MG996R 首サーボ）
- 決定済み：**ARMはESP32に直結した物理ボタンのみ。HTTP、シリアル、SILのテキスト命令からARMできない。**（洋輔、2026-11-05）
- ステータス：**提案・実装前。現行ファーム未入手／実機のピン割当・電源配線未検証**。この文書をそのまま結線して通電しないこと。

参照：`docs/specs/interface-spec-v0.1-rev2.md`、`docs/hardware-safety-and-interface-inventory.md`。

## 1. ARMとE-stopは別の入力／別の機能

| 項目 | ARM（通常の動作許可） | 物理非常停止（E-stop） |
|---|---|---|
| 目的 | 人が動作開始を許可する | 通信やESP32の状態に依存せずサーボ駆動を遮断 |
| 操作器 | モーメンタリ押しボタン（通常開 NO） | ラッチ式押しボタン（通常閉 NC、解除は手操作） |
| ESP32との関係 | GPIO入力を読み取り、状態遷移を判断 | **GPIOによる停止のみには依存しない**、駆動電源の遮断経路を持つ |
| 操作後 | 条件が揃った新規押下でのみ `DISARMED → ARMED` | `FAULT` としてラッチ、解除後も動作を再開しない |

## 2. ARMボタンの仮配線（基板・現行コード照合前の案）

候補：**GPIO27**（一般的な ESP32 DevKit / ESP32-WROOM の場合）。確定ピンではない。GPIO18（過去記録のサーボPWM）と GPIO21/22（I2C）は避ける。ブートストラップや書込み用のピンをARMに使わない。使用中のESP32型番・基板ピン配置・実配線を照合してから確定する。

```text
ESP32 GPIO27  ──────[ NO モーメンタリ ARM ボタン ]────── GND
    ↑
  INPUT_PULLUP（ESP32内部3.3Vプルアップ）
```

- **GPIOに5Vを接続しない。** 3.3Vロジック、押しているときLOW。
- ボタンが外れている／配線が切れているときはHIGHで「押されていない」側にする。
- 電源投入時に押されていたボタンは無視し、**いったん放してから改めて押す**必要がある。
- 少なくとも 30〜50ms のデバウンスと、立下がり（HIGH→LOW）の新規押下イベントを使う。長押しや押しっぱなしで再アームしない。
- 押下時はセッション存在、心拍新鮮度、E-stop解除、`FAULT`なし、電源・制御の自己点検、動作範囲設定済みを確認。条件不足なら**動作しない**。
- **注意：1本のGPIO入力は断線に対しては非動作側になっても、GND短絡で押下と誤認する可能性がある。** 人身安全を保証する冗長な安全回路ではない。必要に応じて二重化や監視型のスイッチへ拡張する。

状態表示LEDは任意：GPIO26（仮） → 330Ω程度の抵抗 → LED → GND。役割案：消灯 DISARMED、点灯 ARMED、点滅 FAULT。ピン競合確認後に実装。LEDは安全状態の証明ではない。

## 3. サーボ電源の駆動許可／物理非常停止（概念設計）

ESP32のGPIOだけで電源遮断を保証しない。**E-stopスイッチを物理的な駆動遮断経路に入れる**。

ベンチ試作用の概念回路（部品・電気定格確定前、配線指示ではない）：

```text
外部サーボ用 5V 電源 ＋
    │
 [適切なDCヒューズ]
    │
 [通常開 NO リレーの負荷接点] ─────── MG996R V+
    │            （上図は直列の概念図）
    └───────────────────────────────

リレーコイル用電源 ＋ ── [ラッチ式 E-stop NC] ── [リレーコイル] ── [駆動トランジスタ] ── GND
                                                           ↑
                                                ARM有効時のみ出力（仮 GPIO25）
                                                デフォルトOFFのプルダウン
```

上図は電源経路とコイル制御を別々に表す模式図で、部品配置図ではない。

- リレーは**非励磁でサーボ電源OFF**とする。ESP32起動直後は制御GPIO LOW／ハイインピーダンス＋外付けプルダウンでOFF。
- 物理E-stopを押すと、ESP32が暴走していてもコイル回路が切れ、NO接点が開く。**接点溶着などの故障までは、この単純な構成では保証できない**。
- E-stop解除だけではESP32が自動ARMしない。リレー制御も明示的な再ARMなしにはONにしない。
- リレーのDC接点定格、サーボの**起動・ストール電流**、コイル電圧、ヒューズ選定、逆起電力保護（フライバックダイオード等）、電源余裕を実測・部品型番で検証する。
- ESP32は別の安定した電源（例：USB）で動作を継続させ、サーボ側電源GNDとの共通接続と、信号線からの逆流・意図しない給電に注意する。電源OFF中はPWM出力も無効にする。
- 首が電源断で倒れたり噛み込んだりしないことを機構側で確認する。保持が必要なら別途フェイルセーフ機構を検討。
- このベンチ試作のリレー／単一スイッチ案は**産業用の安全カテゴリ認証や人に接触する移動機械の安全機能を満たしたものではない**。四輪・HAND-01には安全設計を作り直す。

※ 最小構成としてサーボV+直列の定格適合NC E-stopも検討できるが、解除直後の再給電でサーボが予期せず動く危険があるため、**デフォルトOFFの駆動許可**を別に設ける設計を優先する。

## 4. ファームウェア実装要件（擬似コード）

```text
boot:
  actuator_enable = OFF
  servo_pwm = DISABLED
  state = DISARMED
  arm_button_must_be_released_first = TRUE
  session = NONE

session_open(controller):
  require state == DISARMED && session == NONE
  create sole session; keep actuator_enable OFF

valid_heartbeat(session):
  update heartbeat timestamp only when session matches

physical_arm_button_new_press:
  require button_was_released_since_boot
  require state == DISARMED
  require session != NONE
  require recent_valid_heartbeat(session)
  require E_stop_released && !fault_latched
  require GPIO/output/supply/config checks pass
  then state = ARMED, actuator_enable = ON
  (safe PWM initialization must precede any possible servo motion)

move(request):
  require state in {ARMED, RUNNING}
  require same session, recent heartbeat, E_stop_released
  require target/velocity inside measured hard limits
  apply ESP32-side motion rate limits

heartbeat_expired OR disarm:
  cancel target; inhibit PWM/drive; actuator_enable = OFF
  session = NONE; state = DISARMED

physical_E_stop OR soft_estop OR hardware_fault:
  actuator_enable = OFF
  cancel target; session = NONE; latch FAULT
  (physical E-stop also opens the power circuit independent of code)

fault_recovery:
  must be local/manual and only after cause removed
  return to DISARMED, not ARMED
  require a new session + new button press to ARM again
```

**実装順序の注意**：ソフトだけでは、ESP32完全停止時のPWM固着・GPIO固着を保証できない。別の物理遮断手段を必須とする。ブート時にLOWにならない可能性があるピンは、リレー制御に選ばない。

## 5. 追加の状態遷移・仕様レビュー事項

- rev2の「`stop` が次の `move` で再開可能」という定義は、通常停止として妥当。一方「通信喪失・停止の後は必ず人が物理操作」という設計原則 §0.5 と衝突するため、**通常の `stop` を例外として明文化**する。
- `disarm` でセッション無効にするなら、次は `session_open` → 物理ボタン。 `FAULT` 解除の遠隔命令は導入しない。
- `session_open` を無認証の「誰でも」にすると、制御権の占有（妨害）が可能。v0.1では信頼されたローカル接続だけに制限する。
- rev2の256バイト長は**要求の最大長**としての案。`hello`等の応答・イベントは別の上限で実測する。
- `arm` はネットワーク／シリアルの命令一覧に**存在させない**。すべての旧APIも同一の安全ゲートで検証する。

## 6. 未確認の実機条件

- [ ] ESP32の具体的な基板型番、現在使用中のGPIO割当
- [ ] MG996Rの5V電源供給方法、電源・最大電流、実際の配線
- [ ] 首機構が無通電で安全に保持／脱力できるか
- [ ] 使用するARMボタン・E-stop・リレー・トランジスタの部品型番と定格
- [ ] サーボ信号線のOFF時状態とバックフィードの有無
- [ ] 旧API、現在の状態遷移、安全制御実装状況

## 7. 試験（必ず駆動部を外した低リスク試験から）

1. ボタンなし／電源投入時押下／接点チャタリング／短絡・断線で勝手にARMしないことを確認（短絡検出不可なら制限として記録）。
2. セッションなし、無効heartbeat、E-stop押下、FAULT状態、再起動直後はARM拒否。
3. 正常セッションと有効heartbeatで**人が新しく押したときだけ**ARM。
4. Wi-Fi切断・通信途絶・他SILのheartbeatで動作許可が維持されないこと。
5. E-stop押下でリレー接点からサーボ側の電源が物理的に落ちること（無負荷測定から開始）。
6. 解除だけではサーボ電源・動作が戻らないこと。
7. 最後に、首の干渉・過熱・電源降下を見ながら低速試験。

**この文書は設計案であり、安全確認済みの配線図でも動作保証でもない。**
