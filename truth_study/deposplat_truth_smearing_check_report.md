# DepoFluxSplat Truth Smearing Verification Report

## Document Metadata
* **Created Date:** 2026-08-03
* **Last Updated:** 2026-08-03

## Summary
`DepoFluxSplat`에서 charge deposition이 어떻게 diffusion되고, 어떤 wire에 감지되는지 디버깅 및 시각화를 실행한 기록이다. `DepoFluxSplat.cxx`에 depo/plane별 wire·tick 범위만 담는 경량 JSON 덤프를 추가하고, (2) 그 값을 Python 쪽 wire geometry(`PlaneGeometry`)와 실제 reco blob(cluster graph)에 대조하는 `scripts/truth_smearing_check.ipynb`를 만들어, wire 인덱스 수준의 정성적 검증과 시각화를 완료했다. 

정량적 charge-분포 비교(Plan §3 Success Criteria의 binning 검증)는 이번 범위에서 수행되지 않았다 — 상세는 5절 참고.

---

## Execution

### `DepoFluxSplat.cxx`
- 소스코드에 json파일로 저장하는 로직을 추가해, 가벼운 방식으로 디버깅을 진행하였다.
- `wire-cell-toolkit/gen/src/DepoFluxSplat.cxx`: `#include "WireCellUtil/Persist.h"` 추가. `operator()` 안에 depo/plane 하나당 한 레코드(`depo_id, plane, wire_min, wire_max, tick_min, tick_max`)를 `Json::Value debug_records`에 push, `operator()` 끝에서 `Persist::dump("deposplat_debug.json", debug_records, true)`로 파일당 하드코딩된 이름으로 덤프.
- 단일 `operator()` 호출(=단일 이벤트)마다 truncate 모드로 덮어쓰므로, 여러 이벤트를 도는 job에서는 마지막 이벤트만 남는 한계가 있음(단일 point-depo 테스트라 문제 없음).

### 물리 배경 문서화 — `docs/truth_study/depotransform_vs_deposplat.md` 

`DepoTransform`(main reco 경로, field/electronics response convolution + deconvolution) vs `DepoFluxSplat`(truth 경로, response 없이 diffusion Gaussian을 직접 splat)의 구조를 비교 정리. `GaussianDiffusion`이 두 경로가 공유하는 core math(erf 기반 bin-적분 Gaussian)임을 명시.

### Truth-vs-reco 크기 차이 관찰

시각화 결과 DepoFluxSplat 3평면 strip의 교집합이 실제 reco blob보다 눈에 띄게 크다는 현상을 관찰했다. 근본원인 분석(`smear_tran`/`smear_long`가 유효 σ를 키우는 실측치, `wirecell-gen morse-*`를 통한 그 값들의 물리적 출처, reco 파이프라인 threshold의 역할)은 `DepoTransform`/`DepoFluxSplat` 물리 비교라는 이 문서의 성격에 맞춰 `depotransform_vs_deposplat.md`에 정리했다.

### Blob이 depo 크기에 가까워지는 메커니즘 (img 파이프라인 분석)

위 관찰의 반대 방향 질문 — "smearing으로 넓어진 raw 파형(`gaussN`/`wienerN`)이 `img` 파이프라인 안에서 어떻게 다시 depo 크기에 가까운 blob으로 좁혀지는가" — 을 `img/src/` 소스와 `docs/img_3d_imaging_workflow/` 문서 시리즈 기준으로 분석했다. 결론: 이 좁힘은 `ChargeSolving`/`Deghosting`이 아니라 그보다 먼저 일어나는 두 단계에서 대부분 결정된다.

