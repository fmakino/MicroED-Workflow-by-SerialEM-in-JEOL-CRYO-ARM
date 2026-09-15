# SerialEMによるMicroED連続回転測定

[English](README.md)

JEOL CRYO ARMでSerialEMを用いてMicroED連続回転測定を行うためのスクリプトと、簡潔な測定手順です。

> [!IMPORTANT]
> 本文書は完全な装置操作マニュアルではありません。アライメント、キャリブレーション、安全確認、カメラ、Low Dose、線量などは施設ごとに設定・検証してください。元マニュアルの数値は特定のCRYOARM200/Rio環境の例であり、他施設へそのまま適用しないでください。

## ファイル構成

```text
.
├── README.md
├── README_ja.md
└── JEOL_microED_Osaka-workflow.txt
```

`JEOL_microED_Osaka-workflow.txt`には5つのmacroがあります。

- `MicroEDContinuousRotation`: `TiltDuringRecord`による連続回転測定
- `BeforeTilt`: 測定前のflash/refill確認、Navigator itemへの再アラインなど
- `AfterTilt`: View画像の保存と、測定後の状態復帰
- `ScreeningShot`: Previewで回折像を撮影し、JPEGを保存
- `ScreeningShotDamage`: Preview条件で180秒の連続露光を行い、damage確認用データとJPEGを保存

## 1. ユーザーが確認・変更する設定

### MicroEDContinuousRotation

| 変数 | 内容 | 配布値 |
|---|---|---:|
| `angleBegin` / `angleFinish` | 開始角度 / 終了角度（度） | `-30` / `30` |
| `cameraBinning` | 非DEカメラのbinning | `2` |
| `secondsPerFrame` | 非DEカメラのframe time（秒） | `0.5` |
| `degreesPerSecond` | 回転速度（度/秒）と測定時間計算 | `1.0` |
| `stageRateIndex` | `TiltDuringRecord`へ渡すJEOL stage speed index | `102` |
| `preAcquisitionWait` | Screenを上げた後の待ち時間（秒） | `2` |
| `stageSettlingTime` | tilt後の静定時間（秒） | `0` |
| `cameraID` | SerialEM上のcamera ID | `1` |
| `recordPreset` | 使用するCamera Parameter Set | `R` |
| `processingSetting` | `SetProcessing`へ渡す値 | `2` |
| `useDECamera` | DEなら`1`、非DEなら`0` | `0` |
| `deFramesPerSecond` | DE使用時のframe rate（fps） | `20` |

総測定時間は`ABS(angleFinish - angleBegin) / degreesPerSecond`です。非DEでは`secondsPerFrame`を`SetFrameTime`で設定します。DEではbinningを2にし、`deFramesPerSecond`を設定します。camera ID、binning、processing、frame保存、`degreesPerSecond`と`stageRateIndex`の対応は施設ごとに確認してください。

### BeforeTiltのflash/refill設定

| 変数 | 内容 | 配布値 |
|---|---|---:|
| `FlashInterval` | C-FEG flash間隔（秒） | `8 * 3600` |
| `flash_when_refill` | refill時にflashする場合は`1` | `1` |
| `delay_after_refill` | refill後の待ち時間（分） | `5` |
| `darkRef_when_refill` | 定義済みだが本ファイル内では未使用 | `0` |
| `update_dark` | 定義済みだが本ファイル内では未使用 | `0` |

`BeforeTilt`は外部の`EMProperties`と`Funcs::CustomMoveStage`を呼びます。このリポジトリには含まれません。`LongOperation FF`、`LongOperation RS/RT`、`AreDewarsFilling`を含め、対象装置で検証してから自動flash/refillを使用してください。

### ScreeningShotDamage

