# DepoTransform vs DepoFluxSplat

## 컴포넌트 개요

소스: `../wire-cell-toolkit/gen/src/DepoTransform.cxx`, `../wire-cell-toolkit/gen/src/DepoFluxSplat.cxx`

### 공통점
- `IDepoFramer`: `IDepoSet -> IFrame`, config는 `anode`/`nsigma`/`tick` 공유.
- Diffusion 메커니즘은 `GaussianDiffusion`(`GausDesc`, erf 기반 bin 적분) 사용 
- 출력 trace의 키를 `wire->channel()`로 매김, `face->planes()` 순회 구조 동일.
- Depo/plane이 `nsigma` 범위를 벗어나면(시간 또는 pitch) 스킵.

### 차이점
| | DepoTransform | DepoFluxSplat |
|---|---|---|
| Field/electronics response | 있음(`ImpactTransform`, FFT, `IPlaneImpactResponse`) | 없음 — diffusion된 전하 그대로 |
| Pitch 방향 binning | 촘촘한 impact binning(`pimpos->impact_binning()`, `ImpactTransform` 경유) | wire 단위 `region_binning()` 직접 사용 |
| Depo 선택 방식 | `Aux::sensitive()` (face 단위) | `find_face()` bounding-box 체크 |
| 시간 기준 | depo 시각을 그대로 사용(response-plane 기준) | `nominal_depo_time`으로 `response_plane/speed`만큼 이동(collection-plane 등가 시각) |
| 추가 스미어링 옵션 | 없음 | `smear_long`/`smear_tran`(plane별, quadrature 합산) |
| Depo 훅 | `modify_depo()` virtual(오버라이드 가능) | 없음 |
| 출력 형태 | sparse만 | sparse 또는 dense(`m_sparse` 설정) |
| 전하 부호 | 계산된 그대로(부호 있음) | 강제로 양수(`abs`) |
| Fluctuation | 옵션(`m_rng`) | 항상 없음(`set_sampling` 호출 시 `0` 하드코딩) |
| 계산 비용 | plane당 FFT, 무거움 | patch를 직접 채움, 저렴 |

## `GaussianDiffusion` Detail —  Common diffusion Algorithm

소스: `../wire-cell-toolkit/gen/inc/WireCellGen/GaussianDiffusion.h`,`gen/src/GaussianDiffusion.cxx`.

`DepoTransform`과 `DepoFluxSplat` 둘 다 이 클래스를 완전히 동일한 방식으로 사용하여 depo 하나를 (pitch축, 시간축) 2D Gaussian 전하 분포로 바꾸고, 주어진 binning 격자 위에 적분을 통해 전하량을 계산한다.

이후 단계 각각의 컴포넌트는 다르게 동작하게 된다: `DepoTransform`은 이 patch를 `ImpactTransform`에 넘겨 field/electronics response와 FFT Convolution을 추가로 수행하고, `DepoFluxSplat`은 이 patch를 그대로(abs만 씌워서) 채널에 splat한다. 즉 diffusion 과정은 두 경로가 정확히 동일한 메커니즘을 공유하고, diffusion된 전하가 실제로 wire에 유도되는 물리를 모델링하느냐만 다르다.

### `GausDesc` — 1차원 Gaussian

`center`(중심)와 `sigma`(폭) 두 값을 통해 1차원 가우시안 분포를 구현하는 구조체.
- 이 구조체를 이용해 시간 축과 wire pitch 축을 독립적으로 기술하며, `GaussianDiffusion`은 이 둘을 축으로 받아 outer product를 만든다.
- `Depofluxsplat`에서 `time_desc`(시간축)와 `pitch_desc`(pitch축) 둘 다 이 타입으로 선언
- **`distance(x)`**: `(x-center)/sigma` — x가 몇 시그마 떨어져 있는지. 두 호출자 모두 "depo가 시간창/wire plane을 nsigma 밖으로 완전히 벗어났는지" 체크할 때 씀.
- **`binint(start, step, nbins)`** — 핵심 수학. 각 bin의 경계에서 Gaussian CDF(`erf` 기반)를 계산하고 인접한 두 경계의 차를 취해서, **그 bin 안에 실제로 들어가는 확률질량(정확한 적분값)**을 구한다. Bin 중심에서 PDF를 한 번 점 샘플링하는 것과 다름 — 좁은 bin에서도 정확하다.

