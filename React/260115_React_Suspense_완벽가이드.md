# React Suspense 완벽 가이드: 개념부터 실무 적용까지

## 목차

1. [React Suspense란?](#react-suspense란)
2. [Suspense 이전 vs 이후](#suspense-이전-vs-이후)
3. [Props와 기본 사용법](#props와-기본-사용법)
4. [Waterfall 문제의 진실](#waterfall-문제의-진실)
5. [React 19의 중요한 변경사항](#react-19의-중요한-변경사항)
6. [실무 적용 가이드](#실무-적용-가이드)

---

## React Suspense란?

React Suspense는 **비동기 작업(데이터 페칭, 코드 스플리팅)의 로딩 상태를 선언적으로 관리**할 수 있게 해주는 React 컴포넌트입니다.

```jsx
<Suspense fallback={<LoadingSkeleton />}>
  <UserProfile />
</Suspense>
```

### 핵심 개념

- 자식 컴포넌트가 렌더링 중 일시 중단(suspend)되면, 데이터가 준비될 때까지 fallback UI를 표시
- useState, useEffect 없이 로딩 상태를 관리
- React 18부터 SWR, React Query 같은 라이브러리와 함께 사용 가능
- React 19에서는 데이터 페칭을 공식 지원

---

## Suspense 이전 vs 이후

### Before: 전통적인 방식

```jsx
function UserProfile() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchUserData()
      .then((data) => {
        setData(data);
        setLoading(false);
      })
      .catch((err) => {
        setError(err);
        setLoading(false);
      });
  }, []);

  if (loading) return <Spinner />;
  if (error) return <Error message={error} />;
  return <div>{data.name}</div>;
}
```

**문제점:**

- 매번 loading, error 상태를 반복적으로 관리
- 여러 컴포넌트가 순차적으로 데이터를 요청하면 **Waterfall 발생** (총 7초+)
- 보일러플레이트 코드 과다

### After: Suspense 방식

```jsx
<Suspense fallback={<LoadingSkeleton />}>
  <UserProfile />
</Suspense>;

function UserProfile() {
  const data = useUserData(); // 내부적으로 suspend
  return <div>{data.name}</div>;
}
```

**개선점:**

- ✅ 코드 간결성: 보일러플레이트 코드 최대 70% 감소
- ✅ 선언적 로딩 처리: useState, useEffect 불필요
- ✅ 더 나은 UX: 스켈레톤 화면, 시머 등으로 체감 성능 향상
- ✅ 에러 처리 통합: Error Boundary와 조합

---

## Props와 기본 사용법

Suspense는 **단 2개의 props**만 받습니다.

### 1. children (필수)

렌더링하려는 실제 UI. children이 suspend되면 fallback으로 전환됩니다.

### 2. fallback (필수)

로딩 중 표시할 대체 UI. 컴포넌트, JSX, 텍스트 모두 가능합니다.

```jsx
// 간단한 텍스트
<Suspense fallback={<div>Loading...</div>}>
  <Albums />
</Suspense>

// 컴포넌트
<Suspense fallback={<LoadingSkeleton />}>
  <Albums />
</Suspense>

// 복잡한 JSX
<Suspense fallback={
  <div className="loading-container">
    <Spinner />
    <p>데이터를 불러오는 중입니다...</p>
  </div>
}>
  <Albums />
</Suspense>
```

### 중첩된 Suspense로 세밀한 제어

```jsx
<Suspense fallback={<PageSkeleton />}>
  <Header />

  <Suspense fallback={<PostsSkeleton />}>
    <Posts />
  </Suspense>

  <Suspense fallback={<CommentsSkeleton />}>
    <Comments />
  </Suspense>
</Suspense>
```

---

## Waterfall 문제의 진실

### 오해: Suspense만 추가하면 Waterfall이 해결된다? ❌

```jsx
// 이렇게만 하면 여전히 7초 걸립니다!
<Suspense fallback={<Loading />}>
  <Profile /> {/* 3초 */}
  <Posts /> {/* 2초 */}
  <Comments /> {/* 2초 */}
</Suspense>
```

**핵심은 데이터를 언제, 어떻게 페칭하느냐입니다.**

### Fetch-on-render vs Render-as-you-fetch

```jsx
// ❌ Fetch-on-render: 렌더링 시점에 fetch
function Profile() {
  useEffect(() => {
    fetchProfile().then(setData); // 렌더링 후 시작
  }, []);
}

// ✅ Render-as-you-fetch: 렌더링 전에 fetch 시작
const profilePromise = fetchProfile(); // 미리 시작!
const postsPromise = fetchPosts();
const commentsPromise = fetchComments();

function Page() {
  return (
    <Suspense fallback={<Loading />}>
      <Profile promise={profilePromise} />
      <Posts promise={postsPromise} />
      <Comments promise={commentsPromise} />
    </Suspense>
  );
}
```

### React Query 사용 시 주의사항

```jsx
// ❌ 이렇게 하면 Waterfall 발생
function BadExample() {
  const profile = useSuspenseQuery(['profile']);
  const posts = useSuspenseQuery(['posts']); // profile 이후 시작
  const comments = useSuspenseQuery(['comments']); // posts 이후 시작
}

// ✅ 해결 방법 1: 컴포넌트 분리
<Suspense fallback={<Loading />}>
  <ProfileComponent />
  <PostsComponent />
  <CommentsComponent />
</Suspense>;

// ✅ 해결 방법 2: useSuspenseQueries 사용
function GoodExample() {
  const [profile, posts, comments] = useSuspenseQueries([
    { queryKey: ['profile'], queryFn: fetchProfile },
    { queryKey: ['posts'], queryFn: fetchPosts },
    { queryKey: ['comments'], queryFn: fetchComments },
  ]);
  // 모두 병렬로 페칭됨!
}
```

---

## React 19의 중요한 변경사항

### 💥 Breaking Change: 형제 컴포넌트 병렬 렌더링 제거

#### React 18: 형제 컴포넌트를 Pre-rendering

```jsx
// React 18
<Suspense fallback={<Loading />}>
  <Profile /> {/* suspend → 하지만 계속 진행 */}
  <Posts /> {/* 이것도 렌더링 시도 → fetch 시작! */}
  <Comments /> {/* 이것도 렌더링 시도 → fetch 시작! */}
</Suspense>
// 결과: 3개 모두 병렬 페칭 → 총 3초
```

#### React 19: 첫 suspend에서 즉시 중단

```jsx
// React 19
<Suspense fallback={<Loading />}>
  <Profile /> {/* suspend → 즉시 중단! */}
  <Posts /> {/* 렌더링 안됨 = fetch 안 시작 */}
  <Comments /> {/* 렌더링 안됨 = fetch 안 시작 */}
</Suspense>
// 결과: 순차 페칭 → 총 7초 (Waterfall!)
```

### ⚠️ 왜 이렇게 바뀌었나?

React 팀의 입장:

- **즉각적인 로딩 상태** 표시를 우선시
- Fetch-on-render는 나쁜 패턴이므로 권장하지 않음
- Best practice를 따르면 오히려 더 빠름 (데이터 호이스팅)

### 커뮤니티 반응

- 실제 앱에서 2.5초 → 3.5초로 성능 저하 사례 발생
- react-three-fiber 팀이 React 포크까지 논의
- React 18에서 잘 작동하던 앱이 React 19에서 워터폴 발생

---

## 실무 적용 가이드

### 1. 독립적인 Suspense Boundary (가장 간단)

```jsx
// React 19에서도 병렬 로딩
<>
  <Suspense fallback={<ProfileSkeleton />}>
    <Profile />
  </Suspense>
  <Suspense fallback={<PostsSkeleton />}>
    <Posts />
  </Suspense>
  <Suspense fallback={<CommentsSkeleton />}>
    <Comments />
  </Suspense>
</>
```

**장점:**

- 각 컴포넌트가 독립적으로 데이터 관리
- 부모가 자식의 데이터를 알 필요 없음
- 컴포넌트 재사용성 높음

**단점:**

- "Popcorn UI" 가능성 (순차적으로 팝업)

### 2. 데이터 호이스팅 (React 팀 권장)

렌더링 전에 fetch를 미리 시작해 Promise를 준비해두는 방식입니다.

```jsx
// Router 레벨에서 Prefetch
export async function loader() {
  await Promise.all([
    queryClient.ensureQueryData(['profile'], fetchProfile),
    queryClient.ensureQueryData(['posts'], fetchPosts),
    queryClient.ensureQueryData(['comments'], fetchComments),
  ]);
  return null;
}

// 컴포넌트에서는 이미 캐시된 데이터 사용
function Page() {
  const profile = useSuspenseQuery(['profile'], fetchProfile);
  const posts = useSuspenseQuery(['posts'], fetchPosts);
  const comments = useSuspenseQuery(['comments'], fetchComments);
}
```

**장점:**

- React 19에서도 병렬 fetch 가능
- 워터폴 완전 회피
- 가장 빠른 로딩 시간

**단점:**

- 부모가 자식의 데이터 요구사항을 알아야 함 (캡슐화 약화)
- 코드 복잡도 증가

### 3. Server Components (Next.js App Router)

```jsx
// app/page.tsx
async function Page() {
  // 서버에서 병렬 fetch (latency 낮음)
  const [profile, posts, comments] = await Promise.all([
    fetchProfile(),
    fetchPosts(),
    fetchComments(),
  ]);

  return (
    <>
      <Profile data={profile} />
      <Posts data={posts} />
      <Comments data={comments} />
    </>
  );
}
```

### 4. React 19의 `use` hook

```jsx
function Page() {
  // 렌더링 전에 Promise 시작
  const profilePromise = fetchProfile();
  const postsPromise = fetchPosts();

  return (
    <Suspense fallback={<Loading />}>
      <ProfileComponent promise={profilePromise} />
      <PostsComponent promise={postsPromise} />
    </Suspense>
  );
}

function ProfileComponent({ promise }) {
  const data = use(promise); // Promise unwrap
  return <div>{data.name}</div>;
}
```

---

## 실무 의사결정 체크리스트

### Suspense를 언제 도입해야 할까?

**추천하는 경우:**

- ✅ 여러 컴포넌트가 독립적으로 데이터를 페칭할 때
- ✅ 스켈레톤 UI를 체계적으로 관리하고 싶을 때
- ✅ React Query, SWR 등 Suspense 지원 라이브러리를 사용할 때
- ✅ Next.js App Router (Server Components) 사용 가능할 때

**주의가 필요한 경우:**

- ⚠️ React 19로 마이그레이션 시 (병렬 렌더링 변경사항 확인 필요)
- ⚠️ SEO가 중요한 SSR 앱 (hydration 이슈 테스트 필요)
- ⚠️ 깊이 중첩된 컴포넌트 구조

### 패턴 선택 가이드

| 상황                              | 추천 패턴                               |
| --------------------------------- | --------------------------------------- |
| 간단한 페이지, 독립적인 섹션      | 독립적인 Suspense Boundary              |
| Router 기반 앱                    | 데이터 호이스팅 (Prefetch)              |
| Next.js App Router                | Server Components                       |
| 재사용 가능한 컴포넌트 라이브러리 | 독립적인 Suspense + 컴포넌트 자체 fetch |

---

## 핵심 요약

### Suspense의 본질

- Suspense는 **로딩 상태를 선언적으로 관리하는 도구**
- Waterfall을 자동으로 해결해주지 **않음**
- 올바른 데이터 페칭 패턴과 함께 사용해야 효과적

### React 19 마이그레이션 체크포인트

1. 같은 Suspense boundary 내 형제 컴포넌트들이 각자 fetch하는가? → Waterfall 발생 가능
2. 독립적인 Suspense boundary로 분리하거나 데이터 호이스팅 적용
3. 성능 측정 후 마이그레이션 진행

### 실무 Best Practices

- Fallback은 가볍고 의미 있게 작성
- 중첩된 Suspense로 세밀한 로딩 제어
- Error Boundary와 함께 사용하여 에러 처리
- 개발자 도구로 Suspense boundary 동작 모니터링

---

## 참고 자료

- [React 공식 문서 - Suspense](https://react.dev/reference/react/Suspense)
- [TkDodo's Blog - React 19 and Suspense](https://tkdodo.eu/blog/react-19-and-suspense-a-drama-in-3-acts)
- [React 19 GitHub Issue #29898](https://github.com/facebook/react/issues/29898)
- [TanStack Query - Suspense](https://tanstack.com/query/latest/docs/framework/react/guides/suspense)
