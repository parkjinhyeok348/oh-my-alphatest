# Lean Harness 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** superpowers의 훈련된 프로세스와 GSD의 신선한 컨텍스트 방식을 결합한 경량 Claude Code 하네스 + 브라우저 대시보드를 처음부터 구축한다.

**Architecture:** 4개 슬래시 명령어(`/brainstorm`, `/plan`, `/exec`, `/verify`)가 각각 신선한 서브에이전트로 실행되고, 단계마다 `.artifacts/` 디렉토리에 마크다운 + `index.json`을 git 커밋한다. 정적 HTML 대시보드가 `index.json`을 읽어 PM/QA/기획자에게 진행 상황과 이력을 보여준다.

**Tech Stack:** Node.js (artifact 업데이트 스크립트, 보안 hook), Vanilla HTML/CSS/JS (대시보드), Bash (shell hooks), Markdown (Claude Code 슬래시 명령어)

**Design Spec:** `docs/superpowers/specs/2026-05-20-lightweight-harness-dashboard-design.md`

---

## 파일 구조 (전체)

```
<new-project-root>/           ← 새 독립 프로젝트 디렉토리 (이름은 자유)
├── CLAUDE.md                 ← 하네스 핵심 설정
├── .claude/
│   └── settings.json         ← hook 배선
├── .artifacts/
│   ├── index.json            ← 대시보드 SSOT
│   └── *.md                  ← 단계별 아티팩트 파일
├── commands/
│   ├── brainstorm.md         ← /brainstorm 슬래시 명령어
│   ├── plan.md               ← /plan
│   ├── exec.md               ← /exec
│   └── verify.md             ← /verify
├── hooks/
│   └── security-check.js     ← PreToolUse 보안 게이트 (Node.js)
├── scripts/
│   └── update-artifact.js    ← index.json 업데이터
└── dashboard/
    ├── index.html
    ├── style.css
    └── app.js
```

---

## Task 1: 프로젝트 스캐폴딩 + 초기 index.json

**Files:**
- Create: `<project-root>/` (새 git 저장소)
- Create: `.artifacts/index.json`
- Create: `.gitignore`

- [ ] **Step 1: 새 프로젝트 디렉토리 생성 및 git 초기화**

```bash
mkdir lean-harness && cd lean-harness
git init
```

- [ ] **Step 2: `.gitignore` 작성**

```
node_modules/
.DS_Store
*.log
```

- [ ] **Step 3: `.artifacts/` 디렉토리 + 초기 `index.json` 작성**

파일 경로: `.artifacts/index.json`

```json
{
  "project": "프로젝트명",
  "stages": [
    { "stage": "brainstorm", "label": "요구사항 분석", "status": "pending", "timestamp": null, "commit": null, "artifact": null, "summary": null },
    { "stage": "plan",        "label": "계획",          "status": "pending", "timestamp": null, "commit": null, "artifact": null, "summary": null },
    { "stage": "exec",        "label": "구현",          "status": "pending", "timestamp": null, "commit": null, "artifact": null, "summary": null },
    { "stage": "verify",      "label": "검증",          "status": "pending", "timestamp": null, "commit": null, "artifact": null, "summary": null }
  ],
  "currentStage": "brainstorm",
  "securityGates": []
}
```

- [ ] **Step 4: JSON 유효성 검증**

```bash
node -e "JSON.parse(require('fs').readFileSync('.artifacts/index.json', 'utf8')); console.log('valid')"
```

기대 출력: `valid`

- [ ] **Step 5: 커밋**

```bash
git add .artifacts/index.json .gitignore
git commit -m "chore: scaffold project + initial artifact index"
```

---

## Task 2: `scripts/update-artifact.js` — index.json 업데이터

**Files:**
- Create: `scripts/update-artifact.js`
- Create: `scripts/update-artifact.test.js`

이 스크립트는 각 슬래시 명령어가 단계를 완료할 때 호출한다.

```bash
node scripts/update-artifact.js \
  --stage brainstorm \
  --status approved \
  --summary "OAuth2 기반 로그인 기능" \
  --artifact "brainstorm-20260520T100000.md"
```

- [ ] **Step 1: 테스트 파일 작성**

파일 경로: `scripts/update-artifact.test.js`

```js
const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

const ARTIFACTS_DIR = path.join(__dirname, '..', '.artifacts');
const INDEX_PATH = path.join(ARTIFACTS_DIR, 'index.json');

function resetIndex() {
  const initial = {
    project: "test-project",
    stages: [
      { stage: "brainstorm", label: "요구사항 분석", status: "pending", timestamp: null, commit: null, artifact: null, summary: null },
      { stage: "plan",        label: "계획",          status: "pending", timestamp: null, commit: null, artifact: null, summary: null },
      { stage: "exec",        label: "구현",          status: "pending", timestamp: null, commit: null, artifact: null, summary: null },
      { stage: "verify",      label: "검증",          status: "pending", timestamp: null, commit: null, artifact: null, summary: null }
    ],
    currentStage: "brainstorm",
    securityGates: []
  };
  fs.writeFileSync(INDEX_PATH, JSON.stringify(initial, null, 2));
}

// Test 1: stage status 업데이트
resetIndex();
execSync(`node scripts/update-artifact.js --stage brainstorm --status approved --summary "테스트 요약" --artifact "brainstorm-test.md"`, { stdio: 'inherit' });
const result = JSON.parse(fs.readFileSync(INDEX_PATH, 'utf8'));
const bs = result.stages.find(s => s.stage === 'brainstorm');
console.assert(bs.status === 'approved', `status should be 'approved', got '${bs.status}'`);
console.assert(bs.summary === '테스트 요약', `summary mismatch`);
console.assert(bs.artifact === 'brainstorm-test.md', `artifact mismatch`);
console.assert(bs.timestamp !== null, `timestamp should be set`);
console.assert(result.currentStage === 'plan', `currentStage should advance to 'plan'`);

// Test 2: securityGates 추가
execSync(`node scripts/update-artifact.js --security-gate --file "src/auth/login.ts" --reason "auth 패턴 감지"`, { stdio: 'inherit' });
const result2 = JSON.parse(fs.readFileSync(INDEX_PATH, 'utf8'));
console.assert(result2.securityGates.length === 1, `securityGates should have 1 entry`);
console.assert(result2.securityGates[0].file === 'src/auth/login.ts', `gate file mismatch`);

console.log('All tests passed.');
```

