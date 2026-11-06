# `MaskSlice` — thresholding slicer
* **Created Date:** 2026-08-11
* **Last Updated:** 2026-08-11

## Summary

소스: `img/src/MaskSlice.cxx`, `img/inc/WireCellImg/MaskSlice.h`

`IFrame`(channel x time 파형)을 `tick_span` 단위 시간창(`ISlice`)으로 묶으면서, 각 채널의 신호가 threshold(`nthreshold`)를 넘는지로 채널별 활성 여부를 판정하는 컴포넌트. 이 컴포넌트는 Sigproc의 출력이 아닌 `CMMModifier`/`FrameMasking`/`ChargeErrorFrameEstimator` 전처리를 거친 `IFrame` 데이터를 받는다. ([00b_preprocessing.md](./00b_preprocessing.md) 참조)

여기서 만든 `ISlice`는 이후 `RayGrid` 타일링으로 넘어가 `IBlob`이 된다. ([01_slicing_and_tiling.md](./01_slicing_and_tiling.md) §3 참조)

## 전체 동작 흐름
1. **Preperations**: 먼저 전체 시간 구간에 대해 결과물이 될 시간창들을 신호 유무와 무관하게 미리 준비해둔다 — 다운스트림에서 시간창이 하나도 누락되지 않도록 하기 위함이다.
2. **채널을 평면 그룹으로 분류**: 채널들을 소속 평면(U/V/W)에 따라 세 그룹(활성 평면 / 항상 채워지는 평면 / 죽은 채널 마스크 평면)으로 나누며, 어느 그룹에 속하느냐에 따라 이후 처리 경로가 완전히 갈린다.
3. **Thesholding**: 활성 평면 채널만 매 시점(tick)마다 신호가 유의미한지 판정을 받고, 그 판정을 통과한 시점만 지정된 폭의 시간창으로 묶여 신호가 누적된다 — 판정이 먼저이고 시간창으로의 편입은 그 결과에 종속된다. 나머지 두 그룹(항상 채워지는 평면, 마스크 평면)은 이 판정 과정 없이 정해진 값으로 해당 시간 구간을 그대로 채운다.

각 단계의 세부 로직은 아래 §Preperations(1, 2)·§Thesholding(3)에서 다룬다.

## Config

`MaskSliceBase::default_configuration()`(`MaskSlice.cxx:63-116`)이 선언하는 config 항목과 그 기본값. `configure()`(`:118-172`)가 이 값들을 멤버 변수(`MaskSlice.h:61-83`)로 읽어들인다.

