# 프로젝트 개요
Chrome 확장 프로그램.
웹 페이지에서 텍스트를 선택한 후 우클릭하면 사전 검색 결과를 오버레이로 표시한다.
검색한 단어는 개인 단어장에 저장할 수 있으며, 단어장은 여러 개 생성할 수 있다.
저장된 단어장은 외부 플래시카드 앱에 업로드할 수 있도록 탭 구분 텍스트로 추출할 수 있다.
출력 형식:
```text
단어\t의미
단어\t의미
```


## 기술 스택
* TypeScript
* Vite
* @crxjs/vite-plugin
* Chrome Extension Manifest V3
* 별도 백엔드 없음
* 로컬 데이터는 `chrome.storage.local` 사용
* `unlimitedStorage` 권한 사용
* 테스트: Vitest + jest-chrome


## 명령어
* 개발 서버(HMR): `npm run dev`
* 빌드: `npm run build`
* 빌드 결과: `dist/`
* 테스트: `npm run test`
* 타입 체크: `npm run typecheck`
빌드 결과는 `chrome://extensions`에서 unpacked extension으로 로드한다.


## 아키텍처
### `src/background.ts`
Manifest V3 service worker.
담당 역할:
* 우클릭 context menu 등록
* 선택한 텍스트를 전달받아 사전 API 호출
* 검색 결과를 content script로 전달
* extension lifecycle 및 메시지 처리

### `src/content.ts`
웹 페이지에서 실행되는 content script.
담당 역할:
* 검색 결과 오버레이 DOM 생성
* 오버레이 표시 및 제거
* background service worker와 메시지 통신

### `src/popup/`
Chrome toolbar popup.
담당 역할:
* 현재 사용할 단어장 선택
* 최근 저장한 단어 확인

### `src/viewer/`
별도 Chrome extension 탭에서 열리는 단어장 관리 페이지.
담당 역할:
* 단어장 목록 표시
* 단어장 내용 표시
* 단어장 생성/삭제
* 저장된 단어 관리
* 탭 구분 텍스트 생성
* 텍스트 복사

### `src/storage.ts`
`chrome.storage.local` 접근을 추상화하는 모듈.
단어장과 단어의 CRUD 로직은 이 모듈에 모은다.
다른 모듈에서는 `chrome.storage.*` API를 직접 호출하지 않는다.
스토리지 접근은 반드시 이 모듈을 통해 수행한다.
`chrome.runtime`, `chrome.contextMenus` 등 storage와 관계없는 Chrome Extension API에는 이 제한을 적용하지 않는다.


## 코드 스타일
* ES Modules(`import` / `export`)을 사용한다.
* CommonJS `require`는 사용하지 않는다.
* 비동기 처리는 `async` / `await`를 우선한다.
* 불필요한 `.then()` 체이닝은 지양한다.
* 기존 프로젝트 구조와 코딩 스타일을 우선해서 따른다.
* 필요하지 않은 새로운 dependency를 추가하지 않는다.


## 데이터 구조
`chrome.storage.local`에서 다음 구조를 사용한다.

```ts
interface Word {
  word: string;
  meaning: string;
}

interface Wordbook {
  name: string;
  words: Word[];
}

interface StorageSchema {
  wordbooks: Record<string, Wordbook>;
}
```

저장 형태 예시:

```json
{
  "wordbooks": {
    "<id>": {
      "name": "영어 단어",
      "words": [
        {
          "word": "example",
          "meaning": "예시"
        }
      ]
    }
  }
}
```


## 사전 API
현재 사용할 사전 API provider는 구현 전에 확인한다.
과거의 네이버 사전 Open API가 현재도 사용 가능하다고 가정하지 않는다.
API를 선택하거나 변경할 때 다음 사항을 먼저 확인한다.

* API가 현재 운영 중인지
* Chrome Extension에서 호출 가능한지
* CORS 및 `host_permissions` 요구사항
* 인증 방식
* API key 또는 secret의 client-side 노출 가능성
* 이용약관상 Chrome Extension에서의 사용 가능 여부
* 호출 제한

API provider 또는 인증 구조를 변경해야 한다면 임의로 결정하지 말고 먼저 사용자에게 알린다.


## 보안
* API key, token, client secret을 소스 코드에 하드코딩하지 않는다.
* secret이 포함된 파일은 Git에 커밋하지 않는다.
* `.env`, `.env.local` 등 환경변수 파일을 사용할 경우 `.gitignore` 상태를 확인한다.
* Vite client bundle에 포함되는 환경변수는 secret을 안전하게 숨기는 방법이 아님을 전제로 한다.
* client-side에서 안전하게 보관할 수 없는 secret을 요구하는 API라면 구현 전에 사용자에게 알린다.


## Chrome Extension 권한
`manifest.json`의 다음 항목을 추가하거나 확대해야 하는 경우 변경 전에 사용자에게 알린다.

* `permissions`
* `host_permissions`
* `optional_permissions`
* `optional_host_permissions`

특히 새로운 host 접근 권한을 임의로 추가하지 않는다.
사전 API를 service worker에서 `fetch()`하기 위해 host permission이 필요한 경우에도 먼저 사용자에게 알린다.


## 작업 원칙
코드를 수정한 후 가능한 경우 다음을 실행한다.
```bash
npm run typecheck
npm run test
npm run build
```
오류가 발생하면 오류 내용을 확인하고 원인을 해결한다.
요청받지 않은 대규모 리팩터링은 하지 않는다.
현재 작업과 관계없는 파일을 수정하지 않는다.