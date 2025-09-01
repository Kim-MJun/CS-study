# Micro Frontend

### 1. Micro Frontend란?

간단히 말해, Micro Service Architecture의 개념을 프론트엔드에 적용한 아키텍처 패턴. 하나의 큰 프론트엔드 애플리케이션을 여러 개의 작은, 독립적인 애플리케이션으로 분할하여 개발하고 배포하는 방식

### 2. 전통적인 방식 vs Micro Frontend

#### 2.1 Monilithic Frontend

```
📁 Frontend Application
 ├── 📁 components/
 │   ├── Header
 │   ├── Navigation
 │   ├── Dashboard
 │   └── Profile
 ├── 📁 services/
 ├── 📁 store/
 └── package.json
```

특징으로는,

- 모든 기능이 하나의 애플리케이션에 포함
- 단일 빌드 및 배포
- 기술 스택 통일

#### 2.2 Micro Frontend

```
🏠 Shell App (Container)
├── 📊 Dashboard App (React)
├── 👤 Profile App (Vue)
├── 💳 Payment App (Angular)
└── 📈 Analytics App (Svelte)
```

특징으로는,

- 기능별로 독립적인 애플리케이션
- 독립적인 빌드 및 배포
- 기술 스택 선택 자유

### 3. 주요 구현 방식

#### 3.1 Module Federation (Webpack 5)

- Host 애플리케이션 설정

```js
// webpack.config.js
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  mode: 'development',
  devServer: {
    port: 3000,
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        dashboard: 'dashboard@http://localhost:3001/remoteEntry.js',
        profile: 'profile@http://localhost:3002/remoteEntry.js',
      },
    }),
  ],
};
```

- Remote 애플레케이션 설정

```js
// dashboard/webpack.config.js
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  mode: 'development',
  devServer: {
    port: 3001,
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'dashboard',
      filename: 'remoteEntry.js',
      exposes: {
        './Dashboard': './src/Dashboard',
      },
    }),
  ],
};
```

### 3.2 Single-SPA

- 라우팅 기반 통합

```
import { registerApplication, start } from 'single-spa';

// 애플리케이션 등록
registerApplication({
  name: 'dashboard',
  app: () => import('./dashboard-app'),
  activeWhen: ['/dashboard'],
});

registerApplication({
  name: 'profile',
  app: () => import('./profile-app'),
  activeWhen: ['/profile'],
});

// 애플리케이션 시작
start();
```

### 3.3 iframe 방식

```html
<!-- 간단하지만 제한적인 통합 방식 -->
<iframe src="http://dashboard.example.com" width="100%" height="500px">
</iframe>
```

### 4. 장단점

#### 4-1. 장점

- 팀 독립성
  - 독립적 개발: 각 팀이 독립적으로 기능 개발
  - 기술 스택 자유도: 팀별로 최적의 기술 선택 가능
  - 배포 독립성: 전체 시스템 중단 없이 부분 배포
- 확장성
  - 수평적 확장: 필요한 기능만 확장 가능
  - 점진적 업그레이드: 레거시 시스템 점진적 교체
  - 리소스 최적화: 사용하지 않는 기능 로드하지 않음
- 장애 격리
  - 부분적 장애: 한 부분의 문제가 전체에 영향을 미치지 않음
  - 안정성 향상: 독립적인 장애 복구

#### 4.2 단점

- 복잡성 증가
  - 통합 복잡성: 여러 애플리케이션 간 통합 복잡
  - 배포 관리: 여러 배포 파이프라인 관리 필요
  - 디버깅 어려움: 분산된 시스템 디버깅 어려움
- 성능 이슈
  - 중복 의존성: 공통 라이브러리 중복 로드
  - 번들 크기: 전체 번들 크기 증가 가능성
  - 네트워크 요청: 추가 네트워크 요청으로 인한 지연
- 사용자 경험
  - 일관성 유지: UI/UX 일관성 유지 어려움
  - 상태 공유: 애플리케이션 간 상태 공유 복잡성

### 5. 실제 구현 예시

#### 5.1 Container App

```js
import React, { Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// 동적 import를 통한 마이크로 프론트엔드 로드
const Dashboard = React.lazy(() => import('dashboard/Dashboard'));
const Profile = React.lazy(() => import('profile/Profile'));

function App() {
  return (
    <BrowserRouter>
      <div>
        <header>
          <h1>My Application</h1>
        </header>

        <Suspense fallback={<div>Loading...</div>}>
          <Routes>
            <Route path='/dashboard/*' element={<Dashboard />} />
            <Route path='/profile/*' element={<Profile />} />
          </Routes>
        </Suspense>
      </div>
    </BrowserRouter>
  );
}

export default App;
```

#### 5.2 공통 컴포넌트 공유

```js
// shared-components/Button.js
export const Button = ({ children, onClick, variant }) => {
  return (
    <button className={`btn btn-${variant}`} onClick={onClick}>
      {children}
    </button>
  );
};

// 각 마이크로 프론트엔드에서 사용
import { Button } from 'shared-components/Button';
```

### 6. 도입 고려사항

- 팀 구조
  - 충분한 팀 규모: 여러 독립적인 팀 보유
  - 도메인 분리: 명확한 비지니스 도메인 구분
  - 조직 구조: Conway's Law 고려
- 기술적 요구사항
  - 다양한 기술 스택: 서로 다른 기술 사용 필요성
  - 독립적 배포: 부분적 배포의 필요성
  - 확장성 요구: 수평적 확장 필요
- 비지니스 요구사항
  - 빠른 개발: 병렬 개발을 통한 속도 향상
  - 유연성: 기능별 독립적 변경 필요
  - 실험적 접근: A/B 테스트나 점진적 롤아웃

### 7. 모범 사례

- 디자인 시스템
  - 공통 UI 컴포넌트 라이브러리 구축
  - 일관된 브랜딩 가이드라인
  - 접근성 표준 준수
- 성능 최적화
  - 코드 분할: 필요한 부분만 로드
  - 캐싱 전략: 공통 의존성 캐싱
  - 지연 로딩: 필요시점에 로드
- 모니터링
  - 통합 로깅: 분산된 로그 통합 관리
  - 에러 추적: 마이크로 프론트엔드별 에러 모니터링
  - 성능 메트릭: 각 앱의 성능 지표 추적

### 8. 결론

- Micro Frontend는 대규모 팀과 복잡한 애플리케이션에서 개발 효율성과 확장성을 크게 향상시킬 수 있음
- 추가적인 복잡성과 관리 오버헤드 발생 등, 조직 규모와 요구사항을 신중히 고려하여 도입 여부 결정해야 됨