| 変数 | 内容 | 配布値 |
|---|---|---:|
| `binning` | Previewのbinning | `2` |
| `totalExposureTime` | 総露光時間（秒） | `180` |
| `expTimeperFrame` | 1 frameの露光時間（秒） | `1` |
| `currentCamera` | camera ID（元scriptの注記: `1` Rio、`2` K3） | `1` |
| `darkgain` | processing（元scriptの注記: `0` raw、`1` dark/gain、`2` gain-normalized） | `2` |
| `recordMode` | 使用するCamera Parameter Set | `P` |

camera IDとprocessingの対応は施設ごとに確認してください。

## 2. SerialEMへの登録

1. `JEOL_microED_Osaka-workflow.txt`をSerialEM PCへ置きます。
2. SerialEMで、登録先にする既存のscriptを**Edit**で開きます。
3. `JEOL_microED_Osaka-workflow.txt`をテキストエディターで開きます。
4. `MacroName MicroEDContinuousRotation`から対応する`EndMacro`までをコピーし、SerialEMのscript editorへ貼り付けます。
5. 同様に、`MacroName BeforeTilt`から`EndMacro`までを別のscriptとして貼り付けます。
6. `MacroName AfterTilt`から`EndMacro`までを別のscriptとして貼り付けます。
7. 同じ方法で`ScreeningShot`と`ScreeningShotDamage`も、それぞれ`MacroName`から`EndMacro`まで別のscriptへ貼り付けます。
8. 5つのscriptを保存し、それぞれのmacro名で実行できることを確認します。
9. `BeforeTilt`から外部scriptの`EMProperties`と`Funcs::CustomMoveStage`を呼べることを確認します。

各scriptは、`MacroName`の行から`EndMacro`の行までを1つのまとまりとしてコピーしてください。既存scriptの内容を上書きしないよう、登録先を確認してから貼り付けます。SerialEMのversionや施設設定により**Edit**の開き方や保存方法が異なる場合は、施設担当者の手順に従ってください。

## 3. 測定ワークフロー

### A. 測定前準備

1. GridをStageへLoadします。
2. DigitalMicrographを起動し、Rioを選択します。
3. SerialEMを起動します。別カメラに切り替わった場合はRioを選び直し、自動で始まったViewを停止します。
4. TEM Centerで必要なEmission、flashing、Auto Emissionを設定します。
5. View、Focus、Trial、Record、Preview、Mont-mapを確認します。RecordとPreviewではDiffraction modeとframe保存が必要です。具体値は施設ごとに設定してください。

### B. Atlas撮影とAtlas/View位置合わせ

1. Navigatorを開き、施設のGrid map用Imaging Stateを選びます。
2. Grid観察位置へ移動し、Upper Camera、Column Valve、focusを確認します。
3. 施設のAtlas macro（元マニュアルでは`TakeAtlas`）を実行し、日付・時刻を付けたfolderへ保存します。
4. Atlas上の目立つコンタミなどをlandmarkにしてNavigator pointを追加し、その位置へ移動します。
5. Trial/Viewで同じlandmarkをカメラ中央へ合わせ、View画像上の対応位置にmarkerを置き、**Navigator > Shift To Marker**を全itemへ適用します。

`TakeAtlas`、`CallLDMRecord`、Imaging State、倍率、絞り、alignment fileは本リポジトリに含まれません。施設ごとの設定が必要です。

### C. View montage

1. Atlas上で**Add Points**を押し、結晶がありそうなSquareを選びます。
2. **Stop Adding**を押し、対象pointを測定対象（`A`）にします。
3. **New file at group**を有効にしてmontageを選びます。元マニュアルでは3 × 3、MRC integer出力です。
4. **Navigator > Acquire at Items**でMappingを選び、montage保存、Navigator map作成、Low Dose View、各itemのRough Eucentricityを設定して開始します。
5. 完成したView montageを確認し、施設のStage Z許容範囲外のitemを除外します。

### D. ビーム・絞り調整