- [ ] **Step 2: 테스트 실행 — 실패 확인**

```bash
node scripts/update-artifact.test.js
```

기대 출력: `Error: Cannot find module 'scripts/update-artifact.js'` (또는 유사한 오류)

- [ ] **Step 3: `scripts/update-artifact.js` 구현**

파일 경로: `scripts/update-artifact.js`

```js
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

const INDEX_PATH = path.join(process.cwd(), '.artifacts', 'index.json');
const STAGE_ORDER = ['brainstorm', 'plan', 'exec', 'verify'];

function parseArgs(argv) {
  const args = {};
  for (let i = 2; i < argv.length; i++) {
    const key = argv[i].replace(/^--/, '');
    const val = argv[i + 1] && !argv[i + 1].startsWith('--') ? argv[++i] : true;
    args[key] = val;
  }
  return args;
}

function readIndex() {
  return JSON.parse(fs.readFileSync(INDEX_PATH, 'utf8'));
}

function writeIndex(data) {
  fs.writeFileSync(INDEX_PATH, JSON.stringify(data, null, 2));
}

function getCurrentCommit() {
  try {
    return execSync('git rev-parse --short HEAD', { stdio: ['pipe', 'pipe', 'ignore'] }).toString().trim();
  } catch {
    return null;
  }
}

const args = parseArgs(process.argv);

if (args['security-gate']) {
  const data = readIndex();
  data.securityGates.push({
    file: args.file,
    reason: args.reason,
    approvedAt: new Date().toISOString()
  });
  writeIndex(data);
  console.log(`Security gate recorded: ${args.file}`);
  process.exit(0);
}

const { stage, status, summary, artifact } = args;
if (!stage || !status) {
  console.error('Usage: update-artifact.js --stage <name> --status <pending|approved|rejected> [--summary <text>] [--artifact <file>]');
  process.exit(1);
}

const data = readIndex();
const entry = data.stages.find(s => s.stage === stage);
if (!entry) {
  console.error(`Unknown stage: ${stage}. Valid stages: ${STAGE_ORDER.join(', ')}`);
  process.exit(1);
}

entry.status = status;
entry.timestamp = new Date().toISOString();
entry.commit = getCurrentCommit();
if (summary) entry.summary = summary;
if (artifact) entry.artifact = artifact;

// currentStage를 다음 단계로 진행
if (status === 'approved') {
  const idx = STAGE_ORDER.indexOf(stage);
  if (idx < STAGE_ORDER.length - 1) {
    data.currentStage = STAGE_ORDER[idx + 1];
  } else {
    data.currentStage = 'complete';
  }
}

writeIndex(data);
console.log(`Stage '${stage}' updated to '${status}'.`);
```

- [ ] **Step 4: 테스트 실행 — 통과 확인**

```bash
node scripts/update-artifact.test.js
```

기대 출력: `All tests passed.`

- [ ] **Step 5: 커밋**

```bash
git add scripts/
git commit -m "feat(scripts): add update-artifact.js with stage + security gate support"
```

---

## Task 3: `dashboard/index.html` + `dashboard/style.css`

**Files:**
- Create: `dashboard/index.html`
- Create: `dashboard/style.css`

- [ ] **Step 1: `dashboard/index.html` 작성**

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Lean Harness Dashboard</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <header>
    <h1 id="project-name">로딩 중...</h1>
    <div id="current-stage-label"></div>
  </header>

  <section id="stage-tracker">
    <!-- JS가 렌더링 -->
  </section>

  <section id="history">
    <h2>단계별 히스토리</h2>
    <div id="history-list">
      <!-- JS가 렌더링 -->
    </div>
  </section>

  <section id="security-section" hidden>
    <h2>🔒 보안 승인 이력</h2>
    <ul id="security-list"></ul>
  </section>

  <!-- 마크다운 팝업 -->
  <div id="modal" class="modal" hidden>
    <div class="modal-content">
      <button id="modal-close">✕</button>
      <pre id="modal-body"></pre>
    </div>
  </div>

  <script src="app.js"></script>
