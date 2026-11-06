# `img.pre_proc` — MaskSlice 이전 전처리 (`CMMModifier` -> `FrameMasking` -> `ChargeErrorFrameEstimator`)

## Summary

`OmnibusSigProc`가 만든 `IFrame`이 `MaskSlice`(`01a_maskslice.md`)로 들어가기 전, `img.jsonnet`의 `img.pre_proc(anode, aname)`(`wire-cell-cfg/pdhd/img.jsonnet:11-95`)에 정의된 세 컴포넌트를 거친다.
```
`CMMModifier` -> `FrameMasking` -> `ChargeErrorFrameEstimator`
```

셋 다 `IFrameFilter`(`IFrame -> IFrame`)이며, trace의 실제 파형 대신 **채널 품질 메타데이터(bad-channel 정보)와 오차(error) 추정치**를 정리해 `MaskSlice`가 요구하는 `wiener_tag`/`charge_tag`/`error_tag`/`summary_tag` 네 태그가 갖춰진 프레임을 만드는 것이 목적이다.

소스: `img/src/CMMModifier.cxx`, `img/src/FrameMasking.cxx`, `img/src/ChargeErrorFrameEstimator.cxx`(및 대응 `img/inc/WireCellImg/*.h`), `img/docs`의 `docs/components/{CMMModifier,FrameMasking,ChargeErrorFrameEstimator}.md`.

---

## 1. 파이프라인에서의 위치

```
OmnibusSigProc(SP)
   v
IFrame(gaussN, wienerN, decon_chargeN, tight_lfN, ...)
   | CMMModifier      -- "bad" ChannelMaskMap 정리/보강 (trace는 그대로 통과)
   v
IFrame(변화 없음, cmm만 갱신)
   | FrameMasking     -- gaussN/wienerN에서 "bad" 구간을 0으로. trace_tags에 없는 태그는 이 단계에서 드롭됨(§3)
   v
IFrame(gaussN, wienerN만 남음)
   | ChargeErrorFrameEstimator  -- gaussN의 ROI 길이 기반으로 새 gauss_errorN 태그 생성, 나머지 태그는 유지
   v
IFrame(gaussN, wienerN, gauss_errorN)
   | MaskSlices(img.slicing)   <-- 01a_maskslice.md
   v
ISlice
```

세 컴포넌트는 `img.jsonnet:94`에서 `g.pipeline([cmm_mod, frame_masking, charge_err], "uboone-preproc")`로 이 순서 그대로 묶인다. 참고로 같은 함수 안에 `frame_quality_tagging`(`FrameQualityTagging` 컴포넌트, `img.jsonnet:59-82`)이 정의되어 있지만 `ret` 파이프라인에는 포함되지 않는다 — 정의만 되고 실제로는 안 쓰이는 죽은 지역 변수다.

---

## 2. `CMMModifier` — bad-channel 메타데이터 정리 (`img/src/CMMModifier.cxx`)

Trace 데이터는 건드리지 않고, 프레임의 ChannelMaskMap(CMM) `"bad"` 항목(`chid -> [[start,end), ...]`)만 수정한다. 4개 독립 규칙을 순서대로 적용한다.

1. **Continuity(shorted-wire) veto** (`cont_ch_*`): 아직 bad로 안 잡힌 채널의 좌우(±1 또는 ±2) 이웃이 둘 다 bad면 shorted로 추정해 이웃의 bad range를 복사.
2. **Direct veto** (`veto_ch_*`): 지정 채널 범위를 조건 없이 `[start, end)` bad로 마킹.
3. **Dead-channel-by-charge 판정** (`dead_ch_*`): 채널 그룹마다 `trace_tag`(기본 `"gauss"`) 신호가 `dead_ch_charge`를 넘는 샘플 수를 세어, `dead_ch_ncount` 미만이면 그룹 전체를 bad로.
4. **Boundary alignment** (`ncount_org`/`org_llimit`/`org_hlimit`): 기존 bad range의 시작을 그 이하의 가장 가까운 `org_llimit`으로, 끝을 그 이상의 가장 가까운 `org_hlimit`으로 스냅 후 재병합. **`ncount_org=0`(기본값)이면 이 단계가 기존 bad range를 지우기만 하고 다시 채우지 않는다**(재생성 루프에 아무것도 안 들어감) — 규칙 4를 켜지 않고 두면 CMMModifier가 상류의 bad-channel 정보를 통째로 날리는 부작용이 있다.