```cpp
x = (start + step*ind - center) / (sqrt2*sigma);
erfs[ind] = 0.5*erf(x);                    // CDF
bins[ibin] = erfs[ibin+1] - erfs[ibin];    // bin 적분값
```

### 3.2 `GaussianDiffusion::set_sampling()` — 실제 2D patch를 만드는 곳

시그니처: `set_sampling(tbin, pbin, nsigma, fluctuate, weightstrat)`. `DepoTransform`/
`DepoFluxSplat` 둘 다 `(전체 시간 Binning, plane의 pitch/wire Binning, m_nsigma, ...)`를
넘겨 호출한다.

1. **메모이제이션**: `if (m_patch.size() > 0) return;` — 이미 계산했으면 재계산 안 함(한 `GaussianDiffusion` 객체당 patch는 한 번만 만들어짐).
2. **시간축 샘플링**:
   - `tval_range = time_desc.sigma_range(nsigma)` -> `[center - nsigma*sigma, center +
     nsigma*sigma]`.
   - `tbin_range = tbin.sample_bin_range(...)` -> 그 범위를 절대 bin 인덱스로 변환
     (0~nbins로 clip).
   - `m_toffset_bin = tbin_range.first` — patch의 0번째 열이 대응하는 **절대** tick
     bin(`DepoFluxSplat`의 `tbin0`이 이 값).
   - `tvec = time_desc.binint(...)` — 그 범위의 각 bin에 대한 정규화된 확률질량 벡터.
3. **Pitch축 샘플링**: 완전히 같은 패턴으로 `m_poffset_bin`, `pvec` 계산
   (`DepoFluxSplat`의 `pbin0`).
4. **가중치(`m_qweights`)**: `weightstrat==1`이면 그냥 상수 `0.5`로 채움,
   `weightstrat==2`면 `GausDesc::weight()`로 선형 보간 가중치 계산(인접한 두
   impact-response 위치에 전하를 나눠 배분하기 위한 것 — **`ImpactTransform`이 field
   response를 보간할 때만 의미가 있고**, `DepoFluxSplat`은 이 `weights()`를 아예
   호출하지 않으므로 사실상 안 쓰인다 — `set_sampling(..., 1)`로 넘기는 `1`은 형식적).
5. **Outer product**:
   ```cpp
   for ip in pvec, for it in tvec:
       ret(ip, it) = pvec[ip] * tvec[it];   // 두 독립 Gaussian의 곱 = 2D Gaussian
   ```
   두 축이 통계적으로 독립이라고 가정하고 단순 곱으로 2D 분포를 만든다.
6. **전하로 정규화**: `ret *= depo_charge / raw_sum` — outer product는 확률(합이 1에
   가까운 값)이므로, depo의 실제 전하량(`depo->charge()`)에 맞게 스케일. `raw_sum`이
   1이 아닌 이유는 nsigma로 절단해서 꼬리 부분이 빠졌기 때문(nsigma가 클수록
   1에 가까워짐).
7. **Fluctuation(옵션)**: `fluctuate`가 non-null이면 각 셀을 이항분포
   (`binomial(|depo_charge|, 셀 비율)`)로 재추출해 shot noise 성격의 전하 요동을
   추가하고 다시 총합을 depo 전하에 맞게 정규화. `DepoTransform`은 이 인자에 실제
   `IRandom`(옵션)을 줄 수 있고, `DepoFluxSplat`은 항상 `0`(=끔)을 넘긴다.
8. 결과가 `m_patch`(row=pitch bin, col=tick bin, `Array::array_xxf`)에 저장됨.

### 3.3 나머지 접근자