</body>
</html>
```

- [ ] **Step 2: `dashboard/style.css` 작성**

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #f5f5f5;
  color: #333;
  padding: 24px;
}

header {
  background: #fff;
  border-radius: 8px;
  padding: 20px 24px;
  margin-bottom: 16px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.08);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

h1 { font-size: 1.4rem; font-weight: 700; }
h2 { font-size: 1rem; font-weight: 600; margin-bottom: 12px; color: #555; }

#current-stage-label {
  font-size: 0.9rem;
  color: #666;
  background: #f0f0f0;
  padding: 4px 12px;
  border-radius: 20px;
}

/* Stage tracker */
#stage-tracker {
  background: #fff;
  border-radius: 8px;
  padding: 20px 24px;
  margin-bottom: 16px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.08);
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
}

.stage-chip {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 8px 16px;
  border-radius: 8px;
  background: #f9f9f9;
  border: 1px solid #e0e0e0;
  min-width: 110px;
}

.stage-chip.approved  { background: #e8f5e9; border-color: #a5d6a7; }
.stage-chip.in-progress { background: #fff8e1; border-color: #ffe082; }
.stage-chip.pending   { background: #f5f5f5; border-color: #e0e0e0; color: #999; }
.stage-chip.rejected  { background: #ffebee; border-color: #ef9a9a; }

.stage-chip .icon { font-size: 1.4rem; }
.stage-chip .name { font-size: 0.8rem; font-weight: 600; }
.stage-chip .badge { font-size: 0.7rem; color: #777; }

/* History */
#history {
  background: #fff;
  border-radius: 8px;
  padding: 20px 24px;
  margin-bottom: 16px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.08);
}

.history-item {
  border: 1px solid #e8e8e8;
  border-radius: 6px;
  padding: 12px 16px;
  margin-bottom: 8px;
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center;
  gap: 12px;
}

.history-item.pending { opacity: 0.5; }

.history-meta { font-size: 0.75rem; color: #999; }
.history-summary { font-size: 0.875rem; color: #444; }
.history-commit { font-family: monospace; font-size: 0.7rem; color: #aaa; }

.detail-btn {
  font-size: 0.8rem;
  padding: 4px 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: #fff;
  cursor: pointer;
}
.detail-btn:hover { background: #f0f0f0; }

/* Security */
#security-section {
  background: #fff;
  border-radius: 8px;
  padding: 20px 24px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.08);
}

#security-list { list-style: none; }
#security-list li { font-size: 0.85rem; padding: 6px 0; border-bottom: 1px solid #f0f0f0; color: #555; }

/* Modal */
.modal {
  position: fixed; inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}
.modal[hidden] { display: none; }

.modal-content {
  background: #fff;
  border-radius: 8px;
  padding: 24px;
  max-width: 700px;
  width: 90%;
  max-height: 80vh;
  overflow-y: auto;
  position: relative;
}

#modal-close {
  position: absolute; top: 12px; right: 12px;
  background: none; border: none; font-size: 1.2rem; cursor: pointer;
}

#modal-body {
  white-space: pre-wrap;
  font-family: monospace;
  font-size: 0.85rem;
  line-height: 1.6;
  color: #333;
}
```

- [ ] **Step 3: `npx serve .` 실행 후 `dashboard/index.html` 브라우저에서 열어 구조 확인**

```bash
npx serve .
# 브라우저에서 http://localhost:3000/dashboard/ 열기
```

기대 결과: 빈 대시보드 렌더링 (JS 아직 없어서 "로딩 중..." 표시됨)

- [ ] **Step 4: 커밋**

```bash
git add dashboard/
git commit -m "feat(dashboard): add HTML skeleton + CSS"
```

---

## Task 4: `dashboard/app.js` — 데이터 로딩 + 렌더링

**Files:**
- Modify: `dashboard/app.js` (Create)

- [ ] **Step 1: `dashboard/app.js` 작성**

```js
const STATUS_ICON = {
  approved: '✅',
  'in-progress': '🔄',
  pending: '⏳',
  rejected: '❌'
};

const STATUS_LABEL = {
  approved: '승인됨',
  'in-progress': '진행중',
  pending: '대기',
  rejected: '거부됨'
};

async function loadIndex() {
  const res = await fetch('../.artifacts/index.json');
  if (!res.ok) throw new Error(`index.json 로드 실패: ${res.status}`);
  return res.json();
}

function renderHeader(data) {
  document.getElementById('project-name').textContent = data.project;
  const label = data.currentStage === 'complete'
    ? '모든 단계 완료'
    : `현재 단계: ${stageLabel(data.currentStage, data.stages)}`;
  document.getElementById('current-stage-label').textContent = label;
}

function stageLabel(stageName, stages) {
  const s = stages.find(x => x.stage === stageName);
  return s ? s.label : stageName;
}

function renderTracker(stages) {
  const container = document.getElementById('stage-tracker');
  container.innerHTML = stages.map(s => {
    const icon = STATUS_ICON[s.status] || '⏳';
    const badge = STATUS_LABEL[s.status] || '대기';
    return `
      <div class="stage-chip ${s.status}">
        <span class="icon">${icon}</span>
        <span class="name">${s.label}</span>
        <span class="badge">[${badge}]</span>
      </div>`;
  }).join('');
}

function renderHistory(stages) {
  const list = document.getElementById('history-list');
  list.innerHTML = stages.map(s => {
    const icon = STATUS_ICON[s.status] || '⏳';
    const time = s.timestamp ? new Date(s.timestamp).toLocaleString('ko-KR') : '';
    const commit = s.commit ? s.commit : '';
    const summary = s.summary || (s.status === 'pending' ? '아직 시작 전' : '');
    const detailBtn = s.artifact
      ? `<button class="detail-btn" onclick="showDetail('${s.artifact}')">상세 보기 ▼</button>`
      : '';

    return `
      <div class="history-item ${s.status}">
        <div>
          <span class="stage-label">${icon} ${s.label}</span>
          <div class="history-meta">${time} ${commit ? `<span class="history-commit">${commit}</span>` : ''}</div>
        </div>
        <div class="history-summary">${summary}</div>
        ${detailBtn}
      </div>`;
  }).join('');
}

function renderSecurity(gates) {
  if (!gates || gates.length === 0) return;
  const section = document.getElementById('security-section');
  section.hidden = false;
  const list = document.getElementById('security-list');
  list.innerHTML = gates.map(g => {
    const time = new Date(g.approvedAt).toLocaleString('ko-KR');
    return `<li>📄 ${g.file} — ${g.reason} — 승인됨 ${time}</li>`;
  }).join('');
}

async function showDetail(artifactFile) {
  const modal = document.getElementById('modal');
  const body = document.getElementById('modal-body');
  body.textContent = '로딩 중...';
  modal.hidden = false;

  try {
    const res = await fetch(`../.artifacts/${artifactFile}`);
    body.textContent = res.ok ? await res.text() : `파일을 불러올 수 없습니다: ${res.status}`;
  } catch (e) {
    body.textContent = `오류: ${e.message}`;
  }
}

document.getElementById('modal-close').addEventListener('click', () => {
  document.getElementById('modal').hidden = true;
});

document.getElementById('modal').addEventListener('click', e => {
  if (e.target === document.getElementById('modal')) {
    document.getElementById('modal').hidden = true;
  }
});

async function init() {
  try {
    const data = await loadIndex();
    renderHeader(data);
    renderTracker(data.stages);
    renderHistory(data.stages);
    renderSecurity(data.securityGates);
  } catch (e) {
    document.getElementById('project-name').textContent = '오류: ' + e.message;
  }
}

init();
```

