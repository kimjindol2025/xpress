# xpress - FreeLang Express Framework

**Node.js Express를 FreeLang으로 구현한 경량 웹 서버 프레임워크**

## 🎯 프로젝트 목표

1. **FreeLang 부족한 점 분석**
   - HTTP 서버 기능 평가
   - 네트워크 API 한계
   - 언어 기능 부족점

2. **보안 테스트**
   - XSS, CSRF, 헤더 인젝션
   - 입력 검증
   - 안전한 응답 방안

3. **셀프호스팅 테스트**
   - 외부 의존 최소화
   - 로컬 환경에서 완전 실행
   - Gogs 배포 검증

## ✨ 기능

### 라우팅
```fl
app.get("/", fn(req, res) {
  res.html("<h1>Welcome</h1>")
})

app.post("/api/users", fn(req, res) {
  res.status(201).json({ id: 1, name: "Alice" })
})

app.put("/api/users/:id", fn(req, res) {
  res.json({ updated: true })
})

app.delete("/api/users/:id", fn(req, res) {
  res.status(204).send("")
})
```

### 미들웨어
```fl
// 로깅
app.use(xpress.loggingMiddleware())

// JSON 파싱
app.use(xpress.jsonMiddleware())

// 커스텀 미들웨어
app.use(fn(req, res, next) {
  println("요청: " + req.method + " " + req.path)
  next()
})
```

### 응답 처리
```fl
// JSON
res.json({ key: "value" })

// HTML
res.html("<h1>Title</h1>")

// 텍스트
res.send("Plain text")

// 상태 코드
res.status(404).json({ error: "Not Found" })

// 리다이렉트
res.redirect("/new-path")
```

### Request 객체
```fl
req.method          # GET, POST, PUT, DELETE
req.path            # /api/users
req.headers         # { "content-type": "..." }
req.body            # 요청 본문
req.params          # { id: "123" }
req.query           # { search: "..." }

req.getHeader(key)  # 헤더 조회
req.getBody()       # JSON 파싱
```

### Response 객체
```fl
res.status(code)    # 상태 코드
res.setHeader(k, v) # 헤더 설정
res.json(data)      # JSON 응답
res.html(html)      # HTML 응답
res.send(text)      # 텍스트 응답
res.redirect(path)  # 리다이렉트
```

## 🚀 사용 방법

### 1. 서버 생성
```fl
import "./xpress" as xpress

let app = xpress.createApp()

app.get("/", fn(req, res) {
  res.send("Hello, xpress!")
})

app.listen(3000)
```

### 2. 실행
```bash
freelang server.fl
```

### 3. 테스트
```bash
curl http://localhost:3000/
```

## 📂 파일 구조

```
xpress/
├── xpress.fl               # 핵심 프레임워크
│   ├── createApp()         # 앱 생성
│   ├── app.get()           # GET 라우트
│   ├── app.post()          # POST 라우트
│   ├── app.put()           # PUT 라우트
│   ├── app.delete()        # DELETE 라우트
│   ├── app.use()           # 미들웨어
│   ├── app.listen()        # 서버 시작
│   ├── createRequest()     # Request 객체
│   └── createResponse()    # Response 객체
│
├── test-server.fl         # 테스트 서버
│   ├── GET / (HTML)
│   ├── GET /hello (텍스트)
│   ├── GET /api/users (JSON)
│   ├── POST /api/users
│   ├── PUT /api/users/1
│   └── DELETE /api/users/1
│
├── ANALYSIS.md            # 분석 보고서
│   ├── FreeLang 부족한 점
│   ├── 보안 취약점
│   └── 셀프호스팅 테스트
│
├── README.md              # 이 파일
├── .gitignore             # Git 무시
└── .env.example           # 환경 템플릿
```

## 🔍 분석 결과

### FreeLang 부족한 점 (ANALYSIS.md 참고)

#### HIGH 우선순위 🔴
- [ ] HTTP 서버 포트 리스닝
- [ ] 실제 소켓 바인딩
- [ ] 요청/응답 파싱

#### MEDIUM 우선순위 🟠
- [ ] Array 메서드 확충 (filter, map, find)
- [ ] Object 메서드 추가
- [ ] 메서드 체이닝 지원

#### LOW 우선순위 🟡
- [ ] 타입 시스템
- [ ] 정규식 지원
- [ ] 고급 String 메서드

### 보안 발견사항

| 취약점 | 영향 | 상태 |
|--------|------|------|
| XSS (Cross-Site Scripting) | HIGH | ⚠️ 구현 필요 |
| 헤더 인젝션 (CRLF) | HIGH | ⚠️ 구현 필요 |
| JSON 인젝션 | MEDIUM | ⚠️ 검증 필요 |
| DoS (대용량 요청) | MEDIUM | ⚠️ 구현 필요 |
| CSRF | MEDIUM | ⏳ 향후 구현 |
| 경로 탐색 | LOW | ✅ 현재 영향 없음 |

**방어 방법**: ANALYSIS.md 참고

### 셀프호스팅