- `patch()` — 위에서 만든 2D 배열 반환.
- `weights()` — `m_qweights` 반환(`ImpactTransform` 전용, `DepoFluxSplat`은 미사용).
- `toffset_bin()`/`poffset_bin()` — patch의 (0,0)이 절대 격자 어디에 대응하는지
  (`DepoTransform`/`DepoFluxSplat` 둘 다 이 값으로 절대 위치를 복원).
- `clear_sampling()` — `m_patch`/`m_qweights` 비움(메모리 회수용). `DepoFluxSplat`은
  `GaussianDiffusion` 객체를 plane 루프마다 새로 만들어서(`gd`가 로컬 변수) 명시적
  호출 없이도 사실상 자동으로 버려진다.


## DepoTransform Detail

1. `operator()`: `tbins`(readout window) 생성 후 `face -> plane` 순회.
2. Face의 sensitive volume 안에 있는 depo만 선택(`Aux::sensitive`).
3. Plane별: `modify_depo()` 훅 호출 후, depo마다 `BinnedDiffusion_transform::add()`
   (sigma_time = `extent_long/drift_speed`, sigma_pitch = `extent_tran`).
4. `ImpactTransform`: diffusion된 전하를 impact 위치별 field response(`IPlaneImpactResponse`)
   + electronics response와 FFT로 콘볼루션 -> wire별 파형.
5. Wire별로 0이 아닌 구간만 잘라(`Waveform::edge`), `wire->channel()`로 매핑해 `SimpleTrace` 생성.
6. 출력: `SimpleFrame`(sparse trace만, trim된 상태).

## DepoFluxSplat Detail

### Config 

`default_configuration()`(L40-65)이 기본값을 나열하고, `configure()`(L83-157)가 실제 `Configuration`(jsoncpp `Json::Value`)에서 값을 읽어 멤버 변수에 채운다. `get_n()`(L67-81, 파일-scope `static` 헬퍼)은 `smear_long`/`smear_tran` 두 키를 U/V/W 3-plane에 각각 적용될 `std::vector<double>`로 통일하는 용도로 쓰인다: 스칼라(`isDouble()`)면 그 값을 3번 복제, 정확히 3-원소 배열이면 그대로, 그 외(빈 값 등)면 `{0,0,0}`을 반환한다.

| Config 키 | 기본값 | 읽는 곳 | 저장되는 멤버 | 의미 |
|---|---|---|---|---|
| `field_response` | `""`(빈 문자열 placeholder) | L86-92 | `m_speed`, `m_origin` | 지정하면 `IFieldResponse` 컴포넌트의 `field_response().speed`/`.origin`을 먼저 읽어옴(drift speed/response-plane 위치의 1차 출처) |
| `drift_speed` | 없음(미지정 시 위 `m_speed` 유지) | L93-96 | `m_speed` | `field_response`에서 읽은 speed를 명시적으로 override(나중에 설정되므로 우선순위 더 높음) |
| `response_plane` | 없음(미지정 시 위 `m_origin` 유지) | L97-100 | `m_origin` | `field_response`의 origin을 명시적으로 override |
| `anode` | `[]`(빈 배열 placeholder, 실제론 문자열 하나로 씀) | L102-104 | `m_anode` | `Factory::find_tn<IAnodePlane>(...)`로 anode 컴포넌트 확보 |
| `tick` | `0.5us` | L107 | (로컬 `wtick`) | readout 시간창의 tick 폭 |
| `window_start` | `0` | L108 | (로컬 `wstart`) | 시간창 시작(절대시각) |
| `window_duration` | `8096 * 0.5us` = `4048us` | L109 | (로컬 `wduration`) | 시간창 길이 |
| — | — | L110-111 | `m_tbins` | 위 세 값으로 `nwbins = wduration/wtick` 계산 후 `Binning(nwbins, wstart, wstart+nwbins*wtick)` 생성. `operator()` 전체에서 시간축 "격자"로 재사용됨 |
| `sparse` | `m_sparse`(헤더 기본값) | L114 | `m_sparse` | true면 `SparseAccumulator`, false면 `DenseAccumulator` 사용(2.5 참고) |
| `nsigma` | `m_nsigma`(헤더 기본값) | L117 | `m_nsigma` | Gaussian 절단 기준 시그마 수 |
| `process_planes` | `{0,1,2}`(전부) | L119-127 | `m_process_planes` | 배열로 주면 그 plane 인덱스만 처리하도록 완전히 교체 |
| `smear_long` | `0.0` | L129 (`get_n()` 경유) | `m_smear_long`(길이-3 벡터) | plane별 추가 시간축 스미어링 크기 |
| `smear_tran` | `0.0` | L130 (`get_n()` 경유) | `m_smear_tran`(길이-3 벡터) | plane별 추가 pitch축 스미어링 크기 |
| `reference_time` | `0` | L133-134 | `m_reftime` | 출력 프레임 절대시각 계산 시(`m_tbins.min() - m_reftime`) 빼는 임의 오프셋 — 시간축 0점 재정의용 |
| `time_offsets` | `[]`(비면 전부 0) | L136-149 | `m_tick_offsets`(길이-3, tick 정수 단위) | 3-원소 배열이면 각 원소를 `/wtick`으로 나눠 tick 단위로 저장(시간 단위 값을 정수 tick으로 변환). 3개도 아니고 비어있지도 않으면 `ValueError` throw. `operator()`에서 각 plane trace의 최종 tick 위치에 더해지는 plane별 보정치 |