- [ ] **Step 2: 샘플 `index.json`으로 대시보드 동작 확인**

`.artifacts/index.json`을 다음 내용으로 임시 수정 후 브라우저에서 확인:

```json
{
  "project": "테스트 프로젝트",
  "stages": [
    { "stage": "brainstorm", "label": "요구사항 분석", "status": "approved", "timestamp": "2026-05-20T10:00:00Z", "commit": "abc1234", "artifact": null, "summary": "OAuth2 로그인 기능 추가" },
    { "stage": "plan",        "label": "계획",          "status": "approved", "timestamp": "2026-05-20T11:00:00Z", "commit": "def5678", "artifact": null, "summary": "3단계 구현: auth → api → ui" },
    { "stage": "exec",        "label": "구현",          "status": "in-progress", "timestamp": null, "commit": null, "artifact": null, "summary": null },
    { "stage": "verify",      "label": "검증",          "status": "pending",  "timestamp": null, "commit": null, "artifact": null, "summary": null }
  ],
  "currentStage": "exec",
  "securityGates": [
    { "file": "src/auth/login.ts", "reason": "auth 패턴 감지", "approvedAt": "2026-05-20T11:30:00Z" }
  ]
}
```

```bash
npx serve .
# http://localhost:3000/dashboard/ 열기
```

기대 결과:
- 상단에 "테스트 프로젝트" + "현재 단계: 구현" 표시
- Stage tracker에 ✅✅🔄⏳ 칩 4개
- 히스토리에 brainstorm, plan 행 + summary 텍스트
- 보안 이력 섹션에 `src/auth/login.ts` 항목

- [ ] **Step 3: index.json 원래대로 복원**

```json
{
  "project": "프로젝트명",
  "stages": [
    { "stage": "brainstorm", "label": "요구사항 분석", "status": "pending", "timestamp": null, "commit": null, "artifact": null, "summary": null },
    { "stage": "plan",        "label": "계획",          "status": "pending", "timestamp": null, "commit": null, "artifact": null, "summary": null },
    { "stage": "exec",        "label": "구현",          "status": "pending", "timestamp": null, "commit": null, "artifact": null, "summary": null },
    { "stage": "verify",      "label": "검증",          "status": "pending", "timestamp": null, "commit": null, "artifact": null, "summary": null }
  ],
  "currentStage": "brainstorm",
  "securityGates": []
}
```

- [ ] **Step 4: 커밋**

```bash
git add dashboard/app.js .artifacts/index.json
git commit -m "feat(dashboard): add app.js with stage tracker + history + popup"
```

---

## Task 5: `hooks/security-check.js` — 보안 게이트 (PreToolUse)

**Files:**
- Create: `hooks/security-check.js`
- Create: `hooks/security-check.test.js`

Claude Code `PreToolUse` hook으로 실행. stdin에서 JSON을 읽어 파일 경로가 보안 패턴에 매칭되면 exit 2로 블록.

- [ ] **Step 1: 테스트 작성**

파일 경로: `hooks/security-check.test.js`

```js
const { execSync, spawnSync } = require('child_process');
const path = require('path');

function runHook(toolInput) {
  const input = JSON.stringify({ tool_name: 'Edit', tool_input: toolInput });
  const result = spawnSync('node', ['hooks/security-check.js'], {
    input,
    encoding: 'utf8',
    cwd: path.join(__dirname, '..')
  });
  return result;
}

// Test 1: 안전한 파일 — 허용
const safe = runHook({ file_path: 'src/components/Button.tsx', old_string: '', new_string: '' });
console.assert(safe.status === 0, `Safe file should exit 0, got ${safe.status}`);

// Test 2: auth 패턴 — 블록
const auth = runHook({ file_path: 'src/auth/login.ts', old_string: '', new_string: '' });
console.assert(auth.status === 2, `Auth file should exit 2, got ${auth.status}`);

// Test 3: .env 패턴 — 블록
const env = runHook({ file_path: '.env.production', old_string: '', new_string: '' });
console.assert(env.status === 2, `.env file should exit 2, got ${env.status}`);

// Test 4: secret 패턴 — 블록
const secret = runHook({ file_path: 'config/secrets.yaml', old_string: '', new_string: '' });
console.assert(secret.status === 2, `Secret file should exit 2, got ${secret.status}`);

// Test 5: token 패턴 — 블록
const token = runHook({ file_path: 'src/utils/token-manager.ts', old_string: '', new_string: '' });
console.assert(token.status === 2, `Token file should exit 2, got ${token.status}`);

console.log('All security-check tests passed.');
```

