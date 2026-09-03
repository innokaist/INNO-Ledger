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
- **Phase 5** — INNO Prism의 power scan 분석 JSON 임포트(Run.prism), 자동 게이트 채점(Run.autoGate, 현재 power_scan 1종), 사람 판정 UI(채택/참고/폐기 + 고정 어휘 사유 태그) → 셀 상태 "판정대기"·"성공"·"재측정필요" 자동 전이, Prism JSON 원시 데이터로 그리는 로그-로그 스파크라인
- **Phase 6** — 소급 등록 마법사(스캔된 h5를 측정종류·날짜·시간간격으로 그룹핑해 그룹 단위로 lot·시편·조합 일괄 지정), 대시보드(판정대기·재측정필요·보정없음·마운트미지정 카운터 + 측정종류별 현황표), 아카이브 내보내기(대표 Run 원본을 lot/측정종류/조합 폴더 구조로 로컬에 복사)
- Phase 7 이후 — Spot 매칭, INNO Optics h5 attr 추가, Prism 보정 자동 매칭 (설계 확정본 §9 참고)

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
                       file:{name, relPath, bytes, mtime, hashPrefix, attached, blobChunks, blobBytes},
                       prism:{importedAt, importedFile, engine, paReport, paSource, paGridLimited,
                              threshold, saturation, background, model, censored, quality, decades,
                              ptsPerDecade, minReps, ciRelHalfWidth, spark},
                       autoGate:{verdict, checks:[{id,label,ok,na,value}]},
                       verdict:{state, tags, note, by, at}
  opt_cells/{cellId}   (Phase 5 추가) repRunId — 채택된 대표 Run
  opt_blob/{runId}_{k} idx, total, data (gzip+base64 청크, 문서당 ≤700,000자)