### `intersect()` (L168-183, 파일-scope `static`)

`intrange_t = std::pair<int,int>`로 반열림구간을 정의. `intersect()`함수는 이 반열림 구간 2개(`r1, r2`)를 받아 겹치는 구간을 구한다.
- `r1`이 `r2`보다 완전히 앞서면(`r1.second <= r2.first`) `(r2.first, r2.first)`(길이 0, `r2`의 시작점에서 길이가 0인 빈 구간) 반환.
- `r1`이 `r2`보다 완전히 뒤에 있으면(`r1.first >= r2.second`) `(r2.second, r2.second)` 반환.
- 그 외(겹침)에는 `(max(r1.first,r2.first), min(r1.second,r2.second))`.

이후 `operator()`에서 patch의 pitch/tick 범위를 plane/프레임의 유효 범위로 잘라내는 데 두 번 쓰인다.

### `operator()` (L258-417)

**depo -> face -> plane** 3중 순회로 루프를 돌며 처리
- `find_face()`(L159-166): `m_anode->faces()`를 순회하며 각 face의 `sensitive()` bounding box(`bb`)가 depo 위치를 포함하는지(`bb.inside(depo->pos())`) 확인, 처음 맞는 face를 반환. 못 찾으면 `nullptr`.
- DepoTransform의 `Aux::sensitive()`(여러 depo를 한번에 필터링하는 free 함수)와 달리, 이건 depo 하나마다 반복 호출되는 멤버 함수라는 점이 다르다(속도보다 단순성 우선).
- depo 루프 안에서, plane마다 독립적으로 시간축(`time_desc`)과 pitch축(`pitch_desc`) 두 개의 `GausDesc`를 선언하고,`GaussianDiffusion`이 그 outer product를 wire-binning 격자 위에 정확히 적분한 2D `patch`를 만든 뒤, 유효 범위로 잘라서(`intersect`) wire별로 잘게 썰어 accumulator에 누적하는 것이 전부다. field/electronics response 콘볼루션이나 FFT는 전혀 없다.


