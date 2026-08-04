# 📋 골디쉼터 관리자(goldi_admin.html) 무한 로딩 이슈 기술 요약서

> 본 문서는 **골디쉼터 어드민 웹 애플리케이션(`goldi_admin.html`)**에서 발생 중인 **'데이터 로딩 중...' 무한 대기 현상**에 대해 다른 AI(Claude 등) 및 개발자가 문제 원인을 진단하고 정밀 해결책을 도출할 수 있도록 작성된 종합 기술 리포트입니다.

---

## 1. 프로젝트 및 기술 스택 개요

| 항목 | 내용 |
| :--- | :--- |
| **저장소 (Git Repo)** | `https://github.com/daveheo89/Goldie.git` (main 브랜치) |
| **어드민 라이브 URL** | `https://daveheo89.github.io/Goldie/goldi_admin.html` |
| **계약서 양식 URL** | `https://daveheo89.github.io/Goldie/goldi_contract_form.html` |
| **프론트엔드 스택** | HTML5, Vanilla CSS3, Vanilla ES6+ JavaScript (`type="module"`) |
| **백엔드 / DB** | Firebase Realtime Database (SDK v10.12.0 ESM CDN) |
| **주요 외부 라이브러리**| `html2canvas v1.4.1`, `jsPDF v2.5.1`, `Daum/Kakao 우편번호 API` |

---

## 2. 발생 현상 (Symptom Description)

1. **증상**: 어드민 페이지 접속 후 비밀번호(`5900`) 인증을 완료하거나 접속 시, 화면 중앙의 **`데이터 로딩 중...` (spinner 아이콘)** 구문에서 더 이상 진행되지 않고 무한 대기 현상이 반복됨.
2. **영향 범위**: PC 브라우저(Chrome/Edge) 및 모바일 브라우저(Safari/Chrome/iOS Webview 등) 공통 발생.
3. **특이 사항**: 비밀번호 폼 해제 후 화면 레이아웃(상단 헤더 `🐾 골디쉼터 관리자` 및 6개 탭바 `계약서 발송`, `새끼 관리`, `계약 DB`, `일정`, `방역 관리`, `치료 관리`)은 정상 노출되나, 탭 내부 데이터(강아지 리스트, 계약 데이터 등)가 렌더링되지 않거나 로딩 스피너가 사라지지 않음.

---

## 3. 핵심 시스템 아키텍처 및 데이터 흐름

### A. 비밀번호 인증 방식
- **방식**: 클라이언트 사이드 비밀번호 검증 (`5900`)
- **인증 저장**: `sessionStorage.setItem('goldi_admin_auth', 'ok')`
- **화면 전환**: `<html>` 태그에 `.admin-authenticated` 클래스 추가 
  - CSS: `.admin-authenticated #admin-login-overlay { display: none !important; }`
  - CSS: `.admin-authenticated #admin-main-content { display: block !important; }`

### B. Firebase Realtime Database 노드 구조
- **Database URL**: `https://goldie-shelter-d6c81-default-rtdb.firebaseio.com`
- **노드 목록**:
  - `/litters`: 출산 및 강아지 분양 정보 그룹 (`birthB`, `femaleCode`, `maleCode`, `puppies` 모음)
  - `/contracts`: 전자 계약서 서명 완료 데이터 (`buyer_name`, `buyer_phone`, `signature`, `contract_date`, `pdf_document`, 등)
  - `/reservations`: 방문 및 예약 정보
  - `/deliveries`: 이동 및 배송 일정
  - `/treatments`: 약품 투약 및 치료 기록

### C. 프론트엔드 데이터 로딩 3단계 메커니즘
```mermaid
graph TD
    A[페이지 로드 / 인증 성공] --> B[1단계: restoreLocalCache]
    B -->|localStorage 탐색| C{캐시 데이터 존재?}
    C -->|YES| D[0.001초 만에 로컬 데이터 1차 렌더링]
    C -->|NO| E[Firebase 백엔드 연결 진행]
    D --> E
    E --> F[2단계: REST get 1회성 HTTP 요청]
    E --> G[3단계: onValue 실시간 WebSocket 구독]
    F -->|데이터 수신 시| H[renderPuppyGroups / renderDB 실행 & puppyLoading 숨김]
    G -->|실시간 변경 시| H
```

