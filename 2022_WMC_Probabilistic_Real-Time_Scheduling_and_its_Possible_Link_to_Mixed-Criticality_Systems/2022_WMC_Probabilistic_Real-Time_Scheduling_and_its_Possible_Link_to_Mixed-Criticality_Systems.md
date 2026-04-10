# 確率的リアルタイムスケジューリングとMixed-Criticalityシステムへの可能な関連

**著者:** Georg von der Brüggen, Sergey Bozhko, Mario Günzel, Kuan-Hsun Chen, Jian-Jia Chen, Björn B. Brandenburg
**所属:** TU Dortmund University / Max Planck Institute for Software Systems (MPI-SWS) / University of Twente

---

## 1. 背景と動機

古典的な決定論的スケジューリング分析では、すべてのタスクインスタンスが最悪実行時間（WCET）で実行されると仮定する。この悲観的な仮定はプロセッサの平均的利用率を大幅に低下させる。

**Mixed-Criticalityシステム**は、Vestal（2007）により提案され、タスクが複数の実行モードを持つモデルである。

このMixed-Criticalityモデルは、その最も基本的な形式において、かなりの批判を受けてきた [10], [9], [14]。これは、低Criticalityモードでは全タスクにタイミング保証を提供し、高Criticalityタスクが実行時間予算を超過すると高Criticalityモードに切り替わるが、そへの復帰が考慮されていなかったためである。

その結果、Mixed-Criticality研究では、単一のモード切替を仮定する代わりに、タスクが頻繁に実行挙動を切り替えるシステムをより頻繁に考慮するようになった。このようなシナリオでは、限られた時間区間において少数のタスクのみがより大きな実行時間を示す状況を考慮することが自然である。したがって、システムモード切替の実行は高コストかつ不要である可能性がある。

Akesson et al.（2020）[1] の実証調査によれば、回答者の62%がソフトまたはファームリアルタイムタスクを含むシステムを扱っており、45%のシステムでは最も重要な機能でさえ時折のデッドラインミスを許容できるとしている。

デッドラインミスのリスクは、確率的スケジューラビリティ分析において定量化できる。通常、デッドラインミス率（長期的なデッドラインミスの割合）または最悪デッドライン失敗確率（WCDFP）（すなわち、ビジーウィンドウにおける最初のデッドラインミスの確率の上界）のいずれかを考慮する。

このため、**デッドラインミスの確率を定量化する確率的スケジューリング分析**が代替アプローチとして注目されている。

> 注: 本投稿は、第15回Models and Algorithms for Planning and Scheduling (MAPSP) 2022ワークショップで発表されたものの拡張版であり、元の版は確率的スケジューリングのみに焦点を当て、Mixed-Criticalityシステムは考慮していなかった。


---

## 2. 確率的分析の基礎と課題

### タスクモデル

タスクの実行時間は、複数のモードの集合として記述される。各モードは最大実行時間とその確率のペアで定義される。

$C_i / P_i = { (3, 0.9), (5, 0.1) }$

は、$τ_i$ が確率0.9で最大3の実行時間を持ち、確率0.1で実行時間5を持つことを意味する。

### ジョブレベル畳み込み（Job-Level Convolution）

与えられたリリースパターンに対し、ジョブを順に畳み込むことでデッドラインミスの確率を計算する。ただし、状態数がジョブ数に対して指数的に増大するため、直接適用には限界がある。

### 主要な研究課題

1. **検討すべきリリースパターン数をいかに削減するか**
2. **あるシナリオのデッドラインミス確率をいかに効率的に求めるか**

特にMixed-Criticalityシステムの文脈では、モード切替によりジョブの実行時間が独立でない場合にも、これらの計算が適用可能でなければならない。

---

## 3. デッドラインミス確率の効率的近似手法

| 手法 | 概要 | 特徴 |
|------|------|------|
| **リサンプリング** (Maxim & Cucu-Grosjean, 2013[12]; Markovic et al., 2021[11]) | 状態数が閾値を超えた時点で状態を結合し削減 | 精度損失の限界を定めることは未解決 |
| **モンテカルロ応答時間分析** (Bozhko et al., 2021[2]) | ジョブトレースを個別にサンプリングし、デッドラインミス数をカウント | 大規模シナリオに対応可能、並列化容易、信頼区間が小さい場合はサンプル数が膨大になる |
| **タスクレベル畳み込み** (von der Brüggen et al., 2018[15]) | 各区間を個別に評価し、タスクごとの寄与をモード数のみで計算 | 区間ごとの計算を高速化 |
| **解析的上界（Chernoff Bounds）** (Chen et al., 2019, 2017[6],[4]) | 区間内のワークロードが区間長を超える確率を直接推定 | 実行時間と精度のトレードオフに優れるが精度保証なし |