1. **`MaskSlice`의 진폭 threshold**(`docs/img_3d_imaging_workflow/01a_maskslice.md`): `GaussianDiffusion`이 만드는 patch는 depo 전하 $Q$를 보존하도록 정규화되므로(`depotransform_vs_deposplat.md` §3.2) 피크 진폭은 $\propto Q/\sigma_{\text{eff}}$ — smearing이 $\sigma_{\text{eff}}$를 키우면 피크 진폭이 반비례해서 낮아진다. `MaskSlice`(이 프로젝트 `nthreshold=3.6`, RMS 기반)는 **고정 진폭 threshold**로 활성 wire를 판정하므로, 활성 폭은 $x_{\text{thresh}} \sim \sigma_{\text{eff}}\sqrt{\ln(\text{peak}/\text{threshold})}$로 자란다 — `DepoFluxSplat`의 `nsigma=3.0` 컷(폭 $=3\sigma_{\text{eff}}$, $\sigma_{\text{eff}}$에 선형 비례)보다 훨씬 느리게 커지는 형태라, smearing이 넓히는 만큼을 상당 부분 상쇄한다. 이 단계는 `GridTiling`보다 먼저 일어나므로(`IFrame -> MaskSlice -> ISlice -> GridTiling`), blob이 만들어지기도 전에 이미 활성 wire 범위가 좁혀져 있다.
2. **`GridTiling`의 3평면 교집합**(`01_slicing_and_tiling.md` §3): U/V/W는 서로 다른 각도의 투영이라 smearing이 넓히는 방향도 평면마다 다르다. blob은 세 평면 strip의 교집합이므로, 한 평면의 여분 영역이 다른 두 평면(다른 각도라 그 방향으로는 좁은)과 안 겹치면 자동으로 제거된다.
3. **`ChargeSolving`(LASSO, `02_charge_solving.md`)/`Deghosting`(`InSliceDeghosting`/`ProjectionDeghosting`, `03_deghosting.md`)은 살아남은 blob의 모양(strip/corner)을 바꾸지 않는다** — blob 전체의 전하를 0으로 밀거나(LASSO 정규화) ghost로 판정해 통째로 제거할 뿐, 형태는 `GridTiling`이 만든 그대로 유지된다. 즉 이 단계들은 주로 **여러 후보 blob 사이의 모호성**(궤적이 여러 개 겹칠 때 생기는 조합적 ghost)을 정리하는 역할이라, 고립된 단일 point depo처럼 경쟁 blob이 없는 이번 테스트 케이스에서는 크기 수렴에 거의 관여하지 않는다 — 트랙이 여러 개 겹치는 복잡한 이벤트에서 더 중요해질 것으로 예상된다.

**검증 남은 부분**: 특히 1번의 정량적 관계($x_{\text{thresh}}$가 실제로 raw depo의 $T$에 가깝고 $\sigma_{\text{eff}}$보다는 훨씬 좁다는 것)는 아직 `deposplat_debug.json`의 wire별 `charge`/`time` 배열(§`DepoFluxSplat.cxx` 항목에서 추가된 필드)로 실측 검증되지 않았다 — 다음 액션으로 남겨둠.

### [2026-08-11] 정량적 threshold 폭 검증 (`test_point_depo`)

위에서 남겨둔 검증을 새 WCT 실행이나 C++ 수정 없이 완료했다 — `deposplat_debug.json`의 `wires[].charge`(wire별 시계열, 지금까지 미사용)와 `maskslice_debug.json`을 `true_slice` 창으로 제한한 실측 peak-charge를 `scripts/truth_smearing_check.ipynb`에서 직접 대조했다.

**결과**: `test_point_depo` 기준 wire 폭 비교.

| Plane | DepoFluxSplat(이론, $3\sigma$ 컷) | MaskSlice(threshold 통과) | GridTiling | reco blob |
|---|---|---|---|---|
| U | 6 | 3 | 3 | 3 |
| V | 6 | 2 | 2 | 2 |
| W | 3 | 2 | 2 | 2 |