```cpp
out = nullptr
if (!in) { EOS 처리, ++m_count, return true }             

accum = m_sparse ? SparseAccumulator : DenseAccumulator    

for depo in in->depos():                                    // L278-408 (depo 루프, 최외각 루프)
    if (!depo) continue                                       
    face = find_face(depo); if (!face) continue             // depo 포함 face 찾기 
    nominal_depo_time = depo->time() + m_origin/m_speed     // depo가 collection plane 도달 시간 계 

    for plane in face->planes():                              // L299-407 (해당 face의 plane 루프)
        iplane = plane->planeid().index()
        if (iplane < 0) { ++nplanes_skipped; continue }        
        if (iplane not in m_process_planes) continue         

        // L311-317, 각 plane별 extra smearing 계산 및  longitudinal sigma 수정
        sigma_L = depo->extent_long()  [+ smear_long, quadrature]   
        time_desc = GausDesc(nominal_depo_time, sigma_L/m_speed)     
        if (time_desc가 m_tbins 범위를 nsigma 밖으로 완전히 벗어남) continue

        // L332-334, wire 단위
        wbins = plane->pimpos()->region_binning()              
        sigma_T = depo->extent_tran()  [+ smear_tran, quadrature] 
        center_pitch = pimpos->distance(depo->pos())           
        pitch_desc = GausDesc(center_pitch, sigma_T)           
        if (pitch_desc가 wbins 범위를 nsigma 밖으로 완전히 벗어남) break

        gd = GaussianDiffusion(depo, time_desc, pitch_desc)      // L357
        gd.set_sampling(m_tbins, wbins, m_nsigma, /*fluctuate=*/0, /*weightstrat=*/1)  // L358
        patch = gd.patch(); pbin0 = gd.poffset_bin(); tbin0 = gd.toffset_bin()  // L361-376

        p_range = intersect({pbin0, pbin0+patch.rows()}, {0, wbins.nbins()})    // L368-369, 2.4 참고
        if (p_range 비어있음) { ++nplanes_skipped; continue }    // L370-373
        t_range = intersect({tbin0, tbin0+patch.cols()}, {0, m_tbins.nbins()})  // L379-380
        if (t_range 비어있음) { ++nplanes_skipped; continue }    // L381-384

        for pbin in p_range:                                    // L386-406
            iwire = wires[pbin]; chid = iwire->channel()          // L387-389
            charge = patch.row(pbin-pbin0).segment(t_range 상대구간)  // L391-398
            charge = abs(charge)                                  // L402-403
            accum->add(chid, t_range.first + m_tick_offsets[iplane], charge)  // L405

out = accum->frame(in->ident(), m_tbins.min() - m_reftime, m_tbins.binsize())  // L410
log->debug(...); ++m_count; return true                          // L411-416
```

### Accumulator 계층 (L185-254, 파일-scope 구조체)

`operator()`가 채널별 전하를 모으는 방식을 다형성으로 선택하기 위한 작은 계층.

- **`Accumulator`**(추상 기반, L186-193): 순수 가상 인터페이스
  `add(chid, tbin, charge)`, `frame(ident, time, tick) -> IFrame::pointer`, `ntraces()`.
- **`SparseAccumulator`**(L197-208): `ITrace::vector itraces`를 그냥 계속 `push_back`.
  같은 채널에 여러 depo/plane 조각이 들어와도 **합치지 않고** 별개의 trace로 쌓인다 — 즉
  하나의 채널이 여러 개의 (짧은) `SimpleTrace`로 쪼개져 출력될 수 있음. `frame()`은 그
  벡터로 바로 `SimpleFrame` 생성.
- **`DenseAccumulator`**(L220-254): `std::map<int, SharedSimpleTrace> traces`(chid -> trace)
  로 채널당 trace 하나만 유지하며 실제로 값을 더한다.
  - `add()`(L224-244): 해당 chid의 기존 trace `tp`가 없으면(`!tp`) 그냥 새 `SimpleTrace`를
    만들어 저장하고 끝.
  - 이미 있으면: 기존 range `have = [tp->tbin(), tp->tbin()+oldcharge.size())`와 새로 들어온
    range `want = [tbin, tbin+charge.size())`의 합집합 `need`를 계산(`union_intrange`,
    L214-218). `need`가 `have`보다 넓어져야 하면(`need.first < have.first` 또는
    `need.second > have.second`) 그 넓이의 새 벡터 `newcharge`(0으로 채움)를 만들고,
    옛 값을 올바른 offset(`have.first-need.first`)에 복사, 새 값을 올바른 offset
    (`want.first-need.first`)에 `std::plus<float>()`로 **더해서** 병합한 뒤
    `tp->charge() = newcharge`로 교체. 즉 같은 채널에 겹치는 시간대의 전하가 여러 번
    들어오면 실제로 합산됨(Sparse와의 결정적 차이).
  - `frame()`(L246-252): map을 순회하며 trace들을 모아 `SimpleFrame` 생성.
  - `ntraces()`: `traces.size()`(맵 크기 = 채널 수).