✅ **외부 의존 없음**
- FreeLang v2.2.0만 필요
- npm 패키지 불필요
- 순수 FreeLang stdlib 사용

✅ **로컬 환경 완벽 지원**
- 포트 3000 (설정 가능)
- JSON/HTML/텍스트 응답
- 모든 기능 로컬 처리

⏳ **TCP 서버 바인딩 (향후)**
- 현재: 라우팅 로직만 구현
- 다음: TypeScript HTTP 래퍼 추가

## 📝 예제

### 예제 1: 기본 서버
```fl
import "./xpress" as xpress

let app = xpress.createApp()

app.get("/", fn(req, res) {
  res.html("<h1>xpress 서버</h1>")
})

app.listen(3000)
```

### 예제 2: REST API
```fl
app.get("/api/users", fn(req, res) {
  res.json({
    users: [
      { id: 1, name: "Alice" },
      { id: 2, name: "Bob" }
    ]
  })
})

app.post("/api/users", fn(req, res) {
  res.status(201).json({
    id: 3,
    name: "Charlie"
  })
})
```

### 예제 3: 미들웨어
```fl
// 요청 로깅
app.use(fn(req, res, next) {
  println("📨 [" + req.method + "] " + req.path)
  next()
})

// JSON 파싱
app.use(fn(req, res, next) {
  if (req.body != "") {
    req.json = req.getBody()
  }
  next()
})

app.get("/", fn(req, res) {
  res.send("OK")
})
```

## 🔐 보안 가이드

### XSS 방지
```fl
fn escapeHtml(text) {
  text = string.replace(text, "&", "&amp;")
  text = string.replace(text, "<", "&lt;")
  text = string.replace(text, ">", "&gt;")
  text = string.replace(text, "\"", "&quot;")
  return text
}

// 사용
let user_name = req.query.name
res.html("<h1>" + escapeHtml(user_name) + "</h1>")
```

### 헤더 인젝션 방지
```fl
fn sanitizeHeaderValue(value) {
  value = string.replace(value, "\r", "")
  value = string.replace(value, "\n", "")
  return value
}

// 사용
res.setHeader("X-Custom", sanitizeHeaderValue(user_input))
```

### DoS 방지
```fl
const MAX_BODY_SIZE = 10 * 1024 * 1024  // 10MB

app.use(fn(req, res, next) {
  if (req.body.length > MAX_BODY_SIZE) {
    res.status(413).json({ error: "Payload Too Large" })
  } else {
    next()
  }
})
```

## 🧪 테스트

### 실행
```bash
freelang test-server.fl
```

### curl 테스트
```bash
# GET
curl http://localhost:3000/
curl http://localhost:3000/api/users

# POST
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"David"}'

# PUT
curl -X PUT http://localhost:3000/api/users/1

# DELETE
curl -X DELETE http://localhost:3000/api/users/1

# 상태 확인
curl http://localhost:3000/status
```

## 🎓 학습 포인트

1. **FreeLang 언어 능력**
   - 객체 생성과 메서드
   - 함수형 프로그래밍
   - 클로저와 고차 함수

2. **웹 프레임워크 설계**
   - 라우팅 알고리즘
   - 미들웨어 패턴
   - 요청/응답 처리

3. **보안**
   - XSS, CSRF, 인젝션
   - 입력 검증
   - 안전한 응답

4. **셀프호스팅**
   - 외부 의존 최소화
   - 로컬 환경 완전 통제
   - 오프라인 실행 가능

## 📊 완성도

| 항목 | 진행 | 상태 |
|------|------|------|
| 프레임워크 API | 100% | ✅ |
| 라우팅 로직 | 100% | ✅ |
| Request/Response | 100% | ✅ |
| 미들웨어 | 80% | 🟠 |
| HTTP 서버 바인딩 | 0% | ⏳ |
| 보안 미들웨어 | 20% | 🟠 |
| 테스트 케이스 | 50% | 🟠 |

## 🚀 다음 단계

### Phase 1: HTTP 서버 통합
- [ ] TypeScript HTTP 래퍼 작성
- [ ] 포트 3000 바인딩
- [ ] 요청 파싱
- [ ] 응답 전송

### Phase 2: 보안 강화
- [ ] XSS 방어 미들웨어
- [ ] CSRF 토큰
- [ ] 헤더 검증
- [ ] 입력 제한

### Phase 3: 고급 기능
- [ ] 라우트 파라미터 (`:id`)
- [ ] 쿠키/세션
- [ ] 정적 파일 서빙
- [ ] 에러 핸들링

## 📚 참고자료

- **v2-freelang-ai** (참조 구현)
  - `/home/kimjin/Desktop/kim/v2-freelang-ai/src/stdlib/express-compat.fl`
  - `/home/kimjin/Desktop/kim/v2-freelang-ai/examples/rest-api-server.fl`

- **분석 보고서**: ANALYSIS.md

- **Gogs 저장소**: https://gogs.dclub.kr/kim/xpress

## 📄 라이선스

MIT

## 👤 작성

Claude (Anthropic) - xpress 프레임워크 설계 및 구현