MaskSlice의 활성 wire 폭이 이미 GridTiling/최종 reco blob 폭과 정확히 같다 — 이 고립된 단일 point depo 케이스에서는 **narrowing이 사실상 전부 MaskSlice의 진폭 threshold 단계에서 결정되며, `GridTiling`의 3평면 교집합은 추가로 아무것도 더 잘라내지 않는다**. Peak-charge-vs-wire-index 플롯(`results/pdhd/truth_smearing_check/threshold_width_verification.png`)에서 이론적 Gaussian 프로파일(회색)이 고정 $3\sigma$ 경계(점선)까지 뻗어 있는 반면, 실측 진폭 threshold 경계(파선)는 그보다 안쪽에서 곡선의 꼬리를 잘라낸다는 것을 직접 확인했다 — 1번 메커니즘의 예측과 정성적으로 일치.

이 분석에 쓰인 두 함수(`deposplat_peak_charge_by_wire`, `maskslice_peak_charge_by_wire`)와 표/플롯 셀은 `truth_smearing_check.ipynb`의 "Quantitative Threshold-Width Verification" 절에 있다.

**다음 단계**: 이 결과는 point 하나에 대한 것이다. `point_depo_grid_v3`에서 이미 플래그된 outlier/`dy_extremes` 케이스 몇 개에 대해 (a) `sh_scripts/run_single_point.sh`로 개별 재실행해 디버그 JSON을 얻고 (b) 같은 분석을 반복해, 이 관계가 depo 위치/거리에 따라 달라지는지(특히 outlier로 분류된 만큼 이 좁힘 메커니즘이 정상 케이스와 다르게 작동하는지) 확인하는 것이 자연스러운 후속 작업이다.

### [2026-08-11] `point_depo_grid_v3` outlier 4건으로 확장 검증

위 다음 단계를 수행했다. `results/pdhd/point_depo_grid_v3/outliers/`에서 대표 케이스 4개(서로 다른 region, 서로 다른 drift 거리)를 골라 같은 검증을 반복했다.

**작업 중 발견한 것 — grid config에는 애초에 `DepoFluxSplat`이 없었음**: `wire-cell-cfg/pdhd/wct-sim-nf-sp-img-bdf-grid.jsonnet`(point_depo_grid_v3가 쓰는 config)에는 `deposplat_node`가 아예 인스턴스화돼 있지 않았다 — single-point config(`wct-sim-nf-sp-img-bdf.jsonnet`)에는 있는 패턴이 grid config에는 빠져 있어서, `deposplat_debug.json`이 grid job에서는 한 번도 생성된 적이 없었다. Single-point config의 `deposplat_node` 패턴을 grid config에 그대로 옮겨(`DepoSetFanout` multiplicity 3→4, truth 브랜치 추가) 이 문서 작성 시점에 고쳤다 — jsonnet 설정 변경만으로 충분했고 WCT 재빌드는 필요 없었다.

**Depo-to-blob 매칭도 새로 필요했음**: grid job은 34개 depo(같은 Y,Z, X만 다름)를 한 프레임에 담으므로, `deposplat_debug.json`의 `depo_id` 필드(`DepoFluxSplat`이 보고하는 `depo->id()`)가 전부 0으로 찍혀 각 record를 물리적 depo에 대응시킬 수 없었다. 대신 (1) `deposplat_debug.json`의 자체 `nominal_depo_time`을 기준으로 매칭을 시도했으나 프레임 경계 근처 depo에서 어긋났고, (2) `position_center_comparison.py`의 `match_blobs_to_depos`가 이미 쓰는 `depos["t"][depo_idx] + T_OFFSET(314.5us)*1000` 기준으로 reco blob을 시간 클러스터링해 매칭하니 4개 케이스 전부 잔차 2.1us 이내(슬라이스 폭 1개 미만)로 정확히 들어맞았다 — outlier 플래그를 만든 바로 그 매칭 로직을 재사용한 것이다.

**결과**: 4개 outlier 전부 `test_point_depo`와 동일한 패턴을 보였다 — `MaskSlice` 활성 wire 폭이 `reco blob` 폭과 정확히 일치했다(GridTiling 3평면 교집합의 추가 narrowing 없음), DepoFluxSplat 이론값보다는 항상 좁았다.