1. View montage上で結晶のないカーボン膜を選び、marker位置へ移動します。
2. Upper Cameraを入れ、Low DoseのTrialへ移動します。
3. 施設のデータ測定用CLAにし、必要なSTD focusとSerialEM defocus resetを行います。
4. 絞ったビームをカメラ中央へ合わせ、ビームを広げて絞りを中央へ合わせます。繰り返して調整します。
5. Parallel illuminationにします。
6. Recordの回折条件でDiff centerを最小化し、カメラ中央へ合わせます。
7. Viewへ戻り、大きなビーム位置ずれを直します。
8. Trial → Record → Viewを繰り返し、中心を収束させます。
9. Beam Blankにし、Upper Cameraを停止します。

元施設のCLAやDAC値は施設固有のため、ここでは指定しません。

### E. Screening

1. **Camera & Script > Setup**でPreviewとRecordのframe保存を確認します。
2. Base name、Navigator item label、連番を設定し、Atlas folder下の`screening`を保存先にします。
3. View montageを開き、結晶を選んで移動し、Navigator pointを追加してViewを撮影します。
4. 施設のscreening macro（元マニュアルでは`ScreeningShot`）を実行し、回折点を確認します。
5. 形や大きさの異なる結晶を複数確認します。

`ScreeningShot`と`ScreeningShotDamage`は本script fileに含まれます。`ScreeningShotDamage`の配布設定は、Preview条件で180秒露光、1秒/frameです。試料に適した条件か確認してから使用してください。

### F. 連続測定

1. **Camera & Script > Setup**でRecordのframe保存を確認します。
2. Base name、Navigator item label、連番を設定し、Atlas folder下の`data`を保存先にします。
3. 各View montageで**Add Points**を使って結晶を選び、すべて測定対象（`A`）にします。
4. 必要に応じてビーム・絞り調整を再実行します。
5. **Navigator > Acquire at Items**を開き、Final acquisitionを選びます。
6. **Run script**に`MicroEDContinuousRotation`を指定します。
7. 必要なoptionを設定します。元ワークフローではRough Eucentricityと一部Z moveのskipを使用しますが、施設ごとに適否を確認してください。
8. **Run script before action**に`BeforeTilt`を指定します。
9. **Run script after action**に`AfterTilt`を指定します。
10. **GO**で開始し、最初の測定を監視します。回折像、回折点の角度範囲、TEM CenterとSerialEMのdefocus、Obj/DACなどに異常がないことを確認します。

途中で終える場合、元マニュアルでは**StopではなくEnd Navigator**を使用します。再開前に最後に試行したitemとその`A`設定を確認してください。

## 注意点

- `MicroEDContinuousRotation`は`SetSlitIn 1`でenergy-filter slitを常に挿入します。
- `BeforeTilt`は装置・施設依存性が高く、外部macroなしではそのまま動作しません。
- `AfterTilt`は512 × 512 pixelのView JPEGを保存し、Trialへ移動してeucentric focus/defocus、stage tilt、Image Shiftを戻します。
- Atlas、alignment、calibration、`EMProperties`、`Funcs`は含みません。
- 元packageではdamage macro名が`ScreeningfShotDamage`でした。公開版では`ScreeningShotDamage`へ修正し、処理本体は維持しています。

## 謝辞

本READMEの測定ワークフローは、安達成彦氏作成の「230901-CRYOARM200_microED_マニュアル_v13」から対象部分を簡潔に整理したものです。同マニュアルには、牧野文信、柳澤春明、中根崇智、川本晃大、山田悠介の各氏への謝辞が記載されています。

## 参考文献とコードの由来

本workflowで以下を参照しました。
 - CRmov ver. 1.1, 2019-03-02, M. Jason de la Cruz, MSKCC.
 - Takaba, K et al., JSB (2020)

そのため、本workflowを利用する際は、以下の文献を引用してください。

Takaba, K., Maki-Yonekura, S., & Yonekura, K. (2020). Collecting large datasets of rotational electron diffraction with ParallEM and SerialEM. *Journal of Structural Biology, 211*(2), 107549. [https://doi.org/10.1016/j.jsb.2020.107549](https://doi.org/10.1016/j.jsb.2020.107549)