- [ ] **Step 2: 테스트 실행 — 실패 확인**

```bash
node hooks/security-check.test.js
```

기대 출력: `Error: Cannot find module` (또는 유사한 오류)

- [ ] **Step 3: `hooks/security-check.js` 구현**

```js
#!/usr/bin/env node

const SECURITY_PATTERNS = [
  /auth/i,
  /secret/i,
  /\.env/i,
  /permission/i,
  /token/i,
  /credential/i,
  /password/i,
  /private.?key/i,
  /api.?key/i,
];

function isSecuritySensitive(filePath) {
  return SECURITY_PATTERNS.some(p => p.test(filePath));
}

let rawInput = '';
process.stdin.on('data', chunk => { rawInput += chunk; });

process.stdin.on('end', () => {
  let hookData;
  try {
    hookData = JSON.parse(rawInput);
  } catch {
    // JSON 파싱 실패 시 허용 (정상 실행 방해 안 함)
    process.exit(0);
  }

  const toolName = hookData.tool_name || '';
  const filePath = hookData.tool_input?.file_path || hookData.tool_input?.path || '';

  // Edit, Write 툴만 검사
  if (!['Edit', 'Write'].includes(toolName)) {
    process.exit(0);
  }

  if (!filePath || !isSecuritySensitive(filePath)) {
    process.exit(0);
  }

  // 보안 민감 파일 — stderr에 경고 출력 후 exit 2로 블록
  process.stderr.write(
    `\n🔒 [보안 게이트] 민감한 파일 감지: ${filePath}\n` +
    `이 파일을 수정하려면 개발자 승인이 필요합니다.\n` +
    `승인 후 node scripts/update-artifact.js --security-gate --file "${filePath}" --reason "수동 승인" 을 실행하세요.\n\n`
  );
  process.exit(2);
});
```

- [ ] **Step 4: 테스트 실행 — 통과 확인**

```bash
node hooks/security-check.test.js
```

기대 출력: `All security-check tests passed.`

- [ ] **Step 5: 커밋**

```bash
git add hooks/
git commit -m "feat(hooks): add security-check.js PreToolUse gate with pattern matching"
```

---

## Task 6: `.claude/settings.json` — Hook 배선

**Files:**
- Create: `.claude/settings.json`

- [ ] **Step 1: `.claude/settings.json` 작성**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "node hooks/security-check.js"
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 2: hook이 Claude Code에서 인식되는지 확인**

```bash
# Claude Code CLI에서 현재 디렉토리로 세션 시작 후
# /status 또는 설정 확인 명령으로 hook이 로드됐는지 확인
```

- [ ] **Step 3: 커밋**

```bash
git add .claude/settings.json
git commit -m "feat(config): wire security-check hook in Claude Code settings"
```

---

## Task 7: `commands/brainstorm.md` — /brainstorm 슬래시 명령어

**Files:**
- Create: `commands/brainstorm.md`

Claude Code 슬래시 명령어. `/brainstorm` 실행 시 Claude가 이 파일의 지시에 따라 행동한다.

- [ ] **Step 1: `commands/brainstorm.md` 작성**

```markdown
# /brainstorm — 요구사항 분석 (인터뷰 방식)

당신은 요구사항 분석 전문가입니다. 신선한 컨텍스트로 시작합니다.

## 목적
사용자와 1:1 인터뷰를 통해 요구사항을 명확히 파악하고, `.artifacts/` 폴더에 설계 문서를 생성합니다.

## 규칙
- **한 번에 하나의 질문만** 합니다. 절대 여러 질문을 동시에 하지 마세요.
- **가능하면 객관식** (A/B/C 선택지) 형태로 질문합니다.
- **YAGNI 원칙**: 필요 이상의 기능을 가정하지 마세요.
- 충분히 파악됐다고 판단되면 요약 후 설계 문서를 작성합니다.

## 인터뷰 진행 순서
1. 무엇을 만들고 싶은지 (목적)
2. 누가 사용하는지 (사용자)
3. 핵심 제약사항 (기술, 기간, 환경)
4. 성공 기준 (어떤 상태가 되면 완성인가)

## 완료 시 작업

모든 질문이 완료되면 다음을 순서대로 실행합니다:

1. `.artifacts/requirements-<YYYYMMDDTHHMMSS>.md` 파일 작성:
   - 목적, 사용자, 제약사항, 성공 기준을 명확히 기술

2. `.artifacts/design-<YYYYMMDDTHHMMSS>.md` 파일 작성:
   - 접근 방식, 핵심 컴포넌트, 데이터 흐름 개요

3. git 커밋:
   ```bash
   git add .artifacts/
   git commit -m "docs(brainstorm): capture requirements and design"
   ```

4. index.json 업데이트:
   ```bash
   node scripts/update-artifact.js \
     --stage brainstorm \
     --status approved \
     --summary "<요구사항 한 줄 요약>" \
     --artifact "requirements-<timestamp>.md"
   ```

5. git 커밋:
   ```bash
   git add .artifacts/index.json
   git commit -m "chore(artifacts): update index after brainstorm"
   ```

6. 사용자에게 다음을 안내합니다:
   > "요구사항 분석이 완료됐습니다. 설계가 맞다면 `/plan`을 실행해 구현 계획을 작성하세요."
```

