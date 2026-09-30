# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

우리 반 친구들이 이름과 한 줄 인사를 남기는 방명록 웹앱. `index.html` 파일 하나(인라인 CSS + 인라인 JS)로 완결되며, 데이터는 로컬이 아니라 Supabase 프로젝트("day 06", id `qlqusyzaezadkagvefqj`)의 `guestbook` 테이블에 저장된다. 요구사항과 데이터 모델은 [PRD.md](PRD.md)에, 개발 단계 체크리스트는 [STEPS.md](STEPS.md)에 정리되어 있다. 시각 디자인은 [DESIGN-figma (1).md](DESIGN-figma (1).md)에 정리된 Figma 마케팅 사이트 디자인 시스템(모노크롬 + 파스텔 컬러 블록, pill 버튼)을 기반으로 한다.

## 명령어

빌드/린트/테스트 도구가 없다. 동작 확인은 로컬 서버를 띄워 브라우저로 직접 여는 것뿐이다.

- Supabase가 `https://...supabase.co` 외부 API를 호출하므로 `file://`로 직접 열면 CORS/스크립트 실행 문제로 정상 동작하지 않는다. 반드시 `.claude/launch.json`에 정의된 `guestbook-static` 설정(`npx --yes http-server -p 5173`)으로 로컬 서버를 띄워 `http://localhost:5173/`로 열어야 한다.
- `http-server`는 기본적으로 정적 파일을 캐싱한다. `index.html`을 수정한 뒤 브라우저(또는 미리보기 탭)에서 새로고침해도 이전 내용이 보이면, 쿼리스트링을 붙여(`?v=2`) 캐시를 우회해서 확인한다.
- Supabase 쪽 스키마/정책을 바꿀 때는 Supabase MCP 도구(`apply_migration`, `execute_sql`)를 사용한다. 프로젝트 CLI나 로컬 Supabase 스택은 구성되어 있지 않다.

## 아키텍처

전체 로직이 `index.html` 안의 단일 `<script>` 블록에 있다. `@supabase/supabase-js` UMD 빌드를 CDN에서 불러와 `window.supabase.createClient(...)`로 클라이언트를 만든다.

- **클라이언트 변수명 주의**: Supabase UMD 스크립트가 전역 `window.supabase` 객체를 노출하므로, 클라이언트 인스턴스를 `const supabase = ...`로 선언하면 `Identifier 'supabase' has already been declared` 오류로 스크립트 전체가 죽는다. 반드시 다른 이름(`supabaseClient`)을 써야 한다.
- **저장/조회**: `submitEntry()`가 이름·글 공백 트림 후 둘 다 비어있으면 저장하지 않고 `.error-msg`에 안내를 띄운다. 유효하면 `guestbook` 테이블에 `insert({ name, message })` 후 `loadEntries()`를 다시 불러 목록을 갱신한다. `loadEntries()`는 `created_at` 내림차순(`ascending: false`)으로 조회한다.
- **XSS 방지**: 목록 항목은 `innerHTML` 조합이 아니라 `document.createElement` + `textContent`로만 구성한다. 입력값을 HTML로 해석시키는 경로(`innerHTML`, `insertAdjacentHTML` 등)를 새로 추가하지 않도록 주의.
- **디자인 토큰**: `<style>` 블록의 CSS 커스텀 프로퍼티(`--color-*`, `--rounded-*`, `--space-*`)가 `DESIGN-figma (1).md`의 `colors`/`rounded`/`spacing` 토큰에 대응한다. `figmaSans`/`figmaMono`는 라이선스가 없어 문서가 권장하는 대체 폰트(Inter, JetBrains Mono)를 Google Fonts로 불러와 쓴다. 새 UI를 추가할 때는 새 색상을 만들지 말고 이 토큰들과 `DESIGN-figma (1).md`의 `components:` 섹션(예: `color-block-section`, `button-primary`)을 재사용한다.
- **레이아웃**: `.page`가 `max-width: 430px`로 고정된 모바일 폭 레이아웃이며, 데스크톱 대응 브레이크포인트는 의도적으로 없다(PRD 2단계 요구사항이 휴대폰 폭 한정).

## Supabase 스키마 / 권한 주의사항

`guestbook` 테이블은 RLS가 켜져 있고 `anon` role에 대해 `select`/`insert`를 허용하는 정책이 걸려 있다. **RLS 정책만으로는 부족하고, `anon` role에 대한 `grant select, insert on public.guestbook`(및 identity 컬럼의 시퀀스에 대한 `grant usage, select`)이 별도로 필요하다** — 이걸 빠뜨리면 정책은 통과해도 "permission denied for table guestbook" 오류가 난다. 테이블/컬럼을 추가하거나 새 테이블을 만들 때도 이 grant를 함께 적용해야 한다.

## 작업 방식

기능을 추가하거나 바꿀 때는 먼저 PRD.md에 관련 요구사항이 있는지 확인한다. STEPS.md는 1단계(글 남기기·목록)와 2단계(모바일 폭 디자인)까지 완료 상태다 — 두 문서 모두 현재 구현과 어긋남 없이 마무리되어 있으므로, 새 단계를 추가하기 전에 두 문서를 함께 갱신한다.
