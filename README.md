# sil-gitai

SILと義体を、長期に育て、別の本体や身体にも移せるようにする共同開発リポジトリ。

## 現在の参照資料

- **[SIL–BODY 接続仕様 v0.1 rev4（最新版の下書き）](docs/specs/Interface%20spec%20v0.1%20rev4.md)** — 接続プロトコル、物理ARM、安全状態、初期試験の通信経路
- [移植可能なオフラインSIL 設計方針 v0.1](docs/design/portability-design-v0.1-draft.md) — モデル・記憶・個性・義体制御・実行環境の分離
- [GITAI-01 物理ARMと非常停止の設計案](docs/hardware/physical-arm-button-and-estop-design-v0.1.md) — **GPIO・回路は候補。未配線・未検証**
- [ESP32ファームウェア公開前チェックリスト](docs/checklists/esp32-firmware-publication-preflight.md) — Wi-Fiパスワード等の秘密点検
- [rev3 ハードレビュー](docs/reviews/interface-spec-v0.1-rev3-wired-policy-review-2026-11-05.md)

## 決定済みの方針（2026-11-05）

1. **ARMはESP32に直結した物理ボタンのみ。** SIL・HTTP・シリアルからARMしない。
2. **v0.1の初期安全試験では、動作指令はUSBシリアル等の有線のみ。** Wi-Fiからの動作・無認証の書き込みは受け付けない。この有線限定は、実装と検証が容易な初期段階の**暫定方針**であり、将来の認証済み無線接続を排除しない。
3. **これからの実験・設計は移植前提。** SIL本体と義体の接続仕様を媒体非依存に保つ。はじめに有線で実験することは、本体・義体を永久にケーブルで結ぶという意味ではない。
4. SILは動作を依頼し、BODY（ESP32）が安全条件に基づき最終判断する。物理非常停止は独立に確保する。

**重要：** 現行ESP32ソースの照合・秘密情報の点検・実機での安全確認は、まだ完了していません。文書は設計案であり、動作保証や実装済み機能の一覧ではありません。

以前の [rev3下書き](docs/specs/interface-spec-v0.1-draft.md) や [rev2版](docs/specs/interface-spec-v0.1-rev2.md) は参照用の過去版です。最新版の参照先は上記のrev4です。

共同レビュー記録：[PR #1](https://github.com/YFujisaka-SIL/sil-gitai/pull/1) ／ [PR #2](https://github.com/YFujisaka-SIL/sil-gitai/pull/2)
