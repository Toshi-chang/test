# 音圧・粒子速度・回転トラッキング・機械学習システム
## 要件定義 前段資料 / マスターコンテキスト

---

# 0. 本資料の目的

本資料は、音圧 `p`、粒子速度 `u`、回転数 `RPM` を入力として、

1. データ読込・同期・前処理
2. 回転トラッキング / Order解析
3. 音圧・粒子速度・P-U関係の解析
4. 外乱混入区間・データセルの判定
5. 外乱セルを除外した特徴量生成
6. 回転同期成分を利用した音源・現象解析
7. 機械学習による正常 / 異常判定
8. 結果の可視化・比較・保存

を行うPoCシステムについて、**既存コード・既存システムを参照しながら詳細要件定義を進めるための前段資料**である。

本資料自体は完成版の要件定義書ではない。

別チャットまたは別AIへ本資料と既存コード一式を渡し、

- 現行コード解析
- 現行機能棚卸し
- データフロー把握
- 要件とのGap分析
- 詳細要件定義
- アーキテクチャ再設計
- PoC実装計画

へ展開することを目的とする。

---

# 1. 開発対象の基本方針

## 1.1 今回の対象

今回の対象は **PoCシステム** である。

将来的な本番システム・量産設備・自動検査システムを見据えて目標設定は行うが、現段階では本番システムの詳細要件を定義しない。

理由：

- 本PoCの技術成立性が確認できなければ、その先の本番システム設計へ進めない。
- 現時点ではアルゴリズム・特徴量・外乱判定・異常判定方式が研究開発段階である。
- 将来要件を先に固定するとPoC側の自由度を不必要に制限する可能性がある。

したがって本要件定義では、

> **将来的に拡張可能な構造を意識しつつ、PoCとして必要な機能・解析能力・検証能力を優先する。**

---

# 2. システムの目的

## 2.1 主目的

本システムの主目的は以下の2点である。

### C. 正常 / 異常を機械学習によって判定する

入力された音圧・粒子速度・回転情報から特徴量を生成し、機械学習を用いて正常 / 異常を判定する。

### D. 外乱が混入したデータを異常判定対象から除外する

ここでいう「外乱除去」は、信号処理によって外乱波形そのものを数学的に消去することを必須としない。

本PoCでの基本思想は、

> **外乱が入っていると判断された時間×周波数等のデータセルを特定し、そのセルを機械学習による異常判定対象から除外する。**

である。

したがって、

- 外乱波形の完全復元
- Blind Source Separationによる時間波形分離
- 外乱信号の逆推定

を必須要件とはしない。

---

# 3. 副目的

主目的C/Dに加え、以下をPoC機能として実装対象とする。

## 3.1 解析ツール機能

解析者が以下を確認できること。

- 時間波形
- 回転数
- FFT
- Order
- 回転数-周波数
- 回転数-Order
- P-U関係
- 特徴量
- 外乱判定
- 異常判定
- 条件間比較

単なる自動判定ブラックボックスではなく、

> **人間が現象を確認・理解・検証できる解析環境**

を備えること。

---

## 3.2 音源 / 現象解析

本PoCでは完全な音源分離は要求しない。

対象：

### Level 1
- 対象機械由来音
- 外乱

の識別。

加えて、回転トラッキングによって識別可能な成分について、

- シャフト由来
- 回転同期成分
- ギヤ由来
- 高調波
- サイドバンド
- その他特定Order成分

などを可能な範囲で分離・分類・可視化する。

---

# 4. 想定処理フロー

現時点での基本処理思想を以下とする。