| Config key | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `anode` | string | `""` (필수) | `IAnodePlane` 인스턴스의 type:name 토큰. `Factory::find_tn<IAnodePlane>()`으로 조회하며, 못 찾으면 예외를 던진다(`:120`). Channel → plane 매핑(`IChannel::planeid()`)에 쓰인다. |
| `tick_span` | int | `4` | 몇 개의 tick을 묶어 하나의 `ISlice`로 만들지(slice 폭). |
| `wiener_tag` | string | `"wiener"` | 활성 여부 판정(`thresholding()`)에 쓰는 wiener-필터 trace의 태그. |
| `summary_tag` | string | `"wiener"` | Threshold 계산에 쓰는 `trace_summary`(채널별 RMS)의 태그. 명시적으로 config에 없으면 `wiener_tag` 값으로 대체된다(`:123`). |
| `charge_tag` | string | `"gauss"` | Slice에 누적되는 실제 전하값 trace의 태그. |
| `error_tag` | string | `"gauss_error"` | 전하 불확실성 trace의 태그. |
| `active_planes` | int 배열 (`WirePlaneId::index()`, 0/1/2 = U/V/W) | `[0, 1, 2]` | Tick 단위 thresholding 판정을 받는 평면 목록. 여기 없는 평면 소속 channel은 판정 자체를 건너뛴다. |
| `dummy_planes` | int 배열 | `[]` (빈 배열) | 모든 tick에 무조건 `(dummy_charge, dummy_error)`를 채우는 평면 목록. |
| `masked_planes` | int 배열 | `[]` (빈 배열) | CMM(`"bad"` 태그) 마스크 구간에 `(masked_charge, masked_error)`를 채우는 평면 목록. |
| `dummy_charge` | double | `0.0` | Dummy plane에 채우는 고정 전하값. |
| `dummy_error` | double | `1e12` | Dummy plane에 채우는 고정 오차값 — charge solving에서 사실상 무시되게 하는 크기. |
| `masked_charge` | double | `0.0` | Masked plane에 채우는 고정 전하값. |
| `masked_error` | double | `1e12` | Masked plane에 채우는 고정 오차값. |
| `nthreshold` | double 배열(U/V/W 순) | `[3.6, 3.6, 3.6]` | `threshold = nthreshold[plane] * summary[idx]`(실측 RMS 배수)의 계수. |
| `default_threshold` | double 배열(U/V/W 순) | `[2351.276, 3346.576, 2271.896]` (uBooNE 튜닝값, `5.87819e2*4`/`8.36644e2*4`/`5.67974e2*4`) | `summary[idx]`가 정확히 0(실측 RMS 미제공)일 때만 쓰이는 fallback threshold. |
| `min_tbin` | int | `0` | Slice 생성 범위의 시작 tick. `min_tbin`/`max_tbin`이 둘 다 0이면 입력 `IFrame`의 trace 범위에서 자동으로 결정한다(`:110-113`, `:250-259`). |
| `max_tbin` | int | `0` (미포함 상한) | Slice 생성 범위의 끝 tick. |

이 저장소가 실제로 재설정해 쓰는 값(`tick_span`/`active_planes`/`masked_planes`/`dummy_planes`/`nthreshold`)은 §Preperations(채널의 평면 그룹 분류)와 §3(fanout)에서 다룬다(`wire-cell-cfg/pdhd/img.jsonnet`의 `img.slicing()`/`multi_active_slicing_tiling`/`multi_masked_2view_slicing_tiling`).

## Preperations

`slice()`(`MaskSlice.cxx:219`) 맨 앞부분에서, thresholding을 하기 전에 먼저 빈 `Img::Data::Slice` 컨테이너들을 시간 구간 전체에 대해 만들어둔다.

**1. tick 범위 결정** (`:247-259`): `min_tbin`/`max_tbin`이 config에서 둘 다 `0`(기본값)이면, `charge_tag` trace 전체를 순회해 각 채널의 `tbin`(시작)과 `tbin + charge().size()`(끝)를 모아 전역 최소/최대로 범위를 잡는다 — 채널마다 trace의 시작 tick과 길이가 다를 수 있기 때문.
```cpp
min_tbin = *std::min_element(tbins.begin(), tbins.end());
max_tbin = *std::max_element(tends.begin(), tends.end());
```

**2. tick 범위 → slicebin 범위 변환** (`:261-263`): `tick_span`으로 나눠 `[min_slicebin, max_slicebin)`을 구한다.
```cpp
const size_t min_slicebin = floor((double)min_tbin/m_tick_span);
const size_t max_slicebin = ceil((double)max_tbin/m_tick_span);
```

**3. 빈 slice 선-생성** (`:265-273`): 이 범위의 모든 `slicebin`에 대해, 활동 여부와 무관하게 `Img::Data::Slice` 객체를 만들어 `svcmap`(slicebin → Slice 포인터 맵)에 채워 넣는다. 주석(`:265`)에 따르면 다운스트림 `BlobSetMerge`의 slice 동기화가 깨지지 않도록 하기 위함이다.
```cpp
for (size_t slicebin=min_slicebin; slicebin < max_slicebin; ++slicebin) {
    const double start = slicebin * m_tick_span * tick;
    auto s = new Img::Data::Slice(tlframe_ptr, slicebin, start, span);
    svcmap[slicebin] = s;
}
```
이때 각 `Slice`는 실제 trace 데이터 없이 프레임 정보(ident/time/tick)만 담은 `TracelessFrame`(`:244`, `:27-52`) 하나를 모든 slice가 공유해서 참조한다 — trace 데이터는 이후 thresholding을 통과한 채널만 `s->sum(...)`으로 개별 slice에 채워진다(§Thesholding).

