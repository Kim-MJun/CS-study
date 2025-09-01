# React 렌더링 방식

## 웹 브라우저 동작은?

#### <b>Critical Rendering Path</b> 과정을 거쳐 렌더링 됨.

![브라우저_렌더링_방식](./img/browser_render.png)

- 1단계. HTML, CSS 변환
  - HTML -> DOM
  - CSS -> CSSOM
- 2단계. Render Tree 생성
  - DOM, CSSOM 을 이용해 Render Tree를 생성
- 3단계. Layout
  - Render Tree를 기반으로 실제 웹 페이지에 요소들의 배치를 결정하는 작업
- 4단계 - Painting
  - 실제로 요소들을 화면에 그려내는 과정

#### 업데이트 발생 과정

```
javascript가 DOM을 수정하게 되면서 업데이트가 발생 함
즉, DOM이 수정되면 Critical Rendering Path가 다시 실행 됨
```

특징으로는, Layout(Reflow)과 Painting(Repaint)을 다시 하는건 매우 비싼 과정이다.

---

## React 렌더링 프로세스

React는 2단계를 거쳐 화면에 UI를 렌저링 함

1. Render Phase

- 컴포넌트를 계산하고 업데이트 사항을 파악하는 단계
- 1. 컴포넌트를 호출해 결과값을 계산한 걸 `React Element` 로 저장한다.
- 2. React Element들을 모아 `Virtual DOM`을 생성

2. Commit Phase

- 변경사항을 실제 DOM에 반영하는 단계
- 1. Virtual DOM -> Actual DOM 에다 반영
- 2. Actual DOM이 변경됨을 감지하면 Critical Rendering Path 작동

#### 업데이트 발생 시

1. Render Phase를 처음부터 다시 실행 -> 새로운 Virtual DOM 생성
2. Next Virtual DOM <-> Prev Virtual DOM의 차이점 비교
3. 계산된 차이점을 Actual DOM에 한번에 업데이트

위 방식대로 진행하는 이유는 `DOM 수정을 최소화 하기 위해서` 가상 DOM을 적극 사용중

---

## 최종 정리

- 순수 Javascript만 이용해 DOM을 조작할 때에는 DOM 수정을 최소화 해야 함
  - Reflow, Repaint를 최소한으로 발생시키기 위함
  - 동시에 발생하는 업데이트를 최대한 모아 한번만 DOM을 수정해야 함
  - 서비스 규모가 커질수록 쉽지 않음
- React는 자체적인 렌더링 프로세스를 사용하므로 이런 걱정에서 자유로움
  - Render Phase와 Commit Phase로 나뉨
  - Render Phase에 Virtual DOM을 생성하여 동시에 발생하는 업데이트를 모음
  - Commit Phase에 Virtual DOM에 반영된 모든 업데이트를 Actual DOM에 한번만 반영함