## 4. Truth-vs-Reco 크기 차이: `smear_tran`/`smear_long`와 threshold

`scripts/truth_smearing_check.ipynb`의 시각화에서 `DepoFluxSplat` 3개 평면(U/V/W) strip의 교집합이 실제 reco blob(NF+SP+img를 거친 cluster graph의 blob)보다 눈에 띄게 크게 나오는 현상을 관찰했다. 두 가지 독립적인 원인이 결합되어 있다.

### 4.1 `smear_tran`/`smear_long`가 `DepoFluxSplat`의 유효 σ를 인위적으로 키움

`wire-cell-cfg/pdhd/wct-sim-nf-sp-img-bdf.jsonnet:251-260`의 `DepoFluxSplat` 설정:
```
"smear_long": [2.691862363980221, 2.6750200122535057, 2.7137567141154055],
"smear_tran": [0.7377218875719689, 0.7157764520393882, 0.13980698710556544]
```
이 값들은 주석에 명시된 대로 "sigproc가 유발하는 추가 퍼짐(the extra spread the sigproc induces)"을 재현하려고 튜닝된 값이다. `DepoFluxSplat.cxx:340-345`에서 depo 자신의 `T`(`extent_tran()`)와 **quadrature 합**으로 더해진다:
```cpp
sigma_T = sqrt(sigma_T*sigma_T + extra*extra);   // extra = smear_tran[iplane] * wire_pitch
```
`sigma_L`도 `smear_long`을 같은 방식(`extra = smear_long * tick * drift_speed`)으로 더한다(2.2절 L311-317).

실측 예시(`data/pdhd/test_point_depo`의 point depo, raw $T=1.702$mm, wire pitch U/V≈4.669mm, W≈4.792mm):

| Plane | wire pitch | extra=smear_tran×pitch | 실제 사용된 $\sigma_T$ | raw $T$ 대비 |
|---|---|---|---|---|
| U | 4.669mm | 3.444mm | 3.842mm | 2.26배 |
| V | 4.669mm | 3.342mm | 3.750mm | 2.20배 |
| W | 4.792mm | 0.670mm | 1.829mm | 1.07배 |

U/V는 raw $T$보다 유효 $\sigma_T$가 2배 이상 크고, W는 거의 영향이 없다 — U/V의 `smear_tran`이 W보다 훨씬 크기 때문이다(4.2절 원인 참고).

### 4.2 `smear_tran`/`smear_long`의 물리적 출처

이 값들은 `wirecell-gen morse-*` 커맨드(`wire-cell-python/wirecell/gen/morse.py`)로 측정한다. `generate()`가 diffusion이 0(`sigma=0`)인 point depo를 격자 패턴("morse" 패턴)으로 만들어 실제 sim+SP 파이프라인에 통과시키고, `frame_peaks()`가 결과 파형에서 tick 방향/channel 방향 피크를 Gaussian fit해 폭을 측정한다. 입력 depo의 diffusion이 0이므로, 측정된 폭은 순수하게 아래 세 단계에서 온 것이다 — `DepoFluxSplat`은 이 세 단계를 전부 건너뛰므로(§2.2), 원래는 이 폭을 가지지 않는다.