```text
Raw Measurement
    │
    ├─ Pressure p
    ├─ Particle Velocity u
    └─ RPM
    │
    ▼
Data Validation
    │
    ▼
Pre-processing
    │
    ├─ calibration
    ├─ filtering
    ├─ segmentation
    └─ synchronization check
    │
    ▼
RPM / Order Tracking
    │
    ▼
Time / Frequency / Order / P-U Analysis
    │
    ▼
Feature Extraction
    │
    ├─ Time features
    ├─ Frequency features
    ├─ Order features
    ├─ P-U features
    └─ Mechanical-source features
    │
    ▼
Disturbance Detection
    │
    ▼
Disturbance Cell Mask
    │
    ├─ disturbance cell → exclude
    └─ valid machine cell → keep
    │
    ▼
Source / Phenomenon Analysis
    │
    ▼
ML Anomaly Detection
    │
    ▼
Result Integration
    │
    ├─ Normal / Abnormal
    ├─ anomaly score
    ├─ anomalous order
    ├─ anomalous frequency
    ├─ suspected component/source
    └─ disturbance information
    │
    ▼
Visualization / Comparison / Export
```

---

# 5. 入力データ基本仕様

## 5.1 チャンネル構成

最大想定：

- P-U セット：12組
- 音圧 `p`：12 ch
- 粒子速度 `u`：12 ch
- 回転数：1 ch相当
- 合計：音響24 ch + 回転情報

P-Uは対になるセンサセットとして扱う。

---

## 5.2 サンプリング

想定サンプリング周波数：

```text
48 kHz
```

---

## 5.3 測定時間

1測定あたり：

```text
約2分
```

を基本想定とする。

将来的には測定時間可変に対応できる構造を推奨する。

---

## 5.4 DAQ

音圧・粒子速度は同一DAQで取得する。

P-U解析では位相関係が重要であるため、

- チャンネル間同期
- 位相遅れ
- ADC仕様
- センサ / アンプ遅延
- 校正

を要件定義対象とする。

---

## 5.5 回転数信号

現時点でシステムへ入力される回転情報は、

> **RPMに変換済みの数値**

を基本とする。

生タコパルス・エンコーダパルスを直接扱うことは現時点の必須要件ではない。

ただし、RPM値の、

- サンプリング周期
- タイムスタンプ
- 音響データとの同期方法
- 補間方法
- 遅延
- 更新周期
- 欠損処理

は必ず定義すること。

---

# 6. 回転状態

以下を対象とする。

- 加速
- 減速
- Ramp-up
- Coast-down

固定回転数だけを対象としたシステムではない。

したがってOrder解析では、

> 単純にFFTピーク周波数 ÷ 代表回転数

だけで済ませる設計は原則として不十分である。

RPM変動中の信号について、どのようにOrderを算出・追跡・平均化するかを詳細設計する必要がある。

---

# 7. 既存コード・既存システムの扱い

既存コードは存在するが、基本的には**参考実装**として扱う。

## 7.1 維持対象

以下は可能な限り参照する。

- 既に成立している機能
- 実装済みアルゴリズム
- 有効なUI
- データ読込処理
- 解析処理
- 可視化処理
- ファイル構造
- 過去検証結果
- 既に得られている技術的知見

ただし、

> 既存コードの構造を維持すること自体は目的としない。

---

## 7.2 再構築方針

既存システムは機能追加を積み重ねて構築されている。

そのため、

- 強いモジュール結合
- UIと解析ロジックの混在
- 解析処理の重複
- データ構造の不統一
- 状態管理の複雑化
- 特徴量追加時の影響範囲拡大
- ML処理と信号処理の密結合

などが存在する可能性を前提とする。

必要であれば、

- 大規模リファクタリング
- モジュール再分割
- 新規アーキテクチャへの移行
- 一部機能のみ移植
- 既存コードを仕様確認用に限定

を許容する。

---

# 8. 既存コード解析時のAIへの指示

本資料と既存コード一式が与えられた場合、いきなり改修を始めてはならない。

まず以下を実施すること。

## Phase 0-1：コード全体把握

確認する項目：

- ディレクトリ構成
- エントリーポイント
- UI構成
- データ入力処理
- データ保持構造
- 信号処理モジュール
- FFT処理
- Order処理
- RPM処理
- P-U解析
- 特徴量計算
- ML処理
- 結果保存
- 設定管理
- 外部依存ライブラリ

---

## Phase 0-2：データフロー作成

最低限、

```text
Input
↓
Loading
↓
Pre-processing
↓
Analysis
↓
Feature
↓
ML
↓
Visualization
↓
Export
```