**참고**: 이 선-생성 루프의 `start`는 `slicebin * m_tick_span * tick`으로, 프레임의 절대 시각(`in->time()`)을 더하지 않은 상대 시각이다. 반면 아래 §Thesholding에서 thresholding 통과 시점에 (아직 `svcmap`에 없어서) 뒤늦게 만들어지는 slice는 `in->time() + slicebin * span`으로 절대 시각을 쓴다(`SLICE_START_TIME_IS_RELATIVE`가 파일 상단(`:12`)에서 `#undef`돼 있어 이 분기가 활성화됨) — 같은 함수 안에서 slice 생성 시점에 따라 `start` 기준이 다르다는 점에 유의.

## 채널을 평면 그룹으로 분류: active / dummy / masked

빈 slice 컨테이너가 준비된 뒤, `slice()`는 채널들을 **세 개의 독립된 루프**로 처리한다(순서대로 active → dummy → masked). 하나의 분류 함수가 먼저 전체 채널을 세 그룹으로 나눠놓는 구조가 아니라, 각 루프가 자신의 config 리스트(`active_planes`/`dummy_planes`/`masked_planes`)만 보고 그 리스트에 해당하는 평면의 채널을 그때그때 걸러낸다.

**1. Active planes** (`:298-315`): `charge_tag` trace를 channel 단위로 순회하며 `IChannel::planeid()`로 평면을 얻고, `active_planes_set`에 없으면 그 channel의 trace 전체를 skip한다.
```cpp
IChannel::pointer ich = m_anode->channel(chid);
auto planeid = ich->planeid();
if (active_planes_set.find(planeid.index()) == active_planes_set.end()) {
    continue;
}
```
통과한 channel만 tick 단위 thresholding으로 넘어간다(§Thesholding) — 값은 `s->sum(ich, {q, e})`으로 **누적**된다.

**2. Dummy planes** (`:383-403`): `m_dummy_planes`에 속한 각 평면 번호에 대해 `Aux::plane_channels(m_anode, plane_index)`로 그 평면의 전체 channel 목록을 가져와, thresholding 없이 전체 slice 범위에 고정값을 채운다.
```cpp
for (int plane_index : m_dummy_planes) {
    auto ichans = Aux::plane_channels(m_anode, plane_index);
    for (auto ich : ichans) {
        for (size_t slicebin = min_slicebin; slicebin < max_slicebin; ++slicebin) {
            ...
            s->assign(ich, {(float)m_dummy_charge, (float)m_dummy_error});
        }
    }
}
```

**3. Masked planes** (`:405-441`): 개별 channel이 아니라 CMM(`in->masks()["bad"]`, channel mask map)을 channel 단위로 순회하며, `planeid`가 `m_masked_planes`에 없으면 skip. 통과하면 그 channel의 bad tbin 구간과 겹치는 slice에만 고정값을 채운다.
```cpp
IChannel::pointer ich = m_anode->channel(chid);
auto planeid = ich->planeid();
if (std::find(m_masked_planes.begin(), m_masked_planes.end(), planeid.index()) == m_masked_planes.end()) {
    continue;
}
```

**배타성은 코드가 아니라 config가 책임진다**: 세 리스트가 겹치지 않도록 강제하는 로직은 없다. Dummy/masked 루프는 `s->assign()`(덮어쓰기)을 쓰므로, 어떤 평면이 실수로 두 리스트에 동시에 들어가면 실행 순서(active → dummy → masked) 상 **나중에 도는 루프의 값이 앞선 값을 덮어쓴다**. 이 저장소의 실제 config는 평면마다 정확히 한 그룹에만 속하도록 설정한다(§3 fanout 표 참고).

