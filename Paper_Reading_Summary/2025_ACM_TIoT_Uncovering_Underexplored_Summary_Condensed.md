# 輪講まとめ：Uncovering Underexplored Runtime Behaviors in ROS2-Based Autonomous Systems

**論文情報**
- **著者**: Chenghao Fan, Lanshun Nie, Jiacheng Zhang, Kun Dai, Shenghan Gao, Jing Li
- **所属**: Harbin Institute of Technology / New Jersey Institute of Technology
- **掲載**: ACM Transactions on Internet of Things, 2025年10月
- **DOI**: https://doi.org/10.1145/3772083

---

## 1. 研究の背景と動機

ROS2はDDSの上に構築されたミドルウェアであり、自律走行車やモバイルロボティクスなど安全性・性能が重要なドメインで広く使われている。非同期通信（Publisher/Subscriber、Service、Action）とExecutorによるコールバックベースのnon-preemptiveスケジューリングを提供する。

既存のROS2向けタイミング解析やスケジューリング戦略は、cause-effect chainを固定構造・FIFOデータ消費・全計算がExecutor管理下といった**単純化された仮定**に依存している。本論文は、Navigation2やAutowareの実測データをもとに、これらの仮定が実世界で成り立たないことを示す**4つのギャップ**を実証的に明らかにした。また、`ros2_tracing`を拡張してTFとActionのcause-effect chainトレーシングを実現した。

---

## 2. 4つの実証的観察

### 観察1：Partial cause-effect chainにもハードデッドラインがある

**既存の前提**: タイミング解析はセンサ→アクチュエータのend-to-end cause-effect chainに焦点を当てている。

**実際**: Navigation2では、`goal_pose`受信→Action Server `compute_path_to_pose`の`handle_accept`→Action Client `goal_response`という**end-to-endでないpartial cause-effect chain**に20msのハードデッドラインが存在する。このchainはセンサにもアクチュエータにも直接関与しないが、ナビゲーション開始の起点となる重要なchainである。

デッドラインを超過するとNavigation2のRecoveryメカニズムが発動する。Recoveryはコストマップクリア→リトライ→失敗時にspin（1.57rad回転）→wait（1s停止）→backup（0.15m後退）の順で実行され、連続失敗ごとにエスカレートする。通常動作中にRecoveryが発動すると、速度低下・振動的挙動・最悪の場合ミッション中断といった深刻な機能劣化が生じる。

**示唆**: リアルタイム解析の対象をend-to-end chainだけでなく、内部の意思決定に関わるpartial chainにも拡大する必要がある。Partial chainはセンサのタイマーではなくsporadicなイベントで起動されることが多く、リリースパターンのモデル化も課題となる。

---

### 観察2：Cause-effect chainの構造とタイミングが実行時に動的に変化する

**既存の前提**: Cause-effect chainは固定構造・ほぼ一定の実行時間を持つ。

**実際**:
- **構造の変動**: Navigation2ではビヘイビアツリーのtickにより、cause-effect chainの構造が動的に切り替わる。グローバルプランニング起動時は LiDAR → nav2_amcl → nav2_planner → nav2_bt_navigator → nav2_controller → wheels という長いchainになるが、巡航状態では LiDAR → nav2_amcl → nav2_controller → wheels とGlobal Plannerをバイパスする短いchainになる。この切り替えはデフォルトで約1秒ごとに発生する。Recovery時にはさらに異なる構造（costmapクリア、spin等）に遷移する。
- **タイミングの変動**: Dijkstraベースのグローバルプランニングは、目標から遠い時（約180ms）→近い時（約20ms）と大きく変動。WCETでモデル化すると大部分の時間帯で過度に悲観的になる。

**示唆**: 静的な最悪ケース解析ではなく、動的モデルと適応的スケジューリングが必要。古典的リアルタイムシステムのマルチモード解析の適用も考えられるが、モード遷移が秒単位で頻発するため、遷移中のリアルタイム保証が非自明な課題となる。

---

### 観察3：非FIFOデータアクセスが存在する

**既存の前提**: データバッファはFIFO順序で消費される。

**実際**: Navigation2のローカルプランニングでは、ロボットのposeを`odom`フレームから`map`フレームに変換する必要がある。`map`フレームはLiDAR（AMCL経由）で更新され、`odom`フレームはホイールオドメトリで更新されるが、AMCLの計算遅延により`map`フレームのデータは常に約1秒古い。時間的整合性を保つため、`follow_path`コールバックはTFバッファから**最新のLiDARデータとオドメトリのタイムスタンプに対応する古いLiDARデータの両方**を同時に読み出す。

この非FIFOアクセスにより、MRT（Maximum Reaction Time）とMDA（Maximum Data Age）が等しくならない。FIFOアクセスのみの場合はMRT=MDAが成立するが、古いデータが後続のcallbackで再利用されるとMDAがMRTを上回る。AutowareのPointPaintingノード（点群とRGB画像のfusion）でも同様にTFの過去タイムスタンプを参照する。