- [ ] **Step 2: 내용 완전성 검증**

다음 항목이 모두 포함됐는지 확인:
- [ ] 인터뷰 규칙 (1문 1답)
- [ ] 작성 파일 목록 (`requirements-*.md`, `design-*.md`)
- [ ] git 커밋 명령어
- [ ] `update-artifact.js` 호출
- [ ] 다음 단계 안내 (`/plan`)

- [ ] **Step 3: 커밋**

```bash
git add commands/brainstorm.md
git commit -m "feat(commands): add /brainstorm interview-style requirements gathering"
```

---

## Task 8: `commands/plan.md` — /plan 슬래시 명령어

**Files:**
- Create: `commands/plan.md`

- [ ] **Step 1: `commands/plan.md` 작성**

```markdown
# /plan — 구현 계획 생성

당신은 시니어 소프트웨어 아키텍트입니다. 신선한 컨텍스트로 시작합니다.

## 입력 (읽기 전용)
다음 파일을 읽어 요구사항과 설계를 파악합니다:
- `.artifacts/requirements-*.md` (가장 최근 파일)
- `.artifacts/design-*.md` (가장 최근 파일)

## 목적
요구사항을 바탕으로 **단계별, 검증 가능한 구현 계획**을 작성합니다.

## 계획 작성 규칙
- 각 태스크는 **2~5분 단위**의 단일 작업
- 각 태스크에는 **실제 코드 또는 명령어** 포함
- TDD 원칙 적용: 테스트 → 실패 확인 → 구현 → 통과 확인 → 커밋
- YAGNI: 요구사항에 없는 기능 추가 금지
- 플레이스홀더("TBD", "TODO") 사용 금지

## 완료 시 작업

1. `.artifacts/plan-<YYYYMMDDTHHMMSS>.md` 파일 작성 (전체 구현 계획)

2. git 커밋:
   ```bash
   git add .artifacts/plan-*.md
   git commit -m "docs(plan): write implementation plan"
   ```

3. index.json 업데이트:
   ```bash
   node scripts/update-artifact.js \
     --stage plan \
     --status approved \
     --summary "<계획 한 줄 요약>" \
     --artifact "plan-<timestamp>.md"
   ```

4. git 커밋:
   ```bash
   git add .artifacts/index.json
   git commit -m "chore(artifacts): update index after plan"
   ```

5. 사용자에게 다음을 안내합니다:
   > "구현 계획이 작성됐습니다. `.artifacts/plan-*.md`를 검토 후 `/exec`를 실행해 구현을 시작하세요."
```

- [ ] **Step 2: 내용 완전성 검증**

- [ ] 입력 파일 명시 (requirements, design 읽기)
- [ ] TDD 원칙 언급
- [ ] git 커밋 명령어
- [ ] `update-artifact.js` 호출
- [ ] 다음 단계 안내 (`/exec`)

- [ ] **Step 3: 커밋**

```bash
git add commands/plan.md
git commit -m "feat(commands): add /plan implementation planning command"
```

---

## Task 9: `commands/exec.md` — /exec 슬래시 명령어

**Files:**
- Create: `commands/exec.md`

- [ ] **Step 1: `commands/exec.md` 작성**

```markdown
# /exec — 코드 구현

당신은 숙련된 소프트웨어 엔지니어입니다. 신선한 컨텍스트로 시작합니다.

## 입력 (읽기 전용)
다음 파일을 읽어 구현 계획을 파악합니다:
- `.artifacts/plan-*.md` (가장 최근 파일)

## 목적
구현 계획의 각 태스크를 **순서대로** 실행합니다.

## 실행 규칙
- 계획에 없는 코드를 추가하지 않습니다 (YAGNI)
- 각 태스크 완료 후 반드시 커밋합니다
- 보안 민감 파일 수정 시 hook이 자동으로 블록합니다 — 개발자 승인 후 진행하세요
- 오류 발생 시: 오류 내용을 기록하고 중단합니다 (무한 재시도 금지)

## 완료 시 작업

1. `.artifacts/implementation-<YYYYMMDDTHHMMSS>.md` 파일 작성:
   - 구현한 파일 목록
   - 각 파일의 주요 변경 사항 요약
   - 발생한 오류 및 해결 방법 (있는 경우)

2. git 커밋:
   ```bash
   git add .artifacts/implementation-*.md
   git commit -m "docs(exec): record implementation summary"
   ```

3. index.json 업데이트:
   ```bash
   node scripts/update-artifact.js \
     --stage exec \
     --status approved \
     --summary "<구현 한 줄 요약>" \
     --artifact "implementation-<timestamp>.md"
   ```

4. git 커밋:
   ```bash
   git add .artifacts/index.json
   git commit -m "chore(artifacts): update index after exec"
   ```

5. 사용자에게 다음을 안내합니다:
   > "구현이 완료됐습니다. `/verify`를 실행해 결과를 검증하세요."

## 오류 처리
구현 중 오류가 발생하면:
1. 오류 메시지와 원인을 `.artifacts/implementation-<timestamp>.md`에 기록
2. `node scripts/update-artifact.js --stage exec --status rejected --summary "오류: <오류 요약>"`
3. 사용자에게 상황 보고 후 중단
```

