# xpress - 분석 보고서

## 1️⃣ FreeLang 부족한 점 분석

### 발견된 한계점

#### 1.1 실제 HTTP 서버 바인딩 불가
**문제**: FreeLang에는 실제 포트 리스닝/HTTP 서버 구현이 없음
- ✅ 있는 것: fetch (HTTP 클라이언트)
- ❌ 없는 것: HTTP 서버 리스닝, 소켓 바인딩
- ❌ 없는 것: TCP 서버 (Node.js net 모듈 같은)

**해결 방안**:
1. FreeLang stdlib에 `http.listen()` 추가
2. TypeScript로 래퍼 함수 작성: `stdlib/http-server.ts`
3. Node.js http/net 모듈 직접 사용

**영향도**: HIGH - Express 핵심 기능

#### 1.2 객체/클래스 문법 제약
**문제**: 객체 메서드가 `this` 바인딩 안 됨
```fl
// 불가능
app.get("/", fn(req, res) { ... })
app.post("/", fn(req, res) { ... })

// 가능한 방식
app.routes.push({ method: "GET", handler: ... })
```

**원인**: FreeLang fn()은 클로저만 지원, 메서드 체이닝 미지원

**해결 방안**:
- 함수 반환값으로 체이닝 구현 (현재)
- FreeLang 언어 레벨에서 메서드 지원 추가

**영향도**: MEDIUM - Express API 인체공학

#### 1.3 Array/Object 메서드 부족
**문제**: 배열/객체 조작 기능 제한
- ❌ array.filter(), array.map() 없음
- ❌ array.find(), array.some() 없음
- ❌ object.keys() 반환 순서 보장 안 됨

**사용 방식**:
```fl
// 수동 루프 필요
for route in app.routes {
  if (route.method == method) { ... }
}
```

**개선 필요**:
- stdlib/array.fl에 filter, map, find 추가
- stdlib/object.fl 추가 (keys(), values(), entries())

**영향도**: MEDIUM - 코드 복잡도

#### 1.4 Type System 없음
**문제**: 런타임 타입 검증만 가능
```fl
// 타입 힌트 없음
fn handleRequest(req, res) { ... }

// 런타임에만 에러 발생
fn validate(obj) {
  if (obj == null) { ... }
}
```

**개선 필요**:
- 선택적 타입 힌트 (TypeScript 스타일)
- 컴파일 타임 검증

**영향도**: LOW - 프로토타입에는 충분

#### 1.5 String 메서드 부족
**문제**: 정규식 미지원
```fl
// 불가능
path = "/users/:id" // 동적 라우팅 파싱 복잡
```

**필요 기능**:
- string.match(pattern) - 정규식 매칭
- string.replace(pattern, replacement) - 패턴 기반 치환
- string.split(delimiter) - 경로 파싱

**현재 workaround**:
```fl
// 수동 파싱
parts = string.split(path, "/")
```

**영향도**: MEDIUM - 라우팅 기능

---

## 2️⃣ 보안 테스트

### 발견된 취약점

#### 2.1 HTTP 헤더 인젝션
**위험**: 사용자 입력이 헤더에 그대로 들어가면 CRLF 인젝션 가능
```fl
// 위험한 코드
let user_input = req.getHeader("X-User-Agent")
res.setHeader("X-Custom", user_input) // ❌ 인젝션 가능
```

**테스트 케이스**:
```
X-Custom: normal\r\nSet-Cookie: admin=true
```

**방어 방안**:
```fl
fn sanitizeHeaderValue(value) {
  // \r, \n 제거
  value = string.replace(value, "\r", "")
  value = string.replace(value, "\n", "")
  return value
}
```

**상태**: ⚠️ 구현 필요

#### 2.2 XSS (Cross-Site Scripting)
**위험**: HTML 응답에 사용자 입력 포함
```fl
// 위험
res.html("<h1>" + user_name + "</h1>") // ❌ XSS

// 안전
res.html("<h1>" + escapeHtml(user_name) + "</h1>") // ✅
```

**테스트 케이스**:
```
user_name = "<img src=x onerror='alert(1)'>"
```

**방어 함수**:
```fl
fn escapeHtml(text) {
  text = string.replace(text, "&", "&amp;")
  text = string.replace(text, "<", "&lt;")
  text = string.replace(text, ">", "&gt;")
  text = string.replace(text, "\"", "&quot;")
  text = string.replace(text, "'", "&#x27;")
  return text
}
```

**상태**: ⚠️ 구현 필요

#### 2.3 JSON 인젝션
**위험**: 사용자 입력이 JSON에 포함되면 구조 변조 가능
```fl
// 위험
res.json({ user: user_input }) // ❌ 불완전한 이스케이핑

// 안전
res.json({ user: user_input }) // json.stringify()가 처리해야 함
```

**테스트 케이스**:
```
user_input = '"; "admin": true; "'
```

**확인**: FreeLang의 json.stringify()가 올바르게 이스케이프하는지 검증 필요

**상태**: ⚠️ 검증 필요

