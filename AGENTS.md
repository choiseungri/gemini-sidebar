# Project: Gemini Sidebar Helper Extension
## Agent Task Specification (AGENTS.md)

이 문서는 'Gemini 사이드바 도우미' 크롬 확장 프로그램을 개발하기 위해 각 전문 AI 에이전트에게 전달할 작업 명세서를 정의합니다. 각 에이전트는 담당 파일의 요구사항을 완벽하게 준수하여 코드를 생성해야 합니다.

## Agent 1: 확장 프로그램 설계자 (Architect)

**담당 파일:** `manifest.json`

### 지시사항:
Manifest V3 표준에 맞춰 크롬 확장 프로그램의 기본 설정을 정의하는 manifest.json 파일을 생성하라. 아래의 모든 명세를 포함해야 한다.

- **"name"**: "Gemini 사이드바 도우미"
- **"version"**: "1.0.0"
- **"description"**: "현재 페이지의 컨텍스트를 활용하여 Gemini 사이드바를 더 스마트하게 사용합니다."
- **"permissions"**: "sidePanel", "tabs", "scripting", "offscreen", "downloads" 권한을 포함해야 한다.
- **"host_permissions"**: 모든 웹사이트의 컨텐츠에 접근할 수 있도록 `"<all_urls>"`를 포함해야 한다.
- **"background"**: 서비스 워커로 background.js를 지정한다.
- **"action"**: 툴바 아이콘 클릭 시 사이드 패널을 열도록 설정하고, 기본 아이콘 경로를 icons/ 폴더로 지정한다.
- **"side_panel"**: 기본 사이드 패널 HTML 파일로 sidebar.html을 지정한다.
- **"content_scripts"**: gemini.google.com의 모든 프레임(all_frames: true)에서 document_idle 시점에 gemini_injector.js가 실행되도록 설정한다.

## Agent 2: 백엔드 로직 전문가 (Core Logic Specialist)

**담당 파일:** `background.js`

### 지시사항:
확장 프로그램의 모든 핵심 비동기 로직을 처리하는 서비스 워커 background.js 파일을 생성하라. 다음 요구사항을 async/await를 사용하여 순서대로 정확하게 구현해야 한다.

#### 리스너 설정:
- **chrome.action.onClicked**: 아이콘 클릭 시 사이드 패널을 연다.
- **chrome.runtime.onMessage**: 외부로부터 `{type: 'START_PDF_GENERATION'}` 메시지를 받으면 아래의 handlePdfGeneration 함수를 호출한다. 비동기 응답을 위해 `return true;`를 사용한다.

#### handlePdfGeneration 비동기 함수 구현:
1. **프로세스 시작**: 사이드바에 로딩 상태를 알리는 `{type: 'PDF_STATUS_UPDATE', status: 'processing', message: '...'}` 메시지를 전송한다.
2. **HTML 추출**: 현재 활성 탭의 전체 HTML(`document.documentElement.outerHTML`)을 `chrome.scripting.executeScript`로 가져온다.
3. **Offscreen 문서 준비**: `chrome.offscreen.createDocument`로 PDF 변환용 offscreen.html을 준비한다. 중복 생성을 방지하는 로직은 필수다.
4. **PDF 변환**: Offscreen 문서로 추출한 HTML을 보내 html2pdf.js를 통해 PDF Blob으로 변환하고 결과를 await로 받는다.
5. **로컬 저장**: 변환된 Blob을 `gemini-page-[timestamp].pdf` 형식의 파일명으로 `chrome.downloads.download`를 통해 저장한다.
6. **업로드 유도**: 저장이 시작되면, gemini_injector.js에 `{type: 'TRIGGER_FILE_UPLOAD'}` 메시지를 보내 파일 첨부 버튼 클릭을 유도한다.
7. **프로세스 성공**: 모든 과정이 완료되면 사이드바에 성공 메시지를 전송한다.
8. **예외 처리**: 전체 로직을 try...catch...finally로 감싸고, 실패 시 사용자 친화적인 에러 메시지를 사이드바로 전송하며, finally 블록에서 `chrome.offscreen.closeDocument()`를 호출하여 자원을 반드시 해제한다.

## Agent 3: UI/UX 설계자 (Frontend Architect)

