# 設計書: 常時録画・録音システム（RaspiSC）

**作成者**: Codex  
**作成日**: 2026-01-16  
**対象**: Raspberry Pi 4B 常時録画・録音システム

---

## 1. 目的

Raspberry Pi 4Bで24時間稼働する監視カメラシステムを構築し、映像・音声を1時間単位で保存する。保存先はUSB（短期保持）とGoogleドライブ（長期保持）とし、安定性と再現性を優先する。

---

## 2. 前提条件（確定事項）

- 録画方式: 常時録画・録音（コンビニ方式）
- 画質: 低画質（容量優先、640x480程度）
- 音質: 低音質（モノラル、16kbps程度）
- 分割: 1時間ごと
- 保存: USB 7日、Googleドライブ 30日
- Googleドライブ保存先: フォルダID `1F9CAp4YEuw6OGNctM_w20DSgmRH7eKZb`（RaspiSC_data）
- Googleドライブのサブフォルダ規則: `YYYY-MM-DD`

---

## 3. 全体アーキテクチャ

### 3.1 コンポーネント

1. **録画・録音プロセス**
   - 1時間単位で映像+音声を1ファイルに統合して出力
2. **ローカル保存（USB）**
   - `/mnt/usb/recordings` 配下に日付フォルダを作成
3. **アップロード**
   - rcloneでGoogleドライブへアップロード
4. **ローテーション**
   - USB: 7日超過ファイルの削除
   - Googleドライブ: 30日超過ファイルの削除
5. **監視/復旧**
   - systemdでプロセス監視と自動再起動

### 3.2 処理フロー（概要）

1. 録画開始（1時間区切り）
2. 録画完了後にローカルへ保存
3. 保存完了後にGoogleドライブへアップロード
4. ローテーション処理を定期実行
5. 失敗時は再試行または翌回で補填

---

## 4. 保存設計

### 4.1 USB保存

- 保存先: `/mnt/usb/recordings/YYYY-MM-DD/`
- 例: `/mnt/usb/recordings/2026-01-16/`
- ファイル命名: `RaspiSC_YYYYMMDD_HH00.mp4`
  - 例: `RaspiSC_20260116_1300.mp4`

### 4.2 Googleドライブ保存

- 保存先フォルダID: `1F9CAp4YEuw6OGNctM_w20DSgmRH7eKZb`
- サブフォルダ: `YYYY-MM-DD`
- 例: `RaspiSC_data/2026-01-16/`

---

## 5. Googleドライブ連携方式

### 5.1 採用方式

- **rclone** を採用
  - 安定運用、CLI連携が容易
  - フォルダIDによる固定指定が可能

### 5.2 設定方針

- rcloneのGoogle Driveリモート設定で**root folder ID**を固定
- リモート名: `gdrive_raspisc`（仮）
- 以降の操作は `gdrive_raspisc:` を起点とする

### 5.3 アップロード方針（設計）

- アップロード先: `gdrive_raspisc:YYYY-MM-DD/`
- 1時間ファイル生成後に逐次アップロード
- 失敗時はローカルに残して次回再試行

---

## 6. ローテーション設計

### 6.1 USB側

- 基準: 7日超過ファイルを削除
- 実装: `find /mnt/usb/recordings -type f -mtime +7 -delete`
- 失敗時: 次回のローテーションで補正

### 6.2 Googleドライブ側

- 基準: 30日超過ファイルを削除
- 実装: rcloneの`--min-age 30d`を活用した削除処理
- 例: `rclone delete gdrive_raspisc: --min-age 30d --include "RaspiSC_*.mp4"`

---

## 7. エラーハンドリング

### 7.1 録画失敗

- systemdで録画サービスを監視し、自動再起動
- 連続失敗時はログに記録し、次の録画サイクルで再開

### 7.2 ストレージ不足

- USB: 7日ローテーションを優先的に実行
- それでも不足する場合は古いファイルから削除

### 7.3 ネットワーク障害

- アップロード失敗時はローカルに残す
- 復旧時に未アップロード分を再送

---

## 8. 24H稼働の安定性対策

- systemdで録画プロセスを常駐化
- タイマーでアップロード/ローテーションを定期実行
- 温度監視は将来拡張（設計に備考として記録）

---

## 9. 実装指示（Claude Code向け要点）

- rcloneリモート設定でroot folder IDに `1F9CAp4YEuw6OGNctM_w20DSgmRH7eKZb` を指定
- Googleドライブの保存先は `gdrive_raspisc:YYYY-MM-DD/`
- ローカル保存は `/mnt/usb/recordings/YYYY-MM-DD/`
- ファイル命名: `RaspiSC_YYYYMMDD_HH00.mp4`
- ローテーション: USB 7日、Googleドライブ 30日

---

## 10. 未確定・確認事項

- 音声品質（16kbps想定）の実測値
- 1時間あたりの実際容量（190MB/時以内の達成確認）
- 監視通知（必要性の有無）

