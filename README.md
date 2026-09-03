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
- **Phase 3** — 마운트 세션(수동 시작/종료), 보정 세션 CRUD, Prism 공용 렌즈 목록 연동(RTDB REST 읽기), 셀 모달에서 Run 티켓 발급(마운트·보정 게이트 + override) · 취소, 셀 상태 자동 전이(계획됨→예약)
- **Phase 4** — 측정 데이터 폴더 연결(File System Access API) + h5 스캔, 파일명 규칙 기반 후보 매칭(측정종류·시각), Run에 파일 연결(로컬 지문) + 클라우드 첨부(gzip+base64 청크) + 용량 계기판
- Phase 5 이후 — 자동 판정(Prism JSON 임포트), 소급 등록, Spot 매칭, 대시보드 (설계 확정본 §9 참고)

## 데이터 모델 (현재까지)

```
users/{uid}/
  lots/{id}            NanoLab 소유 (읽기 전용)
  opt_meta/seed_v1, seed_v2_templates   시딩 완료 플래그
  opt_meta/settings    { lensDbUrl }  Prism 공용 렌즈 목록 RTDB URL (설치별 설정)
  opt_presets/{id}     kind: meas | ex | em
  opt_cells/{cellId}   cellId = lotId|measTypeId|exPresetId|emPresetId
  opt_templates/{id}   name, description, rows:[{measTypeId, exPresetId, emPresetId, priority, note}]
  opt_mounts/{id}      specimen, lotId, startedAt, endedAt, note
  opt_calib/{id}       at, lambdaNm, lens:{key,maker,model,mag,na,medium}, coefficient, transmission,
                       umPerPx, cameraPitch, areaBasis, densityMode, pathNote, validUntil
  opt_runs/{id}        cellId, lotId, mountId, specimen, calibId, calibStatus, issuedAt, issuedBy,
                       file:{name, relPath, bytes, mtime, hashPrefix, attached, blobChunks, blobBytes}
  opt_blob/{runId}_{k} idx, total, data (gzip+base64 청크, 문서당 ≤700,000자)
```

## Phase 4 메모

- **h5 스캔은 파일명 규칙만 읽습니다**: 브라우저에서 HDF5 바이너리를 파싱하는 라이브러리를 쓰지 않았기 때문에, INNO Optics 저장 규약 `YYMMDD_HHMMSS_측정종류.h5`의 파일명만으로 시각·측정종류를 추출합니다. 측정종류 슬러그가 `measTypeId`와 정확히 일치하지 않으면 후보로 뜨지 않습니다(퍼지 매칭 없음). h5 속성(attr)에 run_id를 심는 것은 Phase 7(INNO Optics storage.py 쪽 작업) 이후에나 가능하므로, "자동 매칭"은 현재 "측정종류 정확 일치 + 시각 근접도 정렬"로 구현되어 있습니다 — run_id 매칭이 아닙니다.
- **stage:{x,y,z} 필드는 채우지 않았습니다**: 설계 확정본 §9 남은 미결에는 "Phase 4가 h5 파생 필드를 채워야 Spot 매칭이 동작한다"는 메모가 있었지만, 이것도 HDF5 attr 파싱이 필요해 이번 Phase 범위에서 제외했습니다. Spot 매칭(§6)은 아직 동작하지 않으며, 이 필드를 언제 채울지는 Phase 7 이후로 다시 계획해야 합니다.
- **저장은 2단계(T1 로컬 지문 + T3 클라우드 첨부)**: 파일을 "연결"하면 SHA-256 해시(앞뒤 64KB + 크기) 16자만 클라우드에 저장됩니다(T1). 여기에 더해 "클라우드에 첨부"를 누르면(또는 연결 모달에서 "연결과 동시에 첨부" 체크박스로 즉시) gzip 압축 후 base64로 인코딩해 `opt_blob` 컬렉션에 Firestore 문서당 700,000자 이하로 쪼개 저장합니다(T3). 압축 후 크기가 1.4MB를 넘으면 첨부는 거부되고 T1 링크만 유지됩니다(용량 보호).
- **폴더로 스캔한 파일만 나중에 재첨부할 수 있습니다**: "연결" 시점에 폴더가 연결되어 있지 않아 파일 선택창(`<input type=file>`)으로 수동 선택한 경우, 그 `File` 객체는 브라우저 메모리에만 있어 연결 이후에는 다시 접근할 수 없습니다. 이런 파일은 연결 시점에 "연결과 동시에 첨부" 체크박스로 즉시 첨부해야 하며, 나중에 Run 행의 "클라우드에 첨부" 버튼을 눌러도 실패 토스트만 뜹니다(재연결/재스캔 안내). 폴더를 연결해 스캔한 파일은 `relPath`로 다시 찾을 수 있어 언제든 재첨부가 가능합니다.
- **용량 계기판은 자체 기준 600MB**: Firestore 무료 한도(1GiB)는 NanoLab과 공유되므로, `opt_blob` 총합이 600MB를 넘지 않도록 자체적으로 넉넉히 잡은 값입니다. 실제 Firestore 하드 한도가 아니며, 80% 초과 시 경고 문구만 표시합니다. 자동 노후화(aging)·삭제 정책은 이번 Phase에서 구현하지 않았습니다(현재 사용량 추세로는 한도까지 여유가 큽니다).
- **썸네일 미구현**: Phase 5에서 Prism 결과 JSON을 임포트하면 썸네일에 해당하는 데이터가 JSON 안에 이미 있을 가능성이 높아, h5 바이너리를 직접 파싱하지 않고도 썸네일을 채울 수 있을 것으로 보고 의도적으로 미루었습니다.
- **테스트**: `window.showDirectoryPicker`를 `entries()` 비동기 제너레이터를 구현한 가짜 객체로 스텁하고, 실제 `File`/`Blob`/`crypto.subtle`/`CompressionStream`은 그대로 사용해(Mock 아님) 해시·압축 로직을 실제 경로로 검증하는 15개 시나리오로 확인했습니다. Firestore 문서 부분 병합(`{merge:true}` + dot-path 키)이 중첩 맵을 올바르게 갱신하는지도 가짜 Firestore 하네스에 `applyDotMerge`를 추가해 검증했습니다.