---

## 4. 지금까지 진행된 수정 및 조치 이력 (Action History)

| 순번 | 작업 항목 | 세부 내용 및 결과 |
| :---: | :--- | :--- |
| **1** | **직인 도장 Base64 정제** | `goldi_seal.png` 이미지의 Base64 암호화 문자열 내 불필요한 줄바꿈/개행을 단일행으로 정제하고 `onerror` 이중 백업 적용. |
| **2** | **인증 로직 동기화** | `adminLogin()` 함수를 비동기 모듈에서 동기 방식(`<head>` 내 일반 `<script>`)으로 전진 배치하여 버튼 반응 속도 향상. |
| **3** | **HTML 탭바 구조 교정** | 누락되었던 `<div class="tab-bar">` 태그를 복구하여 어드민 메인 컨테이너 태그의 조기 닫힘 및 탭 글자 시인성 문제 교정. |
| **4** | **함수 호이스팅 적용** | `renderDB`, `switchTab`, `switchSubTab` 등 핵심 렌더링 함수를 화살표 함수에서 함수 선언문(`function renderDB() {}`)으로 전환하여 `ReferenceError` 차단. |
| **5** | **캐시 무효화 메타태그** | `<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">` 추가. |

---

## 5. 원인 추정 및 교차 검증 체크리스트 (Candidate Root Causes)

다른 AI(Claude) 및 검수자가 집중 검토해야 할 **4가지 핵심 추정 원인**입니다:

### 1️⃣ Firebase Realtime Database 보안 규칙 (Security Rules Read Permission)
- **추측 원인**: Firebase 콘솔의 규칙(Rules)이 `.read: false` 또는 인증 사용자만 허용(`auth != null`)하도록 설정되어 있어, 비인증 클라이언트의 `onValue` / `get` 요청이 `Permission Denied`로 차단될 가능성.
- **확인 방법**: Firebase Realtime DB 규칙이 아래와 같이 공개 읽기가 허용되어 있는지 확인:
  ```json
  {
    "rules": {
      ".read": true,
      ".write": true
    }
  }
  ```

### 2️⃣ 웹소켓(WebSocket `wss://`) 프로토콜 통신 차단 / CORS / 네트워크 프록시
- **추측 원인**: 특정 모바일 통신사(SKT/KT/LGU+), Wi-Fi, 사내망, 또는 VPN 환경에서 Firebase의 WebSocket 수신 통신(`wss://goldie-shelter-d6c81-default-rtdb.firebaseio.com/.ws`)이 핸드셰이크 단계에서 차단/타임아웃되어 `onValue` 콜백이 전혀 호출되지 않을 가능성.

### 3️⃣ 모바일 사파리/크롬 브라우저의 CDN 모듈 임포트 타임아웃
- **추측 원인**: `https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js` 및 `firebase-database.js` ESM CDN 모듈이 모바일 환경의 네트워크 불안정 또는 브라우저 정책(크로스 오리진 모듈 차단)으로 인해 다운로드 지연/실패하여 전체 자바스크립트 모듈 실행이 멈출 가능성.

### 4️⃣ 브라우저/CDN 수동 캐시 갱신 미반영
- **추측 원인**: GitHub Pages CDN 및 사용자 기기(모바일 Safari/Chrome)의 디스크 캐시에 무한 로딩 오류가 포함되어 있던 과거 구 버전 HTML/JS 파일이 여전히 수동으로 갱신되지 않고 계속 렌더링되고 있을 가능성.

---

## 6. 주요 코드 구조 발췌 (Key Source Code)