**이 프로젝트 config**(`img.jsonnet:19-31`)는 1~3번을 전부 주석 처리하고 4번만 `ncount_org=1, org_llimit=[0], org_hlimit=[8500]`로 켠다. 이 범위는 뒤의 `MaskSlices`의 `min_tbin=0`/`max_tbin=8500`(`img.jsonnet:109-110`)과 정확히 같아서, 의도는 "이벤트 어딘가에서 조금이라도 bad로 잡힌 채널은 전체 이벤트를 죽은 채널로 취급"하는 coarsening이다.

이 컴포넌트의 또 다른 실질적 역할: `cmm.find(m_cm_tag)==cmm.end()`이면 경고만 하고 `auto& cm = cmm[m_cm_tag];`로 **빈 항목을 생성**한다(`std::map::operator[]`). 즉 입력 프레임에 `"bad"` CMM이 아예 없어도 CMMModifier를 거치면 항상 (비어있더라도) 존재하게 된다 — 이는 다음 단계 `FrameMasking`이 `"bad"`가 없으면 예외를 던지기 때문에(§3) 파이프라인 순서상 CMMModifier가 반드시 먼저 와야 하는 이유이기도 하다.

---

## 3. `FrameMasking` — bad 구간 zero-out, 그리고 태그 필터링 (`img/src/FrameMasking.cxx`)

`cm_tag`(기본 `"bad"`) CMM을 읽어(**없으면 `RuntimeError` throw** — `:61-63`), `trace_tags`에 지정된 각 태그의 trace마다 bad range와 겹치는 tick을 0으로 채운다(`:75-83`, `std::generate(..., []{return 0;})`).

**중요한 부작용**: 출력 프레임은 `in->traces()` 전체가 아니라 **`trace_tags`에 열거된 태그의 trace만**으로 새로 구성된다(`:68-89`, `sfout`는 `out_traces`만 사용). 게다가 프레임을 다시 태깅할 때도 `for (auto tag : m_trace_tags)`만 순회하고(`:95-100`), `in->trace_tags()` 전체를 훑는 pass-through 루프가 없다. 즉 **입력 프레임에 있던 다른 모든 trace tag(`decon_chargeN`, `tight_lfN`, `loose_lfN`, ... )는 이 단계에서 조용히 사라진다.**

이 프로젝트 config(`img.jsonnet:84-92`)는 `trace_tags: ['gaussN', 'wienerN']`만 지정하므로, `FrameMasking` 이후로는 `gaussN`/`wienerN` 두 태그만 살아남는다. `MaskSlices`가 요구하는 태그가 정확히 이 둘 + `gauss_errorN`(§4에서 새로 추가됨)뿐이므로 결과적으로 딱 맞아떨어지지만, 만약 나중에 다른 태그(예: `decon_chargeN`)를 img 단계까지 가져가고 싶다면 이 `trace_tags` 목록에 명시적으로 추가해야 한다는 점을 기억해야 한다.

---

## 4. `ChargeErrorFrameEstimator` — ROI 길이 기반 charge 오차 추정 (`img/src/ChargeErrorFrameEstimator.cxx`)

`intag`(기본 `"gauss"`)의 각 trace에서 **0이 아닌 연속 구간(ROI)**을 찾고(`:137-173`), ROI 길이(tick 수)로 오차값을 룩업해 그 ROI 전체에 균일하게 채운 새 trace를 만들어 `outtag`(기본 `"gauss_error"`)로 추가한다. 원본 trace/태그는 그대로 보존한다(§3의 `FrameMasking`과 달리 `in->trace_tags()` 전체를 pass-through, `:222-230`).

알고리즘(`error_traces()`, `:117-187`):
1. Trace의 채널 `chid`로부터 `m_anode->channel(chid)->planeid().index()`로 평면(U/V/W)을 얻는다.
2. `charge[iq] != 0`인 첫 tick에서 ROI 시작, `charge[iq] == 0`(또는 trace 끝)에서 ROI 종료.
3. `roilen = ROI 길이(tick)`를 `time_limits = [12, 800]`(이 프로젝트 config, tick 단위) 범위로 clip.
4. `errs[roilen]`(평면별 미리 로드된 ROI-길이 -> 오차 룩업 테이블, `m_bins`로 이산화)을 `fudge_factor[plane] / sqrt(rebin)`으로 스케일해 그 ROI의 모든 tick에 채움.
5. ROI 밖(charge==0)은 오차 0.