## Phase 3 메모

- **게이트**: 셀 모달에서 Run 티켓을 발급하기 전, (1) 활성 마운트 세션 존재 여부, (2) 이 여기구성(λ)에 대해 유효기한이 지나지 않은 보정 세션 존재 여부를 확인합니다. 둘 중 하나라도 없으면 경고를 표시하고, "위 경고를 확인했고 그대로 발급합니다" 체크박스를 켜야만 발급 버튼이 동작합니다.
- **보정 연결**: 매칭되는(같은 λ) 보정 세션이 있으면 드롭다운에서 골라 연결할 수 있고, 만료된 세션도 표시(선택 시 calibStatus="expired")됩니다. 선택 안 하면 calibStatus="none"으로 발급됩니다.
- **Run 취소**: 발급된 Run 티켓은 셀 모달에서 개별 취소할 수 있고, 해당 셀에 남은 다른 Run이 없으면서 셀 상태가 "예약"이었다면 자동으로 "계획됨"으로 되돌립니다.
- **Prism 렌즈 연동은 읽기 전용**: Prism의 `lensDbUrl`은 설치(로컬 PC)마다 다른 설정값이라 하드코딩할 수 없습니다 — "보정" 탭에 Ledger 자체 설정으로 같은 URL을 붙여넣게 했습니다. REST GET(`{url}/lenses.json`)으로 1회 스냅샷만 불러오며, Prism처럼 EventSource 실시간 구독은 하지 않습니다("새로고침" 버튼으로 갱신). `lensKey()` 정규화 규칙은 Prism의 `src/app.js`와 동일하게 포팅했습니다.
- **테스트**: Playwright로 로그인을 우회(state.fb를 인메모리 fake Firestore로 대체)하고, 실제 `Sync.issueRun/cancelRun/startMount/endMount/saveCalib`를 그대로 호출하는 10개 시나리오로 검증(게이트 차단/통과, 발급→취소 후 셀 상태 롤백, 마운트 시작/종료, 보정 세션 저장·렌즈 자동입력). 라이트/다크 스크린샷으로 스타일도 확인했습니다.