- [ ] **Step 2: 내용 완전성 검증**

- [ ] 입력 파일 명시 (plan 읽기)
- [ ] 보안 hook 언급
- [ ] 오류 처리 절차
- [ ] 아티팩트 파일 작성
- [ ] 다음 단계 안내 (`/verify`)

- [ ] **Step 3: 커밋**

```bash
git add commands/exec.md
git commit -m "feat(commands): add /exec code implementation command"
```

---

## Task 10: `commands/verify.md` — /verify 슬래시 명령어

**Files:**
- Create: `commands/verify.md`

- [ ] **Step 1: `commands/verify.md` 작성**

```markdown
# /verify — 구현 결과 검증

당신은 QA 엔지니어입니다. 신선한 컨텍스트로 시작합니다.

## 입력 (읽기 전용)
다음 파일을 읽어 요구사항과 구현 내용을 파악합니다:
- `.artifacts/requirements-*.md` (가장 최근 파일)
- `.artifacts/implementation-*.md` (가장 최근 파일)

## 목적
구현 결과가 요구사항을 충족하는지 **독립적으로** 검증합니다.

## 검증 항목
1. 요구사항의 각 항목이 구현됐는가
2. 핵심 흐름이 정상 작동하는가 (테스트 실행)
3. 성공 기준이 충족됐는가

## 검증 실행
```bash
# 프로젝트의 테스트 실행 (해당되는 경우)
npm test
# 또는
pytest
# 또는 해당 언어의 테스트 명령어
```

## 완료 시 작업

1. `.artifacts/verification-<YYYYMMDDTHHMMSS>.md` 파일 작성:
   - 요구사항별 충족 여부 체크리스트
   - 테스트 실행 결과
   - 미충족 항목 (있는 경우)

2. git 커밋:
   ```bash
   git add .artifacts/verification-*.md
   git commit -m "docs(verify): record verification results"
   ```

3. 검증 결과에 따라:
   - **통과** 시: `node scripts/update-artifact.js --stage verify --status approved --summary "모든 요구사항 충족" --artifact "verification-<timestamp>.md"`
   - **실패** 시: `node scripts/update-artifact.js --stage verify --status rejected --summary "미충족: <항목>" --artifact "verification-<timestamp>.md"`

4. git 커밋:
   ```bash
   git add .artifacts/index.json
   git commit -m "chore(artifacts): update index after verify"
   ```

5. 사용자에게 결과 보고:
   - **통과** 시: "검증이 완료됐습니다. 모든 요구사항이 충족됐습니다."
   - **실패** 시: "미충족 항목이 있습니다. `/exec`를 재실행해 수정하세요." (미충족 항목 목록 포함)
```

- [ ] **Step 2: 내용 완전성 검증**

- [ ] 입력 파일 명시 (requirements, implementation 읽기)
- [ ] 통과/실패 두 가지 경로 모두 처리
- [ ] 아티팩트 파일 작성
- [ ] 다음 단계 안내 (완료 또는 재실행)

- [ ] **Step 3: 커밋**

```bash
git add commands/verify.md
git commit -m "feat(commands): add /verify QA verification command"
```

---

## Task 11: `CLAUDE.md` — 하네스 설정

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: `CLAUDE.md` 작성**

```markdown
# Lean Harness

경량 Claude Code 하네스. superpowers의 훈련된 프로세스 + GSD의 신선한 컨텍스트 방식.

## 핵심 원칙

- **SSOT = Git**: 모든 산출물은 `.artifacts/`에 git 커밋. 별도 DB 없음.
- **신선한 컨텍스트**: 각 슬래시 명령어는 독립된 서브에이전트로 실행.
- **사람이 핵심 게이트 제어**: 각 단계 완료 후 개발자가 다음 명령어를 직접 실행.
- **YAGNI + TDD**: 필요한 것만, 테스트 우선.

## 워크플로우

```
/brainstorm → (설계 확인) → /plan → (계획 확인) → /exec → /verify
```

각 단계는 `.artifacts/` 폴더에 마크다운 파일을 생성하고 git 커밋합니다.

## 슬래시 명령어

| 명령어 | 설명 |
|---|---|
| `/brainstorm` | 인터뷰 방식으로 요구사항 파악 → `requirements.md` + `design.md` 생성 |
| `/plan` | 설계 기반 단계별 구현 계획 작성 → `plan.md` 생성 |
| `/exec` | 계획 실행 → 코드 구현 → `implementation.md` 생성 |
| `/verify` | 구현 결과 검증 → `verification.md` 생성 |

## 에이전트

- **Planner** (`/plan`): 요구사항 → 실행 계획
- **Executor** (`/exec`): 계획 → 코드
- **Verifier** (`/verify`): 결과 → 검증

## 아티팩트 규칙

각 명령어 완료 시:
1. `.artifacts/<stage>-<YYYYMMDDTHHMMSS>.md` 작성
2. `git commit`
3. `node scripts/update-artifact.js --stage <name> --status approved --summary "<요약>" --artifact "<파일명>"`
4. `git commit`

## 보안 게이트

`Edit`, `Write` 도구 호출 시 `hooks/security-check.js`가 자동 실행됩니다.
민감한 파일 패턴 감지 시 블록 → 개발자 승인 후 `scripts/update-artifact.js --security-gate` 실행.

## 대시보드

```bash
npx serve .
# http://localhost:3000/dashboard/ 에서 진행 현황 확인
```

PM, QA, 기획자는 대시보드에서 진행 상황과 단계별 이력을 확인합니다.
```

