# React Server Components 실무 도입기: 알아야 할 모든 것

## 목차
1. [RSC란 무엇인가?](#rsc란-무엇인가)
2. [SSR vs RSC: 혼란스러운 이유](#ssr-vs-rsc-혼란스러운-이유)
3. [실무에서 마주한 5가지 핵심 이슈](#실무에서-마주한-5가지-핵심-이슈)
4. [성능: 숫자로 말하는 진실](#성능-숫자로-말하는-진실)
5. [보안: 2025년 12월의 뼈아픈 교훈](#보안-2025년-12월의-뼈아픈-교훈)
6. [언제 RSC를 써야 하는가?](#언제-rsc를-써야-하는가)
7. [마이그레이션 실전 가이드](#마이그레이션-실전-가이드)

---

## RSC란 무엇인가?

React Server Components(RSC)는 **서버에서만 실행되는 React 컴포넌트**입니다. 2020년 12월에 처음 발표되어, Next.js 13(2022년 10월)에서 첫 구현이 이루어졌고, 2025년 현재 React 생태계의 가장 뜨거운 논쟁거리이자 혁신입니다.

### 핵심 개념

```jsx
// Server Component (기본값, Next.js App Router)
async function ProductList() {
  // 서버에서만 실행됨
  const products = await db.query('SELECT * FROM products');
  
  return (
    <div>
      {products.map(product => (
        <ProductCard key={product.id} data={product} />
      ))}
    </div>
  );
}

// Client Component (명시적 선언 필요)
'use client';

function AddToCartButton({ productId }) {
  const [loading, setLoading] = useState(false);
  
  // 클라이언트에서만 실행됨
  const handleClick = () => {
    setLoading(true);
    // ...
  };
  
  return <button onClick={handleClick}>장바구니 담기</button>;
}
```

### RSC의 핵심 특징

1. **서버에서만 실행**: DB 직접 접근, 파일 시스템 사용 가능
2. **JavaScript 번들에 포함 안됨**: 클라이언트로 전송되지 않음
3. **async/await 지원**: 컴포넌트 자체가 async 함수 가능
4. **제약사항**: useState, useEffect, 이벤트 핸들러 사용 불가

---

## SSR vs RSC: 혼란스러운 이유

가장 많이 받는 질문: **"SSR이랑 뭐가 달라요?"**

### 전통적인 SSR의 동작

```
1. 서버: React 컴포넌트 실행 → HTML 생성
2. 클라이언트: HTML 수신 → 화면에 표시 (FCP)
3. 클라이언트: JavaScript 다운로드
4. 클라이언트: React 컴포넌트 다시 실행 → Hydration
5. 상호작용 가능 (TTI)
```

**문제점**: 모든 컴포넌트 코드가 클라이언트로 전송되고, **함수가 두 번 실행**됩니다.

### RSC의 동작

```
1. 서버: Server Component 실행 → 특수 포맷(RSC Payload)으로 직렬화
2. 서버: Client Component는 placeholder로 표시
3. 클라이언트: RSC Payload 수신 → React 트리 재구성
4. 클라이언트: Client Component만 다운로드 & Hydration
5. 상호작용 가능
```

**핵심 차이**: Server Component의 코드는 클라이언트로 전송되지 않으며, 한 번만 실행됩니다.

### 실제 번들 크기 비교

```jsx
// 예시: Markdown 렌더링
import marked from 'marked'; // 35.9KB (gzipped 11.2KB)
import sanitizeHtml from 'sanitize-html'; // 206KB (gzipped 63.3KB)

// Client Component: 약 242KB가 클라이언트로 전송
function ArticleClient({ markdown }) {
  const html = sanitizeHtml(marked(markdown));
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// Server Component: 0KB 전송 (서버에서만 실행)
async function ArticleServer({ id }) {
  const article = await db.getArticle(id);
  const html = sanitizeHtml(marked(article.content));
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}
```

---

## 실무에서 마주한 5가지 핵심 이슈

### 1. "어디가 서버고 어디가 클라이언트지?"

**가장 큰 혼란**: 코드만 봐서는 구분이 안됩니다.

```jsx
// 이 코드만 봐서는 Server인지 Client인지 알 수 없음
function MyComponent() {
  const data = useData();
  return <div>{data.title}</div>;
}
```

**해결책**: 명확한 네이밍 컨벤션

```jsx
// components/
// ├── product-list.server.tsx    // Server Component
// ├── add-to-cart.client.tsx     // Client Component
// └── product-card.tsx            // 상황에 따라 다름 (조심!)
```

### 2. "props로 뭘 전달할 수 있고 뭘 못 전달하나?"

```jsx
// ❌ 불가능: 함수는 직렬화 불가
<ServerComponent onClick={handleClick} />

// ❌ 불가능: Date 객체도 직렬화 문제
<ServerComponent createdAt={new Date()} />

// ✅ 가능: 직렬화 가능한 데이터
<ServerComponent 
  user={{ id: 1, name: 'John', createdAt: '2024-01-01' }} 
/>

// ✅ 가능: React 요소도 가능
<ServerComponent>
  <ClientComponent />
</ServerComponent>
```

**규칙**: Server → Client 전달 시 JSON.stringify() 가능한 값만 가능

### 3. "데이터 페칭이 느리면 어떡하지?"

초기 우려: Server Component가 느리면 전체 페이지가 느려지는 것 아닌가?

```jsx
// ❌ 나쁜 패턴: 순차 페칭
async function SlowPage() {
  const user = await fetchUser();        // 1초
  const posts = await fetchPosts();      // 2초
  const comments = await fetchComments(); // 3초
  // 총 6초!
}

// ✅ 좋은 패턴: 병렬 페칭
async function FastPage() {
  const [user, posts, comments] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchComments()
  ]);
  // 총 3초 (가장 긴 요청 기준)
}

// ✅ 더 좋은 패턴: Suspense로 점진적 로딩
async function BetterPage() {
  return (
    <>
      <UserInfo />  {/* 빠른 데이터 먼저 */}
      <Suspense fallback={<Skeleton />}>
        <Posts />  {/* 느린 데이터는 나중에 */}
      </Suspense>
    </>
  );
}
```

### 4. "상태 관리는 어떻게 하나?"

Server Component에서는 **상태가 없습니다**. 매 요청마다 새로 실행됩니다.

```jsx
// ❌ Server Component에서는 불가능
async function ServerCounter() {
  const [count, setCount] = useState(0); // Error!
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// ✅ Client Component로 분리
'use client';
function ClientCounter({ initialCount }) {
  const [count, setCount] = useState(initialCount);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Server Component는 초기값만 제공
async function ServerPage() {
  const initialCount = await db.getCount();
  return <ClientCounter initialCount={initialCount} />;
}
```

**실무 패턴**: 
- 데이터 페칭: Server Component
- 상태 관리: Client Component
- 최대한 Server에서 처리하고, 꼭 필요한 부분만 Client로

### 5. "기존 라이브러리가 안 돼요!"

많은 npm 패키지가 RSC를 고려하지 않고 만들어졌습니다.

```jsx
// react-query는 client-only
'use client'; // 이렇게 해야 함
import { useQuery } from '@tanstack/react-query';

// 또는 Server Component에서 직접 fetch
async function ServerComponent() {
  const data = await fetch('...').then(r => r.json());
  return <ClientComponent data={data} />;
}
```

**확인 방법**: 라이브러리가 `'use client'`를 선언했는지 확인

---

## 성능: 숫자로 말하는 진실

### 실제 측정 사례

한 SaaS 클라이언트의 리포팅 뷰 개선:
- **Before (CSR)**: 6초 이상
- **After (RSC + Next.js 19)**: 2초 미만
- **개선율**: 67% 단축

### 번들 크기 변화

Next.js 13에서 RSC 도입 시:
- **초기 JS 번들**: 18~29% 감소
- **실제 사례**: 300KB → 210KB

### 하지만 주의할 점

#### LCP(Largest Contentful Paint) 트레이드오프

서버에서 데이터를 페칭하면:
- **장점**: 클라이언트 코드 감소
- **단점**: 서버에서 데이터 대기 시간 발생

```
CSR: HTML(즉시) → JS 다운(1s) → 렌더링(0.5s) → 데이터 페칭(2s) → 표시
     총: 3.5초, LCP: 3.5초

SSR: 데이터 페칭(2s) → HTML 생성 → 전송 → 표시
     총: 2.2초, LCP: 2.2초 (개선!)

RSC: 데이터 페칭(2s) → RSC Payload → 표시
     총: 2.1초, LCP: 2.1초 (약간 더 개선)
```

**결론**: 빠른 엔드포인트가 있다면 RSC가 유리하지만, 느린 API라면 장점이 줄어듭니다.

---

## 보안: 2025년 12월의 뼈아픈 교훈

### CVE-2025-55182: React2Shell

2025년 12월 3일, **CVSS 10.0 (최고 등급)** 취약점이 공개되었습니다.

**문제**: RSC의 직렬화 과정에서 악의적인 페이로드를 통해 **인증 없이 원격 코드 실행(RCE)** 가능

**영향 범위**:
- React 19.0.0 ~ 19.2.0
- Next.js 13.3.x ~ 16.x (App Router 사용 시)
- React Router, Waku, Vite RSC 플러그인 등

**공격 성공률**: 거의 100%

### 추가 취약점

12월 11일 추가 공개:
- **CVE-2025-55184**: DoS 공격 가능 (CVSS 7.5)
- **CVE-2025-55183**: 소스 코드 노출 (CVSS 5.3)

### 실무 대응

```bash
# 즉시 업데이트 필수
npm install react@19.0.3 react-dom@19.0.3
# 또는
npm install next@14.2.35
```

**교훈**:
1. RSC는 복잡한 직렬화/역직렬화 메커니즘을 사용
2. 서버-클라이언트 경계가 명확하지 않아 보안 취약점 발생 가능
3. 프레임워크 업데이트를 즉시 적용하는 체계 필요

### 보안 체크리스트

```jsx
// ❌ 위험: 서버 함수에 민감 정보 하드코딩
async function ServerAction() {
  const apiKey = 'sk-1234567890'; // 소스 코드 노출 시 유출
  await fetch('https://api.example.com', {
    headers: { 'Authorization': `Bearer ${apiKey}` }
  });
}

// ✅ 안전: 환경 변수 사용
async function ServerAction() {
  const apiKey = process.env.API_KEY; // 환경 변수는 노출 안됨
  await fetch('https://api.example.com', {
    headers: { 'Authorization': `Bearer ${apiKey}` }
  });
}
```

---

## 언제 RSC를 써야 하는가?

### ✅ RSC가 적합한 경우

1. **데이터베이스 직접 접근이 필요한 경우**
```jsx
async function UserDashboard({ userId }) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    include: { posts: true, profile: true }
  });
  
  return <Dashboard data={user} />;
}
```

2. **대용량 라이브러리 사용**
- Markdown 파서, 이미지 처리, PDF 생성 등
- 클라이언트 번들 크기를 줄이고 싶을 때

3. **SEO가 중요한 콘텐츠**
- 블로그, 뉴스, 제품 페이지
- 서버에서 완전한 HTML 생성

4. **민감한 로직**
- API 키, 비즈니스 로직을 클라이언트에 노출하고 싶지 않을 때

### ❌ RSC가 부적합한 경우

1. **인터랙션이 많은 UI**
```jsx
// 복잡한 폼, 실시간 검색, 드래그앤드롭 등
// → Client Component 사용
'use client';
function SearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    const debounced = debounce(() => {
      fetch(`/api/search?q=${query}`).then(/* ... */);
    }, 300);
    debounced();
  }, [query]);
  
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

2. **레거시 라이브러리 의존성**
- RSC를 지원하지 않는 UI 라이브러리
- Context API를 많이 사용하는 경우

3. **WebSocket, 실시간 통신**
- Server Component는 요청-응답 모델
- 지속적인 연결이 필요한 경우 Client Component 사용

### 하이브리드 전략 (추천)

```jsx
// Server: 데이터 페칭 + 무거운 라이브러리
async function ProductPage({ id }) {
  const product = await db.getProduct(id);
  const recommendations = await getRecommendations(id);
  
  return (
    <>
      <ProductInfo product={product} />
      {/* Client: 인터랙션 */}
      <AddToCartButton productId={id} />
      <ReviewForm productId={id} />
      {/* Server: 정적 콘텐츠 */}
      <Recommendations items={recommendations} />
    </>
  );
}
```

---

## 마이그레이션 실전 가이드

### Phase 1: 준비 (1~2주)

#### 1단계: 팀 교육
- RSC 개념 공유
- Server/Client 구분 기준 합의
- 네이밍 컨벤션 정의

#### 2단계: 환경 구축
```bash
# Next.js 15+ 설치 (RSC 안정 버전)
npx create-next-app@latest my-app --app

# 또는 기존 프로젝트 업그레이드
npm install next@latest react@latest react-dom@latest
```

#### 3단계: 의존성 분석
```bash
# Client-only 라이브러리 찾기
npx are-you-es5 --verbose
```

### Phase 2: 점진적 전환 (2~4주)

#### Step 1: 간단한 페이지부터
```jsx
// pages/about.tsx (Before)
export default function About() {
  return <div>About Us</div>;
}

// app/about/page.tsx (After - Server Component)
export default function About() {
  return <div>About Us</div>;
}
// 아무 변경 없이도 Server Component!
```

#### Step 2: 데이터 페칭 이전
```jsx
// Before: Client-side fetching
'use client';
export default function Users() {
  const [users, setUsers] = useState([]);
  
  useEffect(() => {
    fetch('/api/users')
      .then(r => r.json())
      .then(setUsers);
  }, []);
  
  return <UserList users={users} />;
}

// After: Server Component
export default async function Users() {
  const users = await db.query('SELECT * FROM users');
  
  return <UserList users={users} />;
}
```

#### Step 3: Client Component 최소화
```jsx
// Before: 전체가 Client
'use client';
export default function ProductPage({ product }) {
  const [quantity, setQuantity] = useState(1);
  
  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <input 
        type="number" 
        value={quantity} 
        onChange={e => setQuantity(e.target.value)} 
      />
      <button>Add to Cart</button>
    </div>
  );
}

// After: Server + Client 분리
// app/product/[id]/page.tsx (Server)
export default async function ProductPage({ params }) {
  const product = await db.getProduct(params.id);
  
  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <AddToCartForm product={product} />
    </div>
  );
}

// components/add-to-cart-form.tsx (Client)
'use client';
export function AddToCartForm({ product }) {
  const [quantity, setQuantity] = useState(1);
  
  return (
    <>
      <input 
        type="number" 
        value={quantity} 
        onChange={e => setQuantity(e.target.value)} 
      />
      <button>Add to Cart</button>
    </>
  );
}
```

### Phase 3: 최적화 (진행 중)

#### 병렬 데이터 페칭
```jsx
async function Dashboard() {
  // ❌ 순차 (느림)
  const user = await getUser();
  const posts = await getPosts();
  const stats = await getStats();

  // ✅ 병렬 (빠름)
  const [user, posts, stats] = await Promise.all([
    getUser(),
    getPosts(),
    getStats()
  ]);
  
  return <DashboardView user={user} posts={posts} stats={stats} />;
}
```

#### Suspense 활용
```jsx
export default async function Page() {
  return (
    <>
      <Header /> {/* 빠른 정적 콘텐츠 */}
      
      <Suspense fallback={<Skeleton />}>
        <SlowComponent /> {/* 느린 데이터 페칭 */}
      </Suspense>
      
      <Footer />
    </>
  );
}
```

### 일반적인 실수와 해결

#### 실수 1: 모든 것을 Server Component로
```jsx
// ❌ 불필요한 Server Component
async function Button() {
  return <button>Click</button>;
}

// ✅ 정적이면 그냥 일반 컴포넌트로
function Button() {
  return <button>Click</button>;
}
```

#### 실수 2: Client Component 남용
```jsx
// ❌ 전체를 Client로
'use client';
export default function Page() {
  const [query, setQuery] = useState('');
  const staticContent = '...'; // 이건 Server에서 처리 가능한데!
  
  return (
    <>
      <Header content={staticContent} />
      <SearchBar value={query} onChange={setQuery} />
    </>
  );
}

// ✅ 필요한 부분만 Client로
export default function Page() {
  const staticContent = '...';
  
  return (
    <>
      <Header content={staticContent} /> {/* Server */}
      <SearchBar /> {/* Client */}
    </>
  );
}
```

#### 실수 3: Props로 함수 전달
```jsx
// ❌ Server → Client로 함수 전달 불가
<ClientComponent onClick={handleServerClick} />

// ✅ Server Action 사용
async function serverAction() {
  'use server';
  // 서버 로직
}

<ClientComponent action={serverAction} />
```

---

## 의사결정 플로우차트

```
프로젝트에 RSC를 도입해야 할까?
│
├─ Next.js 13+ 사용 중? 
│  ├─ Yes → App Router로 마이그레이션 고려
│  └─ No → 다른 프레임워크면 현재 RSC 지원 제한적
│
├─ 주요 목표가 뭔가?
│  ├─ SEO 개선 → RSC 적합 ✅
│  ├─ 번들 크기 감소 → RSC 적합 ✅
│  ├─ 서버 부하 감소 → CSR이 나을 수도 ⚠️
│  └─ 복잡한 인터랙션 → Client Component 중심 ⚠️
│
├─ 팀 역량은?
│  ├─ React 경험 풍부 → 학습 곡선 감당 가능
│  ├─ 신규 팀 → 전통적 SSR부터 시작 권장
│  └─ 혼자 개발 → 복잡도 증가 주의
│
└─ 결정!
   ├─ 단계적 도입 (추천) → 새 페이지부터 RSC 적용
   ├─ 전면 도입 → 리스크 높음, 충분한 테스트 필요
   └─ 도입 보류 → 생태계가 더 성숙할 때까지 대기
```

---

## 핵심 요약

### RSC의 진실

**장점**:
- ✅ JavaScript 번들 크기 18~29% 감소
- ✅ 서버 리소스 직접 접근 (DB, 파일 시스템)
- ✅ SEO 친화적
- ✅ 민감한 로직을 클라이언트에 노출하지 않음

**단점**:
- ❌ 학습 곡선 가파름 (Server/Client 구분)
- ❌ 생태계 미성숙 (많은 라이브러리가 아직 미지원)
- ❌ 디버깅 어려움 (코드 실행 위치가 혼란스러움)
- ❌ 보안 취약점 리스크 (2025년 12월 사례)

### 실무 적용 기준

| 프로젝트 유형 | RSC 적합도 | 추천 전략 |
|--------------|-----------|----------|
| 블로그, CMS | ⭐⭐⭐⭐⭐ | 전면 도입 |
| E-commerce | ⭐⭐⭐⭐ | 하이브리드 (상품 페이지는 Server, 장바구니는 Client) |
| 대시보드 | ⭐⭐⭐ | 리포트는 Server, 실시간 차트는 Client |
| SaaS Admin | ⭐⭐⭐ | 읽기는 Server, 쓰기 폼은 Client |
| 실시간 앱 | ⭐⭐ | Client 중심, Server는 초기 데이터만 |
| 게임, 에디터 | ⭐ | CSR 권장 |

### 2025년 현재 상태

- **프로덕션 준비도**: ⭐⭐⭐⭐ (4/5)
- **생태계 성숙도**: ⭐⭐⭐ (3/5)
- **개발자 경험**: ⭐⭐⭐ (3/5)
- **보안 안정성**: ⭐⭐⭐ (3/5) - 최근 취약점 패치됨

### 시작하기 전 체크리스트

- [ ] 팀이 React 18+ 개념을 이해하고 있는가?
- [ ] Next.js 15+ 또는 RSC 지원 프레임워크 사용 가능한가?
- [ ] 주요 의존성 라이브러리가 RSC를 지원하는가?
- [ ] 보안 업데이트 프로세스가 마련되어 있는가?
- [ ] 점진적 마이그레이션 계획이 있는가?
- [ ] 롤백 계획이 있는가?

---

## 참고 자료

- [React 공식 문서 - Server Components](https://react.dev/reference/rsc/server-components)
- [Next.js App Router 문서](https://nextjs.org/docs/app)
- [Josh Comeau - Making Sense of React Server Components](https://www.joshwcomeau.com/react/server-components/)
- [RSC 보안 취약점 공지](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)

---

**작성일**: 2026년 1월  
**대상 독자**: React 중급 이상 개발자  
**검증 환경**: Next.js 15.1, React 19.0.3

---

**면책**: RSC는 빠르게 진화하고 있는 기술입니다. 실무 도입 시 최신 문서를 반드시 확인하고, 충분한 테스트를 거쳐 적용하시기 바랍니다. 특히 보안 업데이트는 즉시 적용하는 것을 강력히 권장합니다.