### `goldi_admin.html` 상단 헤더 및 인증 스크립트
```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
<meta http-equiv="Pragma" content="no-cache">
<meta http-equiv="Expires" content="0">
<title>골디쉼터 관리자</title>
<script>
  function checkAdminAuth() {
    if (sessionStorage.getItem('goldi_admin_auth') === 'ok') {
      document.documentElement.classList.add('admin-authenticated');
    }
  }
  checkAdminAuth();

  function adminLogin() {
    var pwInput = document.getElementById('admin-pw-input');
    var pw = pwInput ? pwInput.value.trim() : '';
    var errEl = document.getElementById('admin-pw-error');
    if (pw === '5900') {
      sessionStorage.setItem('goldi_admin_auth', 'ok');
      document.documentElement.classList.add('admin-authenticated');
      var overlay = document.getElementById('admin-login-overlay');
      var main = document.getElementById('admin-main-content');
      if (overlay) overlay.style.display = 'none';
      if (main) main.style.display = 'block';
    } else {
      if (errEl) errEl.style.display = 'block';
      var card = document.querySelector('.login-card');
      if (card) {
        card.style.animation = 'none';
        setTimeout(function() { card.style.animation = 'admin-shake .4s ease'; }, 10);
      }
      if (pwInput) {
        pwInput.value = '';
        pwInput.focus();
      }
    }
  }
  window.adminLogin = adminLogin;
</script>
```

### `goldi_admin.html` Firebase 수신 및 로딩 해제 코드
```javascript
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getDatabase, ref, onValue, update, push, remove, set, get } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-database.js";

const FB = {
  apiKey:"AIzaSyBYR0-f6YVw_pGSlfOqPNP_UyT_VcHWntk",
  authDomain:"goldie-shelter-d6c81.firebaseapp.com",
  databaseURL:"https://goldie-shelter-d6c81-default-rtdb.firebaseio.com",
  projectId:"goldie-shelter-d6c81",
  storageBucket:"goldie-shelter-d6c81.firebasestorage.app",
  messagingSenderId:"305277425020",
  appId:"1:305277425020:web:1f841f61d733c3a3aec0f9"
};
const app = initializeApp(FB);
const db  = getDatabase(app);

// ── 로컬 캐시 복원
restoreLocalCache();

// ── 1회성 HTTP REST 백업 조회
(async () => {
  try {
    const littersSnap = await get(ref(db, 'litters'));
    if (littersSnap.exists()) {
      window._litters = littersSnap.val() || {};
      saveLocalCache('litters', window._litters);
      renderPuppyGroups();
      renderManageList();
      updateStats();
      fillTreatmentSelects();
    } else {
      const pl = document.getElementById('puppyLoading');
      if (pl) pl.style.display = 'none';
    }
  } catch(e) {
    console.warn('REST get litters warning:', e);
    const pl = document.getElementById('puppyLoading');
    if (pl) pl.style.display = 'none';
  }
})();

// ── 실시간 구독
onValue(ref(db,'litters'),snap=>{
  window._litters=snap.val()||{};
  saveLocalCache('litters', window._litters);
  renderPuppyGroups();
  renderManageList();
  updateStats();
  fillTreatmentSelects();
}, error => {
  showAdminFirebaseError(error);
});
</script>
```

---

## 7. Claude 전달용 프롬프트 가이드 (Suggested Prompt for Claude)

Claude에 질문하실 때 아래 문장을 함께 전달하시면 더욱 정확한 답변을 얻으실 수 있습니다:

> "클로드, 골디쉼터 관리자 웹 앱(`goldi_admin.html`)에서 '데이터 로딩 중...' 스피너가 사라지지 않고 무한 로딩이 반복되는 문제가 있어. 첨부한 `admin_issue_summary.md` 리포트를 바탕으로 1) Firebase Realtime DB 규칙 및 권한 설정 2) 브라우저 모듈 스크립트 실행 타임아웃 3) 네트워크 및 캐시 문제 관점에서 근본적인 원인과 추가 조치 방안을 분석해줘."