1. **Field response convolution** (`DepoTransform`의 `BinnedDiffusion_transform`/`ImpactTransform`): 하나의 impact 위치에서 나온 전하도 field response 파형을 통해 주변 여러 wire에 걸쳐 유도 전류를 만든다. Induction plane(U/V)은 field response가 bipolar 모양으로 인접 wire까지 길게 퍼지는 반면, collection plane(W)은 상대적으로 국소적이다 — 4.1절 실측값에서 U/V의 `smear_tran`이 W보다 훨씬 큰 이유가 이것이다.
2. **Electronics(cold electronics) response convolution**: 평면 종류와 무관하게 시간 방향(tick)으로 파형을 shaping한다. `smear_long` 값이 U/V/W 전부 거의 동일(2.69, 2.68, 2.71)한 게 이 때문이다 — electronics shaping은 wire 종류에 상관없이 공통으로 걸리는 효과다.
3. **`OmnibusSigProc`의 deconvolution 정규화 필터**: `decon_2D_init`/`decon_2D_tighterROI`/`decon_2D_looseROI`/`decon_2D_hits`(`docs/wirecell_sigproc_reference.md`)에서 field×electronics response로 raw 신호를 나눠(deconvolution) 원래 전하를 복원하는데, 노이즈 발산을 막기 위해 `Wiener_tight`/`Wiener_wide`(HF) × `ROI_tighter`/`ROI_tight`/`ROI_loose`(LF) 필터를 곱한다. 이 필터들은 유한한 대역폭을 가지므로, response를 완벽히 역산하더라도 결과는 delta 함수가 아니라 필터 자체의 유효 폭만큼 다시 뭉개진 신호가 된다.

### 4.3 reco 파이프라인의 threshold가 tail을 잘라냄

`DepoFluxSplat`의 `nsigma=3.0`(`gen/inc/WireCellGen/DepoFluxSplat.h:88`) 컷은 threshold 없는 이상적 경계다 — charge 크기와 무관하게 ±3σ 안이면 전부 포함된다. 반면 실제 reco blob은 두 곳에서 threshold를 거친 뒤에 남은 신호로만 만들어진다.

- `OmnibusSigProc`의 ROI 정제 단계(`decon_2D_tighterROI`/`decon_2D_tightROI`/`decon_2D_looseROI`): deconvolved 신호 중 "의미 있는 신호가 있는 구간"만 ROI로 남기고 나머지는 0으로 clip한다.
- `img.slicing`의 `MaskSlices`(`nthreshold`): ROI가 남긴 신호 중에서도 다시 한 번 진폭이 `nthreshold`를 넘는 채널/틱만 "활성"으로 취급해 time slice를 만든다. `CLAUDE.md` §2.2 Stage 2가 "noise-free true frame에서는 `nthreshold`를 0에 가깝게 낮춰라"고 미리 적어둔 게 이 단계를 가리킨다.

이 두 threshold를 통과한 wire/tick만 `img.tiling`으로 넘어가 blob polygon을 만들므로, 최종 reco blob은 (4.1절의 smearing으로 부풀려진) `DepoFluxSplat`의 이론적 ±3σ 범위보다 작게 형성된다. `img.tiling` 자체도 단순 wire-index 범위 겹침이 아니라 특정 tick에서 3개 평면이 동시에 신호를 갖는지 기하학적으로 맞춰 확인하므로, threshold가 없더라도 `DepoFluxSplat` 3개 평면 strip을 통째로 겹친 것보다는 좁아진다.

**결론**: `DepoFluxSplat`의 wire 범위가 reco blob보다 큰 것은 버그가 아니라, (a) sigproc의 추가 퍼짐을 흉내내려고 넣은 `smear_tran`/`smear_long`이 raw diffusion sigma보다 유효 sigma를 키우고, (b) `nsigma=3.0` 컷은 threshold 없는 순수 이론적 경계인 반면 reco blob은 SP ROI + `MaskSlices` threshold를 통과한 부분만 남긴 결과이기 때문이다.


## 6. 아직 안 채운 것 / 다음에 정리할 것

- [ ] DepoFluxSplat만 `nominal_depo_time` 보정을 하는 이유(DepoTransform은 왜 안 함)?
- [ ] impact binning vs wire binning: induction plane 정확도 차이가 실제로 얼마나 나는지?
- [ ] `modify_depo()` — 실제로 이걸 오버라이드하는 곳이 있는지(BlobDepoFill?).