## Thesholding

### 두 서브클래스: `MaskSlicer` vs `MaskSlices`

`MaskSlice.h`는 `MaskSliceBase` + 출력 방식이 다른 두 서브클래스로 구성된다.
- **`MaskSliceBase`**(`:45-84`): thresholding/binning 로직(`slice()`, `thresholding()`)과 모든 config 멤버를 갖는 공통 기반 클래스. `IFrameSlicer`/`IFrameSlices` 같은 인터페이스는 상속하지 않는 순수 로직 보관소다.
- **`MaskSlicer`**(`:86-92`, `IFrameSlicer` 상속): `bool operator()(const input_pointer& in, output_pointer& out)` — **1-in-1-out 함수 노드**. `slice()`가 만든 `ISlice::vector` 전체를 `Img::Data::SliceFrame` 컨테이너 하나로 묶어 `out`에 한 번에 반환한다(`MaskSlice.cxx:444-464`).
- **`MaskSlices`**(`:94-100`, `IFrameSlices` 상속): `bool operator()(const input_pointer& in, output_queue& slices)` — **queued-output 노드**. 같은 `slice()` 결과를 순회하며 `ISlice`를 **하나씩** `slices.push_back(...)`으로 큐에 밀어넣는다(`MaskSlice.cxx:466-492`). EOS일 때도 `out=nullptr`을 리턴하는 `MaskSlicer`와 달리, `slices.push_back(nullptr)`로 큐 안에 종료 신호를 넣는다(`:470`).

두 클래스가 호출하는 `slice()`/`thresholding()`(아래)은 완전히 동일하다 — 차이는 **판정 로직이 아니라 결과를 파이프라인 그래프에 어떻게 내보내는가**뿐이다.

| | `MaskSlicer` | `MaskSlices` |
|---|---|---|
| 출력 | 모든 slice를 `SliceFrame` 하나로 묶어 한 번에 | slice 하나씩 큐에 개별 push |
| DFP 그래프 | 1-in-1-out이라 그래프가 단순 | queued-out 노드라 그래프가 좀 더 복잡 |
| 처리 방식 | 다운스트림이 프레임 전체를 한 덩어리로(monolithic) 처리하도록 강제, 메모리 사용 더 큼 | slice 단위로 흘려보내 스트리밍에 가까움 |

이 저장소의 `img.jsonnet:100`(`img.slicing()` 함수)은 `type: "MaskSlices"`를 쓴다 — 즉 slice를 큐에 하나씩 흘려보내는 쪽이다.

### Active plane의 tick 단위 판정

§Preperations에서 active plane으로 분류된 channel만 매 tick마다 `thresholding()`(`MaskSlice.cxx:173-216`)의 판정을 받는다. 2차 판정은 **다른 wire가 아니라 같은 wire, 이웃한 시간 슬라이스**를 보는 것이라는 점에 유의한다.

**1차 — 이 tick 자신의 wiener 신호**(`:179-184`):
```cpp
const auto q_wiener = wiener_charge[qind];
const auto q_gauss  = gauss_charge[qind];
if (q_wiener > threshold) return true;
```

**2차 — 이웃 slice bin의 wiener 평균과 비교**(1차를 통과 못했을 때만 평가, `:185-215`):
```cpp
const int sbin      = qind / tick_span;   // 현재 tick이 속한 slice(=tick_span개 tick 묶음) 번호
const int sbin_next = sbin + 1;
const int sbin_prev = sbin - 1;

// q_next/q_prev: 다음/이전 slice bin의 wiener 평균 (배열 경계 밖이면 0으로 남음)

if ((q_gauss > q_next / 3. && q_next > threshold) ||
    (q_gauss > q_prev / 3. && q_prev > threshold)) {
    is_active = true;
}
```