```

Phase 6은 새 컬렉션/필드를 추가하지 않았습니다 — 소급 등록도 위 `opt_cells`/`opt_runs` 스키마를 그대로 쓰고(`Run.retro:true` 플래그만 추가), 대시보드·아카이브 내보내기는 기존 필드를 읽기만 합니다.

## Phase 6 메모

- **소급 등록 마법사는 "완전 자동 그룹핑"이 아닙니다**: 설계 확정본 §8의 원안은 "하드웨어 설정 유사도"까지 자동으로 묶는 것이었지만, h5 attr을 읽지 않는다는 Phase 4의 제약이 그대로 이어져서 실제로는 "측정종류(파일명에서 파싱, 실패하면 직접 선택) + 날짜 + 4시간 이상 간격이면 새 세션"으로만 묶습니다. 그룹 안에서도 lot·시편·여기구성·검출채널은 사용자가 직접 골라야 합니다 — 그룹핑은 "같이 처리할 파일을 한 화면에 모아 보여주는" 수준의 보조이지, lot까지 추정해 주지는 않습니다.
- **등록된 Run은 항상 "보정없음"·"마운트 미지정"으로 남습니다**: 소급 등록은 사후에 마운트 세션도 보정 세션도 알 방법이 없으므로 `calibId`/`mountId`를 비운 채 `calibStatus:"none"`으로 씁니다. 이건 숨기지 않고 오히려 대시보드의 "보정없음"·"마운트 미지정" 카운터에 그대로 잡히도록 의도한 것입니다(설계 확정본 §2.4·§4가 원래 요구한 "확인은 되지만 못 쓰는 데이터임을 명시" 원칙과 일치).
- **이미 Run에 연결된 파일은 자동으로 후보에서 빠집니다**: `state.runs`의 `file.relPath`를 전부 모아 스캔 결과에서 제외한 뒤 그룹을 만들기 때문에, 마법사를 여러 번 열어도 같은 파일이 중복 등록되지 않습니다.
- **기존 셀 상태는 "planned"·"none"일 때만 "ticket"으로 올립니다**: 이미 "판정대기"·"성공"·"재측정필요"까지 진행된 셀은 소급 등록으로 Run이 추가돼도 상태를 건드리지 않습니다 — 신규 셀은 "ticket"으로 만듭니다(실측 파일은 있지만 아직 자동/사람 판정 전이라는 뜻).
- **대시보드는 설계 문서가 이미 예고했던 4개 카운터를 그대로 구현**: 판정대기 셀 수(§2.5), 재측정필요 셀 수, 보정없음 Run 수(§4), 마운트 미지정 Run 수(§2.4) + 측정종류별 계획/예약/판정대기/성공/재측정 현황표. 각 목록 행을 클릭하면 해당 셀 모달이 바로 열립니다(최대 12건까지 표시, 나머지는 "외 N건"으로만 안내).
- **아카이브 내보내기는 "대표(채택) Run" 1개씩만 복사합니다**: 셀의 `repRunId`가 가리키는 Run의 원본 파일을 `lot/측정종류/여기구성_검출채널/` 폴더 구조로 복사합니다. Firestore 첨부(T3)를 다시 압축 해제하는 대신, 이번 세션에 폴더를 스캔해서 얻은(=핸들이 살아있는) 파일을 원본 폴더에서 바로 복사하는 방식을 택했습니다 — 훨씬 간단하고 무손실입니다. 그만큼 **수동으로 개별 선택해 연결한 파일은 복사할 수 없어 건너뜁니다**(Phase 4의 재첨부 제약과 같은 이유). 완료 후 토스트로 복사/건너뜀 개수를 알려줍니다.
- **Spot 매칭은 이번 Phase에서도 착수하지 않았습니다**: `Run.stage:{x,y,z}`가 여전히 비어 있어(h5 attr을 읽어야 하는데 Phase 7 몫) 매칭 자체가 성립하지 않습니다 — 설계 확정본 §9에 이미 정리돼 있던 의존성 그대로입니다.
- **테스트**: 인메모리 fake Firestore(경로 문자열 기반, dot-path merge 지원)로 `Sync.retroRegisterRuns`를 실제로 호출해 검증했고, `buildRetroGroups`의 그룹핑 로직(같은 측정종류·4시간 이내는 한 그룹, 그 이상은 새 그룹, 측정종류 인식 실패는 별도 그룹, 이미 연결된 파일은 제외)과 `renderDashboard`의 카운트·클릭 동작, `exportArchive`의 핸들 유무에 따른 복사/건너뜀 분기까지 총 28개 시나리오로 확인했습니다(전부 통과). 해시 계산은 Phase 4와 마찬가지로 실제 `crypto.subtle`을 그대로 사용했습니다(Mock 아님). 라이트/다크 스크린샷으로 대시보드·마법사 UI 스타일도 확인했습니다.

## Phase 5 메모

- **power scan 분석 JSON만 인식합니다**: INNO Prism의 "JSON" 내보내기(`exportPowerJson`) 실제 형식을 소스 코드에서 확인해 그대로 소비합니다(`analysis.paReport`가 있는지로 형식을 판별). TCSPC/rise time 등 다른 측정종류의 Prism 내보내기는 스키마가 달라 이번 Phase에서는 지원하지 않습니다 — §3 L1 게이트 예시 자체가 power scan 기준이라 범위를 거기 맞췄습니다.
- **가져오기 방식**: 셀 모달의 각 Run 행에 "가져오기" 버튼이 있고, 클릭하면 파일 선택창이 뜹니다. Prism에서 내보낸 `*_analysis.json`을 고르면 즉시 파싱해 그 Run에 저장합니다. 연결된 h5 파일명과 JSON 안의 원본 파일명이 다르면 확인창을 띄우지만, 막지는 않습니다 — 사용자가 이미 특정 Run 행에서 눌렀으므로 최종 판단은 사용자에게 맡깁니다.
- **자동 게이트(L1)는 5항목**: 측정 범위(decade) · 반복측정 횟수 · edge-censoring 여부 · 부트스트랩 CI 폭 · 보정 세션 링크 존재. 설계 확정본 §3의 6번째 항목("다크 게이트 초과 데이터점 비율")은 Prism JSON에 바로 쓸 수 있는 완성값이 없어 제외했습니다 — 재계산하려면 Prism의 다크게이트 로직(`paDarkAndGate`)을 다시 구현해야 하는데, 그러면 두 구현이 갈라질 위험이 있다고 판단했습니다. 나머지 5항목의 구체 임계값(`POWER_SCAN_GATE`)은 초기값이며 실측 데이터로 조정이 필요합니다(설계 확정본 §9 "남은 미결"에 이미 있던 항목). 실패 1건이면 "경고", 2건 이상이면 "미달"로 굴려 올립니다.
- **사람 판정 + 사유 태그 강제**: 채택(대표 지정)/참고/폐기 세 가지 상태가 있고, 폐기하거나 자동판정과 반대로 판정할 때(자동판정 "미달"인데 채택 등)는 고정 어휘 사유 태그(9종)를 1개 이상 골라야 저장됩니다. 한 셀에서 새 Run을 채택하면 이전 대표 Run의 판정은 자동으로 "참고"로 강등됩니다(대표는 항상 최대 1개라는 불변식 유지).
- **셀 상태 전이**: Prism 결과를 처음 임포트하면(셀이 "예약" 상태일 때만) "판정대기"로 바뀝니다. Run을 채택하면 셀이 "성공"이 되고 그 Run이 `repRunId`로 기록됩니다. **현재 대표 Run**을 폐기하면 셀이 "재측정필요"로 되돌아가고 대표 지정이 풀립니다 — 대표가 아닌 Run을 폐기해도 셀 상태는 건드리지 않습니다(단순하고 예측 가능하게 유지하려는 의도적 단순화이며, 완벽한 상태 롤업은 아닙니다).
- **T2 썸네일 대신 즉석 스파크라인**: 예상대로 Prism JSON 안에 원시 데이터점(P, I)과 모델곡선(grid, modelCurve)이 이미 들어 있어서, h5를 열거나 이미지를 저장하지 않고도 로그-로그 미니 차트를 그릴 수 있었습니다. Firestore 문서 크기를 지키려고 원본 배열이 아니라 40개(원시점)·30개(곡선) 이하로 다운샘플링한 뒤 저장하고, 렌더링 시점에 인라인 SVG로 그립니다.
- **테스트**: 실제 Prism 소스(`E:\Python\INNO Prism\src\app.js`, `src\fit2.js`)를 읽어 `exportPowerJson`이 만드는 정확한 JSON 형태를 재현한 가짜 데이터로 14개 시나리오를 검증했습니다(임포트 → 자동판정 통과/미달 → 채택 시 대표 지정 및 이전 대표 강등 → 폐기 시 태그 강제 및 셀 재측정필요 전이 → 자동판정-사람판정 불일치 시 태그 강제 → 파일명 불일치 확인창). 라이트/다크 스크린샷으로 스타일도 확인했습니다.

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