の実コード上の流れを特定する。

関数名、クラス名、主要データ型も記録する。

---

## Phase 0-3：実装済み機能一覧

各機能について、

```text
Implemented
Partially Implemented
Prototype
Unused
Broken / Unclear
Not Implemented
```

に分類する。

---

## Phase 0-4：Gap Analysis

本資料の要求と既存コードを比較し、

| 機能 | 既存状態 | 要求 | Gap | 対応案 |
|---|---|---|---|---|
| RPM tracking |  |  |  |  |
| Order analysis |  |  |  |  |
| P-U analysis |  |  |  |  |
| disturbance mask |  |  |  |  |
| feature extraction |  |  |  |  |
| ML |  |  |  |  |
| visualization |  |  |  |  |

を作成する。

---

# 9. 要件定義カテゴリ

以下12カテゴリを詳細要件定義の基本構造とする。

1. システム目的・最終アウトプット
2. 入力データ仕様
3. 時刻同期・データ同期
4. 回転トラッキング / Order解析
5. 信号処理
6. P-U解析
7. 特徴量
8. 外乱 / 音源・現象識別
9. 正常 / 異常判定
10. 学習・評価
11. 可視化 / UI
12. データ・システム設計

以降、それぞれについて要件化すべき論点を列挙する。

---

# 10. 要件1：システム目的・最終アウトプット

## 10.1 主判定

最低限、

```text
Normal
Abnormal
```

を出力可能とする。

ただしPoCでは単一のOK/NGだけでは不十分である。

---

## 10.2 付随情報

可能であれば以下を出力する。

- 異常度スコア
- 異常周波数
- 異常Order
- 寄与したP-Uセット
- 推定音源
- 推定機械要素
- 外乱検出有無
- 外乱除外率
- 判定対象として残った有効データ量
- 主要特徴量
- 正常データとの距離
- 可視化上の異常箇所

---

## 10.3 判定階層

理想形：

```text
Measurement
 ├─ Source / Component A
 │    ├─ Order range
 │    ├─ feature score
 │    └─ anomaly score
 │
 ├─ Source / Component B
 │
 ├─ Disturbance
 │    └─ excluded cells
 │
 └─ Overall judgment
```

---

# 11. 要件2：入力データ仕様

詳細決定事項：

- ファイル形式
- データ列構成
- 時刻列
- P-Uペア定義
- チャンネル名称
- センサID
- 単位
- 校正値
- 感度
- 測定位置
- センサ方向
- RPM列
- メタデータ
- 製品ID
- 測定ID
- 測定条件

---

## 11.1 単位

原則として内部解析では物理量へ変換する。

例：

```text
Pressure: Pa
Particle velocity: m/s
RPM: rpm
Frequency: Hz
Order: -
```

電圧データを入力する場合は感度・校正情報を必須とする。

---

# 12. 要件3：時刻同期・データ同期

P-U解析では重要度が高い。

確認事項：

- p/uのサンプル同期
- RPM時間軸
- RPM更新周期
- 音響サンプルとの対応
- RPM補間方法
- 欠損値
- 時刻ずれ
- DAQ遅延
- センサ位相特性
- アンプ遅延
- クロック精度

---

## 12.1 RPM補間

48 kHz音響データに対してRPM値が低レートで存在する可能性がある。

候補：

- zero-order hold
- linear interpolation
- spline
- smoothing + interpolation

どれを利用するかは実データ特性を確認して決定する。

---

# 13. 要件4：回転トラッキング / Order解析

本システムの中核機能の一つ。

基本式：

```text
rotation frequency = RPM / 60
Order = frequency / rotation frequency
```

ただしRamp-up / Coast-downを扱うため、時間変動する回転数を考慮する。

---

## 13.1 要検討アルゴリズム

以下を比較する。

- 短時間FFT + 窓内代表RPM
- 時間周波数解析後のOrder変換
- RPMベースリサンプリング
- 角度領域リサンプリング相当処理
- computed order tracking
- tachlessに近い処理が必要か
- ridge tracking
- order map生成