---

## 4. 最悪ケースリリースパターンの決定

- **固定優先度スケジューリング:** Maxim & Cucu-Grosjean（2013 [12]）が古典的クリティカルインスタントと同一のシナリオを提案したが、Chen et al.（2022）が反例を示した。2つの過近似が提供されているが、常に最悪ワークロードとなる単一のリリースパターンが存在するかは**未解決**。
- **EDF（最早デッドラインファースト）:** von der Brüggen et al.（2021 [16]）が最悪デッドライン失敗確率の上界となるシナリオを示した。
- **デッドラインミス率の解析的上界:** 固定優先度・EDFいずれにおいても既知の手法は存在しない。

---

## 5. ジョブ間の依存関係

従来の手法はジョブの実行時間が確率的に独立であると仮定している。Mixed-Criticalityシステムのように全タスクまたは一部が同時にモード切替する場合、直接的な適用は困難である。

von der Brüggen et al.（2021 [16]）は、EDF下で依存関係を非循環タスクチェーンとしてモデル化し、後続ジョブのモードが先行ジョブに依存する制限的シナリオに対する過近似を提供した。

---

## 6. Mixed-Criticalityシステムとの可能な接点

- **異なる保証レベルの解釈:** 高Criticality・低Criticalityタスクの異なる保証レベルを、デッドラインミスの許容残留リスクの異なる閾値として解釈可能。
- **モンテカルロ手法の拡張:** モード変更を含むスケジュールもサンプリングすることで、Criticality度ごとの信頼レベルでデッドライン失敗確率の上界を推定できる可能性がある。
- **モード遷移時の確率的利点:** 低Criticalityタスクが最大リソース要求を示す瞬間と、高Criticalityタスクがモード変更を引き起こす瞬間が同時に発生する確率は低いため、確率的分析により悲観性を軽減できる。
- **信頼度の不確実性:** 低Criticalityタスクのモード確率推定が楽観的であった場合に高Criticalityタスクに何が起こるかという「what-if分析」が考えられるが、実行時間分布が実観測により否定される時点を特定することは困難。

---

## 7. 結論

確率的タイミング分析は、高Criticalityタスクのデッドラインミス確率が十分に小さい場合にモード切替の延期・省略の根拠を与えうる有望なアプローチである。しかし、確率的スケジューリング自体に以下の未解決課題が残されている:

- 最悪到着パターンの確立
- デッドラインミス率の上界
- 確率的に依存するジョブの扱い

Mixed-Criticalityシステムへの拡張にはさらなる研究が必要であり、両分野の連携が重要である。

---

## 参考文献（主要）

- [1] B. Akesson, M. Nasri, G. Nelissen, S. Altmeyer, and R. I. Davis. An empirical survey-based study into industry practice in real-time systems. In 41st IEEE Real-Time Systems Symposium, RTSS, 2020.
- [2] S. Bozhko, G. von der Brüggen, and B. B. Brandenburg. Monte Carlo response-time analysis. In 42nd IEEE Real-Time Systems Symposium, RTSS, 2021.
- [9] R. Ernst and M. D. Natale. Mixed criticality systems - A history of misconceptions? IEEE Design & Test, 33(5):65–74, 2016.
- [10] A. Esper, G. Nelissen, V. Nélis, and E. Tovar. How realistic is the mixed-criticality real-time system model? In RTNS, 2015.
- [11] F. Markovic, A. V. Papadopoulos, and T. Nolte. On the convolution efficiency for probabilistic analysis of real-time systems. In Euromicro Conference on Real-Time Systems, ECRTS, 2021.
- [12] D. Maxim and L. Cucu-Grosjean. Response time analysis for fixed-priority tasks with multiple probabilistic parameters. In Real-Time Systems Symposium, 2013.
- [14] G. von der Brüggen, K.-H. Chen, W.-H. Huang, and J.-J. Chen. Systems with dynamic real-time guarantees in uncertain and faulty execution environments. In 37th IEEE Real-Time Systems Symposium, RTSS, 2016.
- [15] G. von der Brüggen, N. Piatkowski, K.-H. Chen, J.-J. Chen, and K. Morik. Efficiently approximating the probability of deadline misses in real-time systems. In Euromicro Conference on Real-Time Systems, 2018.