また、ROS2の有界キュー（デフォルトサイズ10）では、高頻度Publisherに処理が追いつかない場合、古いフレームから順に処理されるため、常に最新でないデータを使用してしまう。TurtleBot3+YOLOv3 Tinyの実験では、カメラ60Hz・バッファサイズ10で約90%のセンサデータが喪失。バッファサイズ1に設定すると常に最新データが使用され、反応時間が安定した。

**示唆**: タイミング解析を非FIFOアクセスに対応させる必要がある。コールバック内のデータ使用パターンを明示的に識別し、古いデータの「年齢」を制約する解析手法の開発が求められる。

---

### 観察4：Executor管理外の計算スレッドが存在する

**既存の前提**: 全ての計算はROS2 Executorの管理下にある。

**実際**: ROS2のExecutorはnon-preemptiveスケジューリングのため、長時間実行されるコールバックは他のコールバックをブロックしてしまう。これを回避するため、開発者は重い計算をOS APIで生成した別スレッドに委任する。

- Navigation2のAction Server: グローバルプランニング（最大180ms）を`std::thread`で別スレッドに委任。goal受付時にスレッド生成、計算完了後に破棄。
- Costmap2DROSの`mapUpdateLoop`: ノード初期化時に起動される常駐バックグラウンドスレッド。グローバルコストマップを1Hz、ローカルコストマップを5Hzで更新。
- AutowareのNDT Scan Matcher: `#pragma omp parallel for`で複数のOpenMPスレッドを生成し、点群マッチングを並列実行。

これらのスレッドはExecutorのReadySetやコールバック優先度の外にあるため、既存のスケジューリング解析ではモデル化されない。しかしCPUリソースを競合し、cause-effect chainの一部としてROS2メッセージや共有データ構造を介してExecutor管理下のコールバックとやり取りする。

**示唆**: 非Executor制御スレッドをリアルタイム解析にファーストクラスのエンティティとして組み込む必要がある。スレッドの挙動をROS2ランタイムに公開する標準的なアノテーションやインターフェースの開発も今後の課題である。

---

## 3. 実験結果のハイライト

### Navigation2シミュレーション（TurtleBot3 + Gazebo）

- **Config 1（AMCL+cv_detectionの同一Executor配置）**: Chain 1の59/100インスタンスで1.2sデッドライン超過。ミッション完了時間が39s→48.5sに悪化。非FIFOアクセスのため最新LiDARデータの遅延が直接影響。
- **Config 2（Action Serverとcamera_img_recorderの競合）**: チェックポイント到着時にExecutorの暗黙優先度（Subscriber > Action）によりhandle_acceptが遅延し、約5/100回で20msデッドライン超過→Recovery発動。
- **Config 3（Executor外スレッドとの競合）**: グローバルプランニングスレッドがweb_video_serverを阻害。ロボットが目標に近づくにつれ改善（実行時間の動的変動を反映）。

### Autowareケーススタディ（CARLA + Autoware.universe）

- Config 1では`pointcloud_container`との競合によりChain 1のMRTが1691msまで増加し、制御精度が劣化して車線逸脱が発生。
- Config 2で優先度を調整することで改善。Partial cause-effect chainのスケジューリングが自律走行の機能的正確性に直接影響することを実証。

---

## 4. ツール貢献

既存の`ros2_tracing`（C. Bédard et al.）はTimer、Publisher、Subscriberのcause-effect chainのみ対応していた。本論文ではTFとActionをサポートする拡張を行い、より現実的なcause-effect chainの再構築を可能にした。

**TFトレーシング**: ROS2ではTFバッファは各ノードがローカルに保持する分散設計のため、Publisher-Subscriberリンクの解析だけではノード内通信を捕捉できない。以下の3トレースポイントを導入：
- `store_in_buffer`: 変換メッセージのTFバッファへの格納を記録。格納を行ったcallbackインスタンスを上流partial chainの末尾として識別。
- `calculate_in_buffer`: 格納済みメッセージの使用を記録。使用したcallbackインスタンスを下流partial chainの先頭として識別。
- `interpolate_in_buffer`: 2つのTFメッセージを補間して変換を計算する場合に記録。

これらにより、`store_in_buffer`で記録されたメッセージを`calculate_in_buffer`や`interpolate_in_buffer`が参照した際に、上流と下流のcallbackを自動的にリンクし、TFバッファ経由のcause-effect chainセグメントを再構築する。

**Actionトレーシング**: Client-Service通信の双方向追跡（request/response両方のID・タイムスタンプ記録）を追加。さらに、Executor外で実行される長時間計算スレッド用に`action_callback_start` / `action_callback_end`トレースポイントをアプリケーションコードに挿入し、Action関連のcause-effect chain全体（Client→Server→計算スレッド→結果返却）の時間的フットプリントを取得可能にした。

**オーバーヘッド**: 追加トレースポイントは最大48バイトのデータを書き込み、平均オーバーヘッドは0.0012ms（既存トレースポイントの0.003msより小さい）。