RPM値しか取得できないため、

> 生タコパルスが存在する場合と同等の高精度角度同期が可能とは限らない

点を明示すること。

---

## 13.2 決定すべきパラメータ

- Order分解能
- 最大Order
- 最小Order
- 周波数範囲
- FFT length
- window
- overlap
- averaging
- RPM bin
- Order bin
- interpolation
- smoothing
- 加減速速度への追従性
- Orderピーク抽出方法

---

# 14. 要件5：信号処理

候補：

- DC removal
- detrend
- HPF
- LPF
- band-pass
- anti-alias
- resampling
- windowing
- FFT
- STFT
- PSD
- CPSD
- averaging
- smoothing
- transient処理
- clipping検出
- saturation検出

---

## 14.1 原則

前処理は必ず設定値を保存し、

```text
Raw data
→ Processing parameters
→ Processed result
```

を再現可能にする。

---

# 15. 要件6：P-U解析

音圧と粒子速度を別々に解析するだけではなく、両者の関係を重要特徴として扱う。

候補：

- Pressure spectrum
- Particle velocity spectrum
- Auto spectrum
- Cross spectrum
- Coherence / MSC
- Phase difference
- Active acoustic intensity
- Reactive acoustic intensity
- p/u ratio
- Order-domain P-U relation

---

## 15.1 MSC

例：

```text
MSC(f) = |S_pu(f)|² / (S_pp(f) S_uu(f))
```

用途候補：

- pとuの関連度
- 機械由来音と外乱の識別
- 信頼性の低いセル除外
- 特徴量
- 異常検知補助

---

# 16. 要件7：特徴量

特徴量は固定構造ではなく**追加可能なPlugin型設計**を基本方針とする。

概念：

```text
FeatureExtractor
 ├─ TimeFeatureExtractor
 ├─ FrequencyFeatureExtractor
 ├─ PUFeatureExtractor
 ├─ OrderFeatureExtractor
 ├─ MechanicalFeatureExtractor
 └─ ExperimentalFeatureExtractor
```

---

## 16.1 時間領域

候補：

- RMS
- peak
- peak-to-peak
- crest factor
- kurtosis
- skewness
- variance
- impulsiveness
- envelope-related metrics

---

## 16.2 周波数領域

候補：

- band energy
- band RMS
- spectral peak
- peak frequency
- spectral centroid
- spectral entropy
- spectral slope
- harmonic energy
- narrow band energy

---

## 16.3 P-U

候補：

- MSC
- mean MSC
- max MSC
- MSC variance
- P-U phase
- cross-spectrum magnitude
- active intensity
- reactive intensity

---

## 16.4 Order

候補：

- Order amplitude
- Order band energy
- peak Order
- harmonic Order
- sideband energy
- Order width
- Order stability
- RPM dependency
- Order-domain MSC
- Order-domain intensity

---

## 16.5 機械構造由来

機械構造情報を利用可能とする。

入力候補：

```text
shaft
gear
gear teeth
gear ratio
motor pole
bearing
known order
known resonance
```

これにより、

- shaft order
- gear mesh order
- harmonic
- sideband
- component-specific band

を自動設定できる構造を検討する。

---

# 17. 要件8：外乱 / 音源・現象識別

## 17.1 外乱除去の定義

本PoCでの「外乱除去」は、

> 外乱が含まれると判断されたデータセルを異常判定対象から除外すること

である。

元波形から外乱を完全除去することではない。

---

## 17.2 データセル定義

今後必ず明確化する。

候補：

```text
time × frequency
time × order
rpm × frequency
rpm × order
channel × frequency
P-U pair × frequency
```

MLへ投入する際の最小評価単位を決定する必要がある。

---

## 17.3 外乱判定材料候補

- P-U coherence
- P-U phase
- intensity
- Order同期性
- RPMとの相関
- 複数P-Uセンサ間比較
- 時間局所性
- 空間局所性
- 周波数局所性
- 学習済み外乱モデル

---

# 18. 要件9：正常 / 異常判定

## 18.1 基本思想