**담당 파일:** `sidebar.html`, `sidebar.css`

### 지시사항:
사이드 패널의 UI를 구성하는 sidebar.html과 스타일을 정의하는 sidebar.css 파일을 생성하라.

#### sidebar.html:
- 최상단에 확장 프로그램의 고유 UI를 담을 `<header>`를 배치한다. 헤더에는 ID가 `analyze-button`인 버튼과, 로딩 상태를 표시할 ID `status-container`인 div를 포함한다.
- 헤더 아래에는 `gemini.google.com/app`을 로드하는 `<iframe>`을 배치하고, ID는 `gemini-frame`으로 설정한다.

#### sidebar.css:
- 전체적인 레이아웃을 Flexbox로 구성하여 헤더와 iframe이 수직으로 배치되도록 한다.
- `#analyze-button`에 현대적인 버튼 스타일(패딩, 둥근 모서리, 호버/액티브 효과)을 적용한다.
- `#status-container` 내부의 스피너와 메시지가 보기 좋게 정렬되도록 스타일을 지정한다.
- 상태 메시지에 error, success 클래스에 따라 다른 색상(빨간색, 녹색)이 적용되도록 스타일을 정의한다.
- 스피너가 회전하는 @keyframes 애니메이션을 추가한다.

## Agent 4: 프론트엔드 로직 전문가 (Frontend Logic Specialist)

**담당 파일:** `sidebar.js`

### 지시사항:
sidebar.html의 사용자 인터랙션을 처리하는 sidebar.js 파일을 생성하라.

- **이벤트 리스너**: `analyze-button` 클릭 시 background.js로 `{type: 'START_PDF_GENERATION'}` 메시지를 전송하는 로직을 구현한다.
- **상태 업데이트**: background.js로부터 `{type: 'PDF_STATUS_UPDATE'}` 메시지를 수신하는 `chrome.runtime.onMessage` 리스너를 구현한다.
- 수신한 메시지의 status (processing, error, success)에 따라 버튼을 비활성화/활성화하고, 스피너를 표시/숨김 처리하며, status-message의 텍스트와 CSS 클래스를 동적으로 변경하는 `updateStatus` 함수를 작성한다.

## Agent 5: 웹 페이지 제어 전문가 (Content Injection Specialist)

**담당 파일:** `gemini_injector.js`

### 지시사항:
gemini.google.com 웹페이지 내부에서 실행되어 페이지 DOM을 제어하는 gemini_injector.js 파일을 생성하라.

- background.js로부터 `{type: 'TRIGGER_FILE_UPLOAD'}` 메시지를 수신하는 `chrome.runtime.onMessage` 리스너를 구현한다.
- 메시지를 받으면 Gemini 페이지의 파일 첨부 버튼 (`<button>` 요소)을 `document.querySelector`로 찾아서 프로그래밍적으로 `.click()` 이벤트를 발생시킨다.
- **주의**: Gemini 웹사이트의 구조는 변경될 수 있으므로, 최대한 유연하고 깨지기 어려운 선택자(예: aria-label 활용)를 사용해야 한다고 주석으로 명시한다.

## Agent 6: 백그라운드 작업자 (Offscreen Task Handler)

**담당 파일:** `offscreen.html`, `offscreen.js`

### 지시사항:
백그라운드에서 PDF 변환 작업을 안전하게 수행할 offscreen.html과 offscreen.js 파일을 생성하라.

#### offscreen.html:
- 최소한의 HTML 구조를 가지며, PDF 변환 라이브러리인 `html2pdf.bundle.min.js`와 로직 스크립트인 `offscreen.js`를 `<script>` 태그로 포함한다.

#### offscreen.js:
- background.js로부터 `{type: 'CONVERT_TO_PDF', target: 'offscreen', html: ...}` 메시지를 수신하는 리스너를 구현한다.
- 전달받은 HTML 문자열을 `html2pdf()` 라이브러리를 사용하여 PDF Blob 객체로 변환한다.
- 변환이 완료되면 Promise의 then을 통해 결과 Blob을 `sendResponse`로 background.js에 반환한다.
- 비동기 응답을 위해 리스너 마지막에 `return true;`를 반드시 포함한다.