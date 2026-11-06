# WireCell-Study

Wire-Cell Toolkit(WCT) 알고리즘/데이터 모델을 공부하며 정리한 레퍼런스 문서 모음이다. 특정 프로젝트에 종속되지 않는, 여러 프로젝트에서 공유해서 참고할 수 있는 문서만 모은다 — 실행 가능한 코드나 실험 결과가 아니라 **문서 전용** 저장소다.


## 레이아웃

```
WireCell-Study/
├── wirecell_sigproc_reference.md        # OmnibusSigProc(디컨볼루션/ROI) 파이프라인 레퍼런스
├── wirecell_gen_reference.md            # wirecell.gen 패키지(depo 생성/드리프트) 인벤토리
├── wirecell_depo_reference.md           # Depo 데이터 모델, .npz 포맷, C++ 파이프라인(gen/iface/aux/sio)
├── wirecell_img_reference.md            # wirecell.img 패키지 인벤토리
├── wirecell_params_jsonnet_reference.md # Jsonnet params/simparams 상속, 단위 체계, det.volumes/faces 컨벤션
├── wirecell_sim_field_reference.md      # sim field 분해: nimpacts/fluctuate/continuous/fixed/depo_toffset/ductor/reframer
├── img_time_slicing_reference.md        # img의 ISlice 시간창 폭(tick x tick_span) 결정 방식
│
├── img_3d_imaging_workflow/             # WCT img 3D 파이프라인 단계별 walkthrough
│   ├── 00_pipeline_overview.md
│   ├── 00b_preprocessing.md
│   ├── 01_slicing_and_tiling.md
│   ├── 01a_maskslice.md
│   ├── 02_charge_solving.md
│   ├── 03_deghosting.md
│   └── 04_constants_and_caveats.md
│
├── geometry/                             # 와이어/민감볼륨 지오메트리 레퍼런스
│   ├── wirecell_wires_reference.md      # wirecell.util.wires 스키마/persist
│   └── wirecell_sensitive_volume.md     # 3D 민감볼륨 경계: wire store JSON vs simparams.jsonnet vs AnodePlane.cxx
│
└── truth_study/                          # DepoTransform vs DepoFluxSplat 비교 분석
    ├── deposplat_truth_smearing_check_report.md
    └── depotransform_vs_deposplat.md
```

## 사용하는 프로젝트

- `wirecell-img-evaluation`: `docs/` 아래 심볼릭 링크로 이 저장소를 직접 참조. `CLAUDE.md`에서 이 README를 자동으로 읽어들이도록 연결돼 있다.