現時点では周波数軸での評価を主体とする。

対象時間区間については、必要に応じて平均化・統計化する。

理想的には、

```text
Component
×
Order / Frequency region
×
Feature
```

単位で異常判定し、最後に総合判定する。

---

## 18.2 ML方式

以下を両方試す。

### 正常学習系

- Isolation Forest
- LOF
- One-Class SVM
- PCA reconstruction error
- Autoencoder
- VAE
- その他One-Class / unsupervised

### 正常 + 異常利用系

- supervised classifier
- semi-supervised
- weak supervision
- metric learning
- anomaly score calibration

方式はPoC比較によって決定する。

---

# 19. 要件10：学習・評価

必ず検討する事項：

- train
- validation
- test
- measurement split
- product split
- day split
- machine split
- operating-condition split

同一測定データから切り出したセルをtrain/test双方へ混在させることによるリークに注意する。

---

## 19.1 評価指標

候補：

- Precision
- Recall
- F1
- False Positive Rate
- False Negative Rate
- ROC-AUC
- PR-AUC
- anomaly score separation
- detection rate per measurement
- detection rate per component
- disturbance rejection performance

---

## 19.2 外乱除去性能

異常判定性能とは別に評価する。

例：

```text
Disturbance detection recall
Disturbance false rejection rate
Valid machine-cell retention rate
```

重要なのは、

> 外乱を除外できること

だけではなく、

> 正常な機械由来情報まで除外していないこと

である。

---

# 20. 要件11：可視化 / UI

PoCでは解析者向けUIを重視する。

---

## 20.1 基本画面候補

### Time View

- p(t)
- u(t)
- RPM(t)
-解析対象区間
- 外乱区間

### Frequency View

- Pressure spectrum
- Particle velocity spectrum
- P-U coherence
- phase
- intensity

### Order View

- Order spectrum
- RPM-Order map
- RPM-Frequency map

### Feature View

- feature table
- feature trend
- anomaly score
- feature contribution

### ML View

- disturbance mask
- anomaly cells
- final judgment
- model score

---

## 20.2 連動表示

理想例：

Orderをクリック
↓
該当Orderに関連する

- p
- u
- MSC
- phase
- intensity
- RPM
- anomaly score
- source / component

を同期表示する。

---

# 21. 要件12：データ・システム設計

## 21.1 データ階層

推奨概念：

```text
Measurement
 ├─ Raw
 │    ├─ pressure
 │    ├─ particle_velocity
 │    └─ rpm
 │
 ├─ Metadata
 │    ├─ sensor
 │    ├─ machine
 │    ├─ operating_condition
 │    └─ calibration
 │
 ├─ Processed
 │
 ├─ Spectra
 │
 ├─ Orders
 │
 ├─ PU
 │
 ├─ Features
 │
 ├─ DisturbanceMask
 │
 └─ MLResult
```

---

## 21.2 再現性

最低限以下のversionを保持する。

```text
processing_version
feature_version
model_version
configuration_version
calibration_version
```

---

# 22. 推奨アーキテクチャ

既存コードを解析した後、以下のような分離を検討する。

```text
Application / UI
        │
        ▼
Analysis Controller
        │
 ┌──────┼────────┐
 ▼      ▼        ▼
Data   Signal    RPM/Order
I/O    Process   Tracking
        │
        ▼
      P-U
        │
        ▼
     Features
        │
 ┌──────┴──────────┐
 ▼                 ▼
Disturbance       Source /
Detection         Phenomenon
 └──────┬──────────┘
        ▼
        ML
        │
        ▼
 Result / Visualization
```

---

# 23. 重要な設計原則

## 23.1 UIと解析ロジックを分離

GUIがなくても解析処理単体を実行できること。

---

## 23.2 生データを保持

解析手法は今後変更されるため、生波形を残す。

---

## 23.3 特徴量は追加可能

研究開発段階で特徴量を固定しすぎない。

---

## 23.4 MLモデル交換可能

特定モデルに依存したシステム構造を避ける。

---

## 23.5 外乱判定と異常判定を分離

