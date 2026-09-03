# INNO Ledger

측정 계획 · 이력 · 보정 게이트. INNO NanoLab · INNO Prism의 형제 앱.
설계 확정본: Claude 프로젝트 문서 `nanolab-INNO-Ledger-측정매트릭스-설계확정-0903.md` 참고.

## 실행

`INNO Ledger.html`을 그대로 더블클릭(`file://`)하면 화면 구조는 보이지만
**Google 로그인은 동작하지 않습니다** (브라우저 보안 정책). 로그인까지 테스트하려면:

```
cd "이 폴더"
python -m http.server 8080
```
후 브라우저에서 `http://localhost:8080/INNO Ledger.html` 접속.
(NanoLab처럼 GitHub Pages에 올려 배포할 수도 있습니다.)

로그인은 INNO NanoLab과 **동일한 Firebase 프로젝트(inno-nanoil) · 동일 Google 계정**을 씁니다.
lot은 NanoLab이 저장한 것을 읽기 전용으로 그대로 보여줍니다.

## 진행 상황

- **Phase 1** — 로그인, lot 읽기, 프리셋(측정종류/여기구성/검출채널) 시드·CRUD
- **Phase 2** — 매트릭스 뷰(측정종류 축 전환 + 압축 표시), 셀 계획/해제, 계획 템플릿 CRUD + 선택 lot 일괄 적용
- Phase 3 이후 — Run 티켓, 보정 게이트, 첨부/용량, 자동 판정, 소급 등록, 대시보드 (설계 확정본 §9 참고)

## 데이터 모델 (현재까지)

```
users/{uid}/
  lots/{id}            NanoLab 소유 (읽기 전용)
  opt_meta/seed_v1, seed_v2_templates   시딩 완료 플래그
  opt_presets/{id}     kind: meas | ex | em
  opt_cells/{cellId}   cellId = lotId|measTypeId|exPresetId|emPresetId
  opt_templates/{id}   name, description, rows:[{measTypeId, exPresetId, emPresetId, priority, note}]