| Region | depo_idx | Plane | DepoFluxSplat | MaskSlice | reco blob |
|---|---|---|---|---|---|
| y584p205_z38p346 (원거리 drift) | 29 | U/V/W | 7/6/3 | 2/3/1 | 2/3/1 |
| y15p098_z161p371 | 15 | U/V/W | 6/6/3 | 3/3/2 | 3/3/2 |
| y15p098_z161p371 (anode 근접) | 0 | U/V/W | 6/6/2 | 3/3/1 | 3/3/1 |
| y75p004_z145p993 | 2 | U/V/W | 6/5/2 | 3/2/1 | 3/2/1 |

depo29(원거리, 큰 diffusion) 플롯에서는 이론적 Gaussian 프로파일이 test_point_depo보다 훨씬 넓게 퍼져 있는데도(피크 대비 꼬리가 완만) threshold가 그만큼 더 공격적으로 잘라내, 결과적으로 활성 wire가 여전히 몇 개뿐이라는 걸 시각적으로 확인했다 — outlier로 분류될 만큼 위치가 어긋난 케이스도 "좁힘의 근본 원인"은 정상 케이스와 다르지 않다는 뜻이다. Outlier 자체(거리 편차)의 원인은 이 좁힘 메커니즘 바깥, 즉 blob-depo 위치 매칭/시간축 쪽에 있을 가능성이 높다는 것이 이번 검증의 함의다.

**참고**: 이 4건은 정상적인 노이즈 필터(NF) 환경에서도 `maskslice_debug.json`이 실제 신호와 무관한 다른 채널들의 threshold-crossing을 폭넓게 포함한다는 것도 확인했다(같은 `tick_span` 창에 U/V 평면 800개 이상 채널) — `anode_smearing_check.ipynb`가 NF를 끈 상태에서 관찰했던 것과 같은 현상이 grid job(NF 켜짐)에서도 나타난다. Reco blob 자체의 wire 범위(±10 wire)로 필터링해야 의미 있는 MaskSlice 폭 비교가 가능했다 — `scripts/outlier_threshold_check.py`의 `BOUNDS_PAD` 처리.

이 절의 재현 스크립트는 `scripts/outlier_threshold_check.py`(모듈 docstring에 매칭 로직 상세)이며, plot은 `results/pdhd/point_depo_grid_v3/outlier_threshold_check/`에 저장된다.

---

## Related Documents

* **Parent docs:**
* **Child docs:**
* **Sibling docs:**
  - [depotransform_vs_deposplat.md](./depotransform_vs_deposplat.md): DepoTransform vs DepoFluxSplat 물리 비교. §4가 truth-vs-reco 크기 차이(`smear_tran`/`smear_long`, threshold)의 근본원인 분석 본문.
  - [../geometry/wirecell_wires_reference.md](../geometry/wirecell_wires_reference.md): §4의 `Correction` 체인 상세.
  - [../img_3d_imaging_workflow/01a_maskslice.md](../img_3d_imaging_workflow/01a_maskslice.md): `MaskSlice` 진폭 threshold — "Blob이 depo 크기에 가까워지는 메커니즘" 절 1번 근거.
  - [../img_3d_imaging_workflow/01_slicing_and_tiling.md](../img_3d_imaging_workflow/01_slicing_and_tiling.md): `GridTiling` 3평면 교집합 — 같은 절 2번 근거.
  - [../img_3d_imaging_workflow/02_charge_solving.md](../img_3d_imaging_workflow/02_charge_solving.md): `ChargeSolving`(LASSO)이 blob 모양을 바꾸지 않고 전하만 0으로 미는 이유.
  - [../img_3d_imaging_workflow/03_deghosting.md](../img_3d_imaging_workflow/03_deghosting.md): `InSliceDeghosting`/`ProjectionDeghosting`이 blob 자체를 제거하는 방식(모양 변경이 아님).