概念上、

```text
DisturbanceDetector
AnomalyDetector
```

を別処理として扱う。

---

## 23.6 回転同期情報を最大限利用

既知の機械情報はMLへ推測させず、解析ロジック側で利用する。

---

# 24. 現時点で確定している事項

| 項目 | 状態 |
|---|---|
| 対象 | PoC |
| 主目的 | 異常判定 |
| 主目的 | 外乱混入セル除外 |
| 副目的 | 解析ツール |
| 副目的 | 回転同期由来の音源/現象解析 |
| P-U | 最大12セット |
| 音響ch | 最大24ch |
| Fs | 48 kHz |
| 測定時間 | 約2分 |
| DAQ | 同一DAQ |
| RPM入力 | RPM変換済み値 |
| 回転状態 | Ramp-up / Coast-down含む |
| 音源分離 | 外乱 vs 対象音 + Orderで識別可能な成分 |
| ML | 正常学習、異常利用の双方を試験 |
| 機械構造情報 | 利用する |
| 特徴量 | Plugin型 / 拡張可能 |
| 既存コード | 参考実装 |
| 再設計 | 許容 |
| コード提供 | リポジトリ / アーカイブ単位 |

---

# 25. 要件定義開始後に優先して決める事項

以下は依存関係が強いため、優先順位を高くする。

## Priority 1：RPM時間軸仕様

- RPMデータ更新周期
- timestamp
- 音響データとの同期
- RPM補間
- RPM精度
- ノイズ
- 欠損

---

## Priority 2：Order Tracking方式

RPM値のみでRamp-up / Coast-downを扱うため、

- 時間窓方式
- 角度相当リサンプリング
- Order map
- 分解能

を決定する。

---

## Priority 3：MLへ渡す「セル」の定義

外乱除外単位および異常判定単位を定義する。

最重要候補：

```text
time × frequency
time × order
rpm × order
frequency band
order band
```

---

## Priority 4：外乱の定義

実際に想定する外乱を分類する。

例：

- 周囲設備
- 人
- 衝撃
- 他機械回転音
- 空調
- 突発音
- 広帯域ノイズ
- 回転同期外乱
- 製品近傍音だが対象外

---

## Priority 5：正常の定義

正常データの変動要因：

- 製品個体差
- 回転数
- 温度
- 負荷
- センサ位置
- 日差
- 設備差
- ロット差

を定義する。

---

# 26. 次チャットでの進め方

AIは一度に全要件を質問してはならない。

以下の順番で進める。

```text
STEP 1
既存コード解析

STEP 2
現行仕様の再構築

STEP 3
Gap Analysis

STEP 4
最上位要件確認

STEP 5
入力・同期仕様

STEP 6
RPM / Order Tracking

STEP 7
P-U解析

STEP 8
外乱判定

STEP 9
特徴量

STEP 10
ML

STEP 11
UI

STEP 12
データ / システム構造

STEP 13
要件定義書化
```

---

# 27. AIへの対話ルール

以下を守ること。

1. 一度に大量質問を投げない。
2. 前の回答によって次の質問を変える。
3. 未確定事項を勝手に決めない。
4. 仮定した場合は仮定と明記する。
5. 既存コードに存在するからという理由だけで仕様採用しない。
6. 物理的妥当性と実装都合を分離して議論する。
7. PoC段階で過剰な本番要件を要求しない。
8. 将来拡張性を考慮する。
9. Signal Processing / Acoustic Physics / ML / Software Architectureを分離して整理する。
10. 変更による他機能への影響を明示する。

---

# 28. 要件決定記録

要件を決めるたびに以下を記録する。

```text
Requirement ID:
Category:
Decision:
Reason:
Alternatives:
Impact:
Open Issues:
Date:
```

---

# 29. 未決事項管理

各論点を以下で管理する。

```text
DECIDED
PROVISIONAL
OPEN
BLOCKED
REJECTED
```

---

# 30. 最終成果物

要件定義完了時には最低限以下を作成する。

## 30.1 System Requirement Specification