이 저장소의 `tick_span=4`(`img.jsonnet:103`) 기준 예: `qind=5`면 `sbin=1`(tick 4~7), `sbin_next=2`(tick 8~11), `sbin_prev=0`(tick 0~3)이고, `q_next`는 tick 8~11의 `wiener_charge` 평균이다.

**물리적 의미**: 실제 펄스의 피크가 이웃 slice bin 안에 있고 지금 tick은 그 펄스의 꼬리(tail)만 걸치는 경우를 구제하기 위한 로직이다. 이 tick의 wiener 값만으로는 threshold를 못 넘지만(피크가 아니라 꼬리라서), gauss(전하) 값이 이웃 slice 평균의 1/3 이상이면 "이웃 slice에 있는 진짜 펄스의 연장선"으로 보고 활성 처리한다. 단, 이웃 slice 자체가 threshold를 넘는 진짜 신호일 때만(`q_next > threshold` 또는 `q_prev > threshold`) 적용된다 — 이웃도 신호가 약하면 현재 tick의 gauss 값이 아무리 커도 통과되지 않는다. 이 2단계 로직의 의도는 "본 slice에서는 threshold를 살짝 못 넘었지만, 실제로는 인접 slice와 이어지는 진짜 신호의 tail"인 경우를 살려주는 것으로 보인다.

### Threshold 값 계산

`thresholding()`에 넘겨지는 `threshold`는 다음과 같이 계산된다(`:327-330`):
```cpp
double threshold = m_nthreshold[planeid.index()] * summary[idx];
if (threshold == 0) {
    threshold = m_default_threshold[planeid.index()];
}
```
`summary`(`summary_tag` trace의 `trace_summary`)는 트레이스 파형 자체와는 별개로, `OmnibusSigProc`이 `ROI_formation::get_{u,v,w}plane_rms()`로 계산한 **채널(wire)별 실측 RMS**를 붙여 넘긴 것이다(`sigproc/src/OmnibusSigProc.cxx:1868`, `save_data()`의 `threshold.push_back(perwire_rmses[wire])`). 즉 신호 대 잡음비를 비교하는 게 아니라, **`summary[idx]`가 0이 아니면(실측 RMS가 있으면) `nthreshold * RMS`를 쓰고, 정확히 0이면(RMS 미제공) `default_threshold[plane]`(uBooNE 튜닝값)로 대체**하는 단순 fallback이다. 이 저장소는 `nthreshold: [3.6, 3.6, 3.6]`(`img.jsonnet:114`)을 명시적으로 재설정해 쓰며, 실제로는 거의 항상 RMS가 채워져 있어 `default_threshold` fallback은 드물게만 작동한다.

판정을 통과한 `(charge, error)`는 `s->sum(ich, {q, e})`로 slice에 누적된다 — §Preperations "채널을 평면 그룹으로 분류"에서 이미 언급한 그 `sum()` 호출이다.

## 두 종류의 fanout: `active`/`masked`

### 두 fanout 함수의 구성

`multi_active_slicing_tiling`(`img.jsonnet:158-168`)과 `multi_masked_2view_slicing_tiling`(`:171-182`)은 같은 패턴이다: 서로 다른 `active_planes`/`masked_planes`/`dummy_planes` 조합으로 `slicing()`+`tiling()` 파이프라인을 여러 개 만들고, `FrameFanout`으로 같은 `IFrame`을 각 조합에 복제해 흘려보낸 뒤 `BlobSetMerge`로 결과를 하나로 합친다.

**`multi_active_slicing_tiling`** (`tick_span=4`, 4갈래):

| n | `active_planes` | `masked_planes` | 의미 |
|---|---|---|---|
| 0 | `[0,1,2]` (U,V,W) | `[]` | 3평면 모두 active |
| 1 | `[0,1]` (U,V) | `[2]` (W) | W만 dead-masked |
| 2 | `[1,2]` (V,W) | `[0]` (U) | U만 dead-masked |
| 3 | `[0,2]` (U,W) | `[1]` (V) | V만 dead-masked |