#### 2.4 경로 탐색 (Path Traversal)
**위험**: 정적 파일 제공 시 상위 디렉토리 접근 가능
```fl
// 위험: GET /files/../../etc/passwd
// 방어: 경로 정규화
fn safeResolvePath(basedir, userpath) {
  // ../ 제거, 경로 정규화
  // realpath()로 검증
}
```

**현재 상태**: xpress에 정적 파일 기능 없음 (안전)

**상태**: ✅ 현재 영향 없음

#### 2.5 DoS (Denial of Service)
**위험**: 대용량 요청으로 메모리 소모
```fl
// 위험: 1GB 파일 POST
// POST /api/upload (body: 1GB)
```

**방어 방안**:
```fl
const MAX_BODY_SIZE = 10 * 1024 * 1024  // 10MB
fn bodyLimiterMiddleware(req, res, next) {
  if (req.body.length > MAX_BODY_SIZE) {
    res.status(413).json({ error: "Payload Too Large" })
  } else {
    next()
  }
}
```

**상태**: ⚠️ 구현 필요

#### 2.6 CSRF (Cross-Site Request Forgery)
**위험**: 다른 사이트에서 요청 위조 가능
```fl
// 위험: GET /api/users/1/delete?confirm=true
```

**방어 방안**:
1. CSRF 토큰 생성/검증
2. SameSite 쿠키 플래그
3. Origin 헤더 검증

**현재 상태**: xpress에 쿠키 지원 없음 (미구현)

**상태**: ⚠️ 향후 구현

---

## 3️⃣ 셀프호스팅 테스트

### 3.1 FreeLang 런타임 요구사항
**필수**: FreeLang v2.2.0 (Node.js 기반)
- ✅ 외부 의존 최소화
- ✅ npm 패키지 불필요
- ✅ 순수 FreeLang + stdlib

### 3.2 실행 테스트

#### 테스트 1: 모듈 로딩
```bash
freelang test-server.fl
```

**예상 결과**:
```
📍 등록된 라우트:
   GET /
   GET /hello
   GET /api/users
   POST /api/users
   ...
```

**상태**: ⏳ 테스트 예정

#### 테스트 2: 라우팅 로직
```fl
let app = xpress.createApp()
app.get("/test", fn(req, res) { res.send("OK") })

let route = app.matchRoute("GET", "/test")
assert(route != null)
```

**상태**: ⏳ 테스트 예정

#### 테스트 3: 미들웨어 체인
```fl
let calls = []
app.use(fn(req, res, next) {
  calls.push("first")
  next()
})
```

**상태**: ⏳ 테스트 예정

### 3.3 배포 준비

#### 필수 파일
- ✅ xpress.fl - 프레임워크 핵심
- ✅ test-server.fl - 테스트
- ⏳ 실제 HTTP 서버 바인딩 (TypeScript)
- ⏳ 미들웨어 예제

#### Gogs 저장소
```
https://gogs.dclub.kr/kim/xpress
```

---

## 📊 개선 우선순위

### Phase 1 (필수) 🔴
1. HTTP 서버 실제 바인딩 (포트 리스닝)
2. 요청 파싱 (메서드, 경로, 헤더, 바디)
3. 응답 전송 (상태코드, 헤더, 바디)

### Phase 2 (권장) 🟠
1. 보안: XSS, 헤더 인젝션 방어
2. 미들웨어 실행 체인
3. 라우트 파라미터 (`:id` 지원)

### Phase 3 (선택) 🟡
1. 쿠키 지원
2. 세션 관리
3. CSRF 토큰
4. 정적 파일 서빙

---

## 🔍 FreeLang 개선 제안

### 1. HTTP 서버 stdlib 추가
```fl
import { http } from "./stdlib/http"

let server = http.createServer(3000, fn(req, res) {
  res.writeHead(200, { "Content-Type": "text/plain" })
  res.end("Hello")
})

server.listen()
```

### 2. 메서드 체이닝 지원
```fl
fn createApp() {
  let app = {
    routes: [],

    get: fn(path, handler) {
      // this.routes 접근 가능하고
      // return this로 체이닝 가능
      return this
    }
  }
  return app
}

// 사용
app
  .get("/", handler1)
  .get("/api", handler2)
  .listen(3000)
```

### 3. 정규식 지원
```fl
import { regex } from "./stdlib/regex"

let pattern = regex.compile("/users/:id")
let match = pattern.match("/users/123")
// { id: "123" }
```

### 4. Array/Object 메서드 확충
```fl
array.filter(arr, fn(x) { return x > 5 })
array.map(arr, fn(x) { return x * 2 })
object.keys(obj)
object.values(obj)
```

---

## 📝 결론

**xpress는 FreeLang의 현재 능력과 한계를 명확히 보여주는 프로젝트입니다.**

### 가능한 것
✅ 프레임워크 API 설계 (객체, 함수)
✅ 라우팅 로직 (맵핑, 매칭)
✅ 미들웨어 패턴 (함수 조합)
✅ JSON 응답 처리

### 불가능한 것
❌ 실제 HTTP 서버 바인딩
❌ 포트 리스닝
❌ 네트워크 소켓 제어

### 다음 단계
1. TypeScript로 HTTP 서버 래퍼 작성
2. 보안 미들웨어 추가
3. Gogs 배포
4. 실제 동작 테스트