- 背景
- 目的
- スコープ
- 入力
- 処理
- 出力
- UI
- ML
- データ
- 制約
- 非機能

---

## 30.2 Software Architecture

- module diagram
- data flow
- class / package responsibility
- interfaces
- data structures

---

## 30.3 Signal Processing Specification

- preprocessing
- FFT
- P-U
- RPM
- Order Tracking
- averaging

---

## 30.4 Feature Specification

各特徴量について、

```text
name
definition
formula
input
output
unit
parameters
physical meaning
use in ML
```

を定義する。

---

## 30.5 ML Specification

- training data
- validation
- input features
- preprocessing
- model
- score
- threshold
- evaluation
- versioning

---

## 30.6 Gap Analysis

既存システムから新PoCへの移行方針を定義する。

---

# 31. 現段階での重要な技術課題

以下は要件定義中に必ず深掘りする。

### A. RPM値だけでの高精度Order Tracking

生タコパルスがない条件で、

- Ramp-up
- Coast-down
- 高Order
- 急加減速

をどこまで精度良く追跡できるか。

---

### B. 外乱セル判定

何を根拠として

```text
machine signal
disturbance
unknown
```

へ分類するか。

---

### C. P-U情報の有効性

- coherence
- phase
- intensity

が外乱判定および異常判定へどの程度寄与するか。

---

### D. 時間平均と異常情報保持のトレードオフ

周波数軸評価のため時間平均を行う一方、

- 一時的異常
- 回転変化依存
- 突発異常

を失わない設計が必要。

---

### E. 多チャンネル利用

P-U × 12セットを、

- 独立評価
- 統合
- センサ間特徴
- 空間特徴

のどこまで利用するか。

---

### F. 特徴量爆発

P-U × Order × Frequency × RPM × Feature

の組合せによって特徴量数が極端に増える可能性がある。

Feature selection / dimensionality reductionを検討する。

---

# 32. PoC成功条件

最終的な数値閾値は要件定義で決める。

概念上、PoC成功には以下が必要。

1. Ramp-up / Coast-downに対してOrderを追跡できる。
2. P-U情報を解析できる。
3. 外乱候補セルを可視化できる。
4. 外乱セルを異常判定対象から除外できる。
5. 機械由来のOrder成分を抽出できる。
6. 特徴量を再現可能に生成できる。
7. MLによる異常度を算出できる。
8. 正常 / 異常データで性能比較できる。
9. 結果を人間が解析できるUIを持つ。
10. 新しい特徴量・MLモデルを追加可能である。

---

# 33. この資料を受け取ったAIが最初に行うこと

既存コードまたはリポジトリが同時に渡された場合：

> **コード変更を開始せず、まず現状解析を行うこと。**

最初のアウトプットとして以下を提示すること。

### 1. 現行システム概要

### 2. 実装済機能

### 3. データフロー

### 4. モジュール依存関係

### 5. 技術的問題点

### 6. 本資料とのGap

### 7. 要件定義を進めるために最初に確認すべき質問

質問は優先順位順に少数ずつ提示する。

---

# 34. 注意事項

本PoCでは、

> 「MLを実装すること」

自体を目的にしない。

重要なのは、

```text
物理現象
↓
信号処理
↓
回転同期解析
↓
P-U解析
↓
外乱識別
↓
特徴量
↓
ML
```

という因果関係が説明可能なこと。

特に、外乱が存在する非暗騒音環境では、

> 単に異常スコアが高い

という結果だけでは、製品異常なのか外乱なのか区別できない。

したがって、

**外乱判定 → 有効データ抽出 → 異常判定**

をシステム上明確に分離する。

---

# 35. 最終的な設計思想

本PoCは、

> 「音をAIに入れてOK/NGを出すシステム」

ではなく、

> **音圧・粒子速度・回転同期情報を用いて、物理的意味を保持した状態で外乱を切り分け、対象機械の回転同期現象を解析し、その結果をMLへ渡して異常を判定する解析・検証プラットフォーム**

として設計する。

この思想を、今後の要件定義・アーキテクチャ設計・実装判断の基準とする。