**`multi_masked_2view_slicing_tiling`** (`tick_span=500`, 3갈래, `active_planes=[]`로 thresholding 자체를 끔):

| n | `dummy_planes` | `masked_planes` | 의미 |
|---|---|---|---|
| 0 | `[2]` (W) | `[0,1]` (U,V) | W는 dummy로 무시, U/V는 dead-masked |
| 1 | `[0]` (U) | `[1,2]` (V,W) | U dummy, V/W dead-masked |
| 2 | `[1]` (V) | `[0,2]` (U,W) | V dummy, U/W dead-masked |

### `imgpipe()`의 5가지 모드

`img.jsonnet` 최하단의 `imgpipe(anode, multi_slicing)`(`:299-342`)는 `multi_slicing` 문자열로 선택되는 5가지 파이프라인 변형을 갖는다.

| `multi_slicing` | 구성 |
|---|---|
| `"single"` | 단일 `slicing()`(`active_planes=[0,1,2]`, `tick_span=4`) + `tiling` + `solving` — fanout 없음 |
| `"pdhd1"` | 단일 `slicing()`(`active_planes=[0,1]`, `dummy_planes=[2]`) + `tiling` + `solving("full")` + `dump` |
| `"active"` | `multi_active_slicing_tiling`만 + `solving` + `dump` |
| `"masked"` | `multi_masked_2view_slicing_tiling`만 + `clustering` + `dump` |
| (그 외, default) | `active_fork`와 `masked_fork`를 바깥쪽 `FrameFanout`으로 한 번 더 병렬 실행(`"fan_active_masked"`) |

### 실제로 실행되는 것은 `"single"` 모드뿐

`per_anode(anode)`(`:344-347`)는 `imgpipe(anode, "single")`을 호출하고, 이 저장소의 실제 진입점(`wct-sim-nf-sp-img-bdf.jsonnet:113`, `wct-sim-nf-sp-img-bdf-grid.jsonnet:123`)이 부르는 것도 `img_maker.per_anode(a)`뿐이다. 즉 **위 두 fanout 함수와 default 모드의 이중 fanout은 코드에는 존재하지만, 이 저장소가 실제로 돌리는 파이프라인에서는 쓰이지 않는다** — 실행되는 건 `active_planes=[0,1,2]`, `masked_planes=[]`, `dummy_planes=[]`인 plain `MaskSlices` 하나뿐이다(`img.jsonnet:304`). `GeomClusteringUtil`의 `dead_clus` 정책(`01_slicing_and_tiling.md` §6)도 마찬가지로 `clustering_policy="uboone"`(`:184`)이 쓰이는 한 현재는 실질적으로 관여하지 않는다.

---

## Related Documents

- [00b_preprocessing.md](./00b_preprocessing.md): 이 문서 바로 이전 단계인 `CMMModifier`/`FrameMasking`/`ChargeErrorFrameEstimator` 전처리 상세.
- [01_slicing_and_tiling.md](./01_slicing_and_tiling.md): 이 문서 바로 다음 단계인 `RayGrid`/`GridTiling` 타일링과 `BlobGrouping`/`BlobClustering`/`GeomClusteringUtil` (원래 이 문서가 §1로 속해 있던 문서, 분량 때문에 분리).
- [00_pipeline_overview.md](./00_pipeline_overview.md): 전체 파이프라인 개요.
- [04_constants_and_caveats.md](./04_constants_and_caveats.md): `nthreshold`/`default_threshold` 등 하드코딩 상수 모음.
- [../img_time_slicing_reference.md](../img_time_slicing_reference.md): `tick`/`tick_span`이 slice 폭을 어떻게 결정하는지에 대한 상세 분석.