**이 프로젝트 config**(`img.jsonnet:19-31`):
```
intag: "gaussN", outtag: "gauss_errorN",
rebin: 4,
fudge_factors: [2.31, 2.31, 1.1],   // [U, V, W]
time_limits: [12, 800],              // ticks
errors: WaveformMap{filename: "microboone-charge-error.json.bz2"},
```
룩업 테이블 자체가 **MicroBooNE에서 가져온 파일**이라는 점에 주의 — PDHD 고유 캘리브레이션이 아니라, U/V 평면에 `2.31`, W 평면에 `1.1`이라는 fudge factor로 근사 보정한 값이다(`sp.jsonnet`의 "BIG FAT FIXME: we are taking from uboone" 주석과 같은 종류의 이식 흔적). `rebin=4`는 룩업 테이블의 원래 시간 binning과 이 프로젝트의 tick 크기 차이를 보정하는 계수이며, `configure()`(`:110-112`)가 `rebin`이 실제 파일의 `waveform_period()/tick`과 일치하는지 확인해 안 맞으면 `log->error`로 경고한다.

---

## 5. 이 프로젝트 테스트 데이터에 대한 실질적 함의

`wct-sim-nf-sp-img-bdf.jsonnet`의 `reco_fork`는 `nf_pipes[n]`(실제 채널 품질 정보를 만드는 NF 단계)를 주석 처리해 두었다(이전 대화에서 확인). 그 결과:

- **`CMMModifier`/`FrameMasking`은 사실상 no-op**이다 — `"bad"` CMM을 채워주는 상류 단계가 없어 `cm`이 비어 있고(§2 1~3번 규칙도 이 config에서는 꺼져 있음), `FrameMasking`도 마스킹할 대상이 없다. 다만 §3에서 설명한 **trace tag 필터링 부작용**(`gaussN`/`wienerN`만 남기는 것)은 CMM 유무와 무관하게 항상 일어난다는 점은 구분해야 한다.
- **`ChargeErrorFrameEstimator`는 NF 여부와 무관하게 항상 의미 있게 동작**한다 — `gaussN` trace의 ROI 구조(신호가 있는 구간)만 보고 동작하므로, 노이즈 필터링 유무와 무관하다.
- **하지만 `GridTiling`이 만드는 blob의 "모양"(어떤 wire가 활성인가) 자체는 이 세 단계와 무관**하다 — 그건 전적으로 `MaskSlice`의 boolean thresholding(`wienerN`/`gaussN` 진폭 vs `wienerN`의 RMS summary, `01a_maskslice.md`)이 결정하며, `gauss_errorN`은 그 판정에는 안 쓰이고 `ISlice`의 `Measurement`(값+불확실성) 중 **불확실성** 성분으로 저장되어 나중에 `ChargeSolving`의 LASSO whitening(`02_charge_solving.md`)에 쓰인다. 즉 **DepoFluxSplat 데이터로 blob의 기하학적 모양만 Python으로 재현**하려면 이 pre_proc 세 단계(CMMModifier/FrameMasking/ChargeErrorFrameEstimator)는 전부 건너뛰고 `MaskSlice`의 thresholding 로직만 봐도 되지만, 나중에 `ChargeSolving`이 매기는 전하값/불확실성까지 재현하려면 `ChargeErrorFrameEstimator`의 ROI-오차 모델이 필요해진다.

---

## Related Documents

- [00_pipeline_overview.md](./00_pipeline_overview.md): 전체 파이프라인 개요, 이 저장소의 `img.jsonnet` 실제 설정.
- [01a_maskslice.md](./01a_maskslice.md): 이 문서 바로 다음 단계인 `MaskSlice`의 thresholding 로직.
- [01_slicing_and_tiling.md](./01_slicing_and_tiling.md): `MaskSlice` 이후 `RayGrid`/`GridTiling` 타일링과 blob clustering.
- [../wirecell_sigproc_reference.md](../wirecell_sigproc_reference.md): `gaussN`/`wienerN`/`decon_chargeN` 태그를 만드는 `OmnibusSigProc`(이 문서의 입력 출처).