- [ ] **Step 2: 내용 완전성 검증**

- [ ] 4개 슬래시 명령어 설명
- [ ] 아티팩트 작성 규칙
- [ ] 보안 게이트 설명
- [ ] 대시보드 실행 방법

- [ ] **Step 3: 커밋**

```bash
git add CLAUDE.md
git commit -m "feat: add CLAUDE.md lightweight harness configuration"
```

---

## Task 12: E2E 스모크 테스트

**Files:**
- Create: `scripts/smoke-test.sh`

전체 파이프라인을 모의 데이터로 검증합니다.

- [ ] **Step 1: `scripts/smoke-test.sh` 작성**

```bash
#!/usr/bin/env bash
set -e

echo "=== Lean Harness 스모크 테스트 ==="

# 1. 초기 상태 확인
echo "1. index.json 유효성 확인..."
node -e "JSON.parse(require('fs').readFileSync('.artifacts/index.json', 'utf8')); console.log('  ✅ valid')"

# 2. brainstorm 단계 시뮬레이션
echo "2. brainstorm 단계 시뮬레이션..."
echo "# 요구사항\n- OAuth2 로그인 기능 추가" > .artifacts/requirements-smoke.md
echo "# 설계\n- JWT 기반 인증 플로우" > .artifacts/design-smoke.md
node scripts/update-artifact.js --stage brainstorm --status approved --summary "OAuth2 로그인" --artifact "requirements-smoke.md"
BRAINSTORM_STATUS=$(node -e "const d=JSON.parse(require('fs').readFileSync('.artifacts/index.json','utf8')); console.log(d.stages.find(s=>s.stage==='brainstorm').status)")
[ "$BRAINSTORM_STATUS" = "approved" ] && echo "  ✅ brainstorm approved" || (echo "  ❌ brainstorm 상태 오류: $BRAINSTORM_STATUS" && exit 1)

# 3. plan 단계 시뮬레이션
echo "3. plan 단계 시뮬레이션..."
echo "# 계획\n- Task 1: auth 모듈 구현" > .artifacts/plan-smoke.md
node scripts/update-artifact.js --stage plan --status approved --summary "auth 모듈 구현 계획" --artifact "plan-smoke.md"
PLAN_STATUS=$(node -e "const d=JSON.parse(require('fs').readFileSync('.artifacts/index.json','utf8')); console.log(d.stages.find(s=>s.stage==='plan').status)")
[ "$PLAN_STATUS" = "approved" ] && echo "  ✅ plan approved" || (echo "  ❌ plan 상태 오류" && exit 1)

# 4. 보안 hook 테스트
echo "4. 보안 hook 테스트..."
node hooks/security-check.test.js

# 5. 대시보드 파일 확인
echo "5. 대시보드 파일 확인..."
[ -f "dashboard/index.html" ] && echo "  ✅ index.html 존재" || (echo "  ❌ index.html 없음" && exit 1)
[ -f "dashboard/app.js" ] && echo "  ✅ app.js 존재" || (echo "  ❌ app.js 없음" && exit 1)

# 6. 스모크 테스트 아티팩트 정리
rm -f .artifacts/requirements-smoke.md .artifacts/design-smoke.md .artifacts/plan-smoke.md

# 7. index.json 초기화
node -e "
const fs = require('fs');
const data = {
  project: '프로젝트명',
  stages: [
    { stage: 'brainstorm', label: '요구사항 분석', status: 'pending', timestamp: null, commit: null, artifact: null, summary: null },
    { stage: 'plan',        label: '계획',          status: 'pending', timestamp: null, commit: null, artifact: null, summary: null },
    { stage: 'exec',        label: '구현',          status: 'pending', timestamp: null, commit: null, artifact: null, summary: null },
    { stage: 'verify',      label: '검증',          status: 'pending', timestamp: null, commit: null, artifact: null, summary: null }
  ],
  currentStage: 'brainstorm',
  securityGates: []
};
fs.writeFileSync('.artifacts/index.json', JSON.stringify(data, null, 2));
"

echo ""
echo "=== 모든 스모크 테스트 통과 ✅ ==="
```

- [ ] **Step 2: 실행 권한 부여 후 테스트**

```bash
chmod +x scripts/smoke-test.sh
bash scripts/smoke-test.sh
```

기대 출력:
```
=== Lean Harness 스모크 테스트 ===
1. index.json 유효성 확인...
  ✅ valid
2. brainstorm 단계 시뮬레이션...
  ✅ brainstorm approved
3. plan 단계 시뮬레이션...
  ✅ plan approved
4. 보안 hook 테스트...
All security-check tests passed.
5. 대시보드 파일 확인...
  ✅ index.html 존재
  ✅ app.js 존재

=== 모든 스모크 테스트 통과 ✅ ===
```

- [ ] **Step 3: 커밋**

```bash
git add scripts/smoke-test.sh
git commit -m "test(e2e): add smoke test for full pipeline"
```

---

## 완료 기준

- [ ] `bash scripts/smoke-test.sh` 전체 통과
- [ ] `node hooks/security-check.test.js` 전체 통과
- [ ] `node scripts/update-artifact.test.js` 전체 통과
- [ ] `npx serve .` 후 `dashboard/index.html`이 샘플 데이터를 올바르게 렌더링
- [ ] Claude Code에서 `/brainstorm` 명령어가 인식됨
