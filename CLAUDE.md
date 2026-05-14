# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

SISTECH 사내 AI 활용 준비도 설문 PWA. 모바일 우선 한국어 UI. 빌드 파이프라인·테스트·외부 의존성 없는 단일 파일(`index.html`) 바닐라 JS 앱. 응답은 `localStorage`에 저장되고, 선택적으로 Google Apps Script Web App을 통해 Google Sheets로 미러링됨.

## 개발·실행

빌드 단계가 없습니다. 어떤 정적 파일 서버에서든 바로 실행됩니다.

```bash
# 루트에서 정적 서버 띄우기 (Service Worker 동작 필요 → file:// 금지)
python3 -m http.server 8000
# 또는
npx serve .
```

- `sw.js`는 `/`, `/index.html`, `/manifest.json`을 캐시합니다. 자산을 수정한 뒤에는 **반드시 `sw.js`의 `CACHE` 버전 문자열(`sistech-survey-v1`)을 올려야** 사용자가 새 빌드를 받습니다. 안 그러면 PWA가 구버전을 계속 서빙합니다.
- Service Worker는 `script.google.com`으로 가는 요청은 의도적으로 캐싱하지 않습니다 (Sheets 제출이 항상 네트워크를 타도록).

## 아키텍처

### 단일 파일 레이아웃 (`index.html`)

모든 마크업·CSS·JS가 한 파일에 있습니다. 스크립트 블록(`index.html:287` 이하)은 다음 순서로 구성됩니다.

1. **설정 상수** (`index.html:288-298`) — `ADMIN_PW`, `localStorage` 키(`STORE`, `CFG`, `AUTH`), 저장/로드 헬퍼.
2. **Apps Script 템플릿** (`index.html:300-324`) — `APPS` 문자열은 관리자가 Google Apps Script 편집기에 붙여넣을 코드입니다. 시트 컬럼 순서가 여기 박혀 있어서 설문 키를 추가/변경하면 이 문자열의 `h[]` 헤더 배열과 `sh.appendRow([...])` 인자 순서도 **함께** 수정해야 합니다.
3. **도메인 상수** (`index.html:328-336`) — `DEPTS`, `JOBS`, `FREE_TOOLS`, `PAID_TOOLS`, `PAY`, `USE_WAYS`, `HELP`, `BARRIERS`, `SUPPORTS`. 선택지를 손볼 때는 여기.
4. **섹션 메타** `SEC` (`index.html:337-343`) — 5개 섹션의 라벨·소제목·진행률(%) 정의. 섹션을 늘리려면 `SEC`, `renderStep`의 `[buildS0..buildS4]` 빌더 배열, 그리고 `goNext`의 `step<4` 비교값을 같이 수정해야 합니다.
5. **상태** `st` — 모든 응답을 담는 단일 객체. 초기 모양은 `initSt()`(`index.html:347`)에 정의. **q-키 추가 시 `initSt`, 해당 빌더 함수, 검토 화면(`renderReview`), 관리자 대시보드(`renderAdmin`), Apps Script 헤더 5곳을 모두 동기화**해야 합니다.
6. **화면 라우팅** — `showOnly(id)`로 `pg-survey` / `pg-review` / `pg-done` / `pg-login` / `pg-admin` 다섯 페이지 중 하나만 보입니다. 라우터·프레임워크 없음.
7. **빌더 헬퍼** `mkCard`, `addRadio`, `addCheck`, `addScale` (`index.html:435-438`) — 모든 질문 카드는 이걸 통해 그립니다. UI 일관성을 위해 새 질문도 같은 헬퍼를 통해 추가하세요.
8. **검증** — `isValid()`(`index.html:406`)이 현재 `step`에 대한 필수 응답만 확인하고, `chkBtn()`이 매 상호작용마다 호출되어 "다음" 버튼 활성화를 토글합니다.

### 데이터 흐름

- **로컬**: 최종 제출(`submitFinal`)이 `record = {id, ts, ...st}` 를 만들어 `localStorage[STORE]`(`'st_v5'`) 배열에 push합니다. 관리자 대시보드는 오직 이 로컬 배열에서만 통계를 계산합니다 — Sheets에서 다시 읽지 않습니다.
- **원격 (옵션)**: `sendToSheets`가 `mode:'no-cors'` 로 Apps Script URL에 POST합니다. URL은 관리자 모달(⚙ Sheets 버튼)에서 입력되어 `localStorage[CFG]`(`'st_cfg_v2'`)에 저장됩니다. `no-cors`이므로 응답을 읽을 수 없고, 실패해도 조용히 삼킵니다(`catch(e){}`). 시트로 안 들어가는 문제가 보고되면 Apps Script 로그를 확인하세요.
- **관리자 인증**: `sessionStorage[AUTH]`. 비밀번호는 클라이언트 상수 `ADMIN_PW`(`index.html:289`, 기본 `'sistech2025'`)로 평문 비교 — 신뢰 경계가 약하다는 점을 의식하고, 진짜 비밀이 있는 곳에 끼워 넣지 마세요.

### PWA

- `manifest.json`은 PWA 메타데이터(이름 "SISTECH AI 설문", 색상, 아이콘 8종).
- `index.html:582`에서 `sw.js`를 등록합니다.
- `beforeinstallprompt`를 가로채 상단 그린 배너(`#install-bar`)로 직접 설치 버튼을 노출합니다.

## 변경 시 흔히 놓치는 곳

- 질문 키를 추가했을 때 **Apps Script 시트 컬럼 헤더**(`APPS` 안의 `h` 배열) 갱신 누락 — 새 응답은 마지막 컬럼에 빈칸으로 들어가고 옛 시트는 깨집니다.
- **`sw.js`의 `CACHE` 버전**을 안 올린 채 배포 — 기존 사용자는 캐시된 구버전 `index.html`을 계속 받습니다.
- `q1_jobs`처럼 배열 형태로 저장되는 키는 관리자 대시보드(`renderAdmin`)와 검토 화면(`renderReview`)이 각각 다르게 직렬화합니다. 새 복수선택 키를 추가하면 둘 다 확인하세요.
- `st_v5`, `st_cfg_v2` 같은 `localStorage` 키 이름에 버전 접미사가 있는 이유: 응답 스키마가 호환되지 않게 바뀌면 키를 올려서 옛 데이터를 자연스럽게 무시합니다.
