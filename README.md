
## ✅ 핵심 구현 내용
1. Atom 생성 및 관리
Atom은 상태의 최소 단위로, 특정 값(initialValue)을 담고 있으며 이 값을 읽거나 쓸 수 있도록 구현했습니다.

제공하는 API

createAtom(initialValue: T): Atom<T>
→ 초기값을 기반으로 새로운 atom을 생성하는 함수

get(atom: Atom<T>): T
→ atom의 현재 상태 값을 반환하는 함수

set(atom: Atom<T>, newValue: T): void
→ atom의 값을 newValue로 갱신하는 함수. 값이 변한 경우에만 구독자에게 알림을 전달합니다.

2. 프로바이더 시스템
구독자 관리 및 React 렌더 트리거 추상화 계층을 구현했습니다. 전역 Store나 Context 없이도 atom 단위로 구독자를 관리할 수 있습니다.

구현한 역할

각 atom은 자체적으로 구독자 목록(Set)을 관리합니다 (Redux의 Provider처럼 전역이 아닌, atom-local 스코프)

useAtom에서 렌더 트리거를 구독자로 등록하여 컴포넌트 리렌더링이 가능하도록 구현했습니다

unsubscribe 시 내부 메모리도 정리되도록 구현하여 GC-friendly하게 동작합니다

3. 구독 (subscribe) 시스템
구독 API

subscribe(atom: Atom<T>, callback: (val: T) => void): () => void
→ callback을 구독자로 등록합니다. 값이 바뀌면 실행됩니다.
→ 반환되는 함수는 unsubscribe 역할을 수행합니다.

구독 동작 방식

set 호출 시 값이 이전 상태와 다를 경우에만 구독자에게 알림을 전달합니다

동일 값으로 set하면 callback 호출 없이 무시되도록 구현했습니다

4. React 연동
React Hook API

useAtom(atom: Atom<T>): [T, (val: T) => void]
→ 상태값을 읽고, setter를 통해 값을 변경할 수 있는 Hook입니다

## 렌더링 동작

atom의 값이 변경되면 해당 atom을 사용하는 컴포넌트만 리렌더링되도록 구현했습니다

useSyncExternalStore를 활용하여 구독/해제를 관리합니다

---
## ✨ 추가로 구현한 고급 기능

핵심 기능을 구현한 후 추가로 학습하고 구현한 고급 기능들입니다.

구현한 기능

파생 Atom (Derived Atom)
createDerivedAtom(get => get(atomA) + get(atomB)) 형식으로 계산된 상태를 생성할 수 있도록 구현했습니다

비동기 Atom (Async Atom)
createAsyncAtom(Promise) 기반으로 비동기 로직을 처리할 수 있도록 구현했습니다. Suspense와도 대응 가능합니다

배치 업데이트 (Batch Update)
set(atomA, 1); set(atomB, 2); 같은 연속된 업데이트를 일괄 처리하여 useAtom이 한 번만 리렌더되도록 최적화했습니다

메모리 관리
마지막 구독자가 unsubscribe하면 atom의 내부 리스너 Set도 GC 대상에 포함되도록 메모리 관리를 구현했습니다

Frameworkless 사용
React 외 환경에서도 get, set, subscribe만으로 상태 관리가 가능하도록 설계했습니다

---

# 원자 상태 관리 라이브러리 (Atom State Management Library)

React를 위한 간단하고 효율적인 원자 상태 관리 라이브러리입니다.  
성능 최적화와 React의 Concurrent Features를 고려하여 설계했습니다.

## 🚀 주요 기능

- **원자 상태 (Atom)**: 최소 단위의 상태 관리
- **파생 상태 (Derived Atom)**: 다른 상태로부터 계산되는 상태
- **비동기 상태 (Async Atom)**: Suspense와 Error Boundary 지원
- **자동 배치 업데이트**: 마이크로태스크를 활용한 성능 최적화
- **React 18 호환**: useSyncExternalStore 활용

## 📦 설치 및 사용

```bash
pnpm install
pnpm dev
```

## 🏗️ 원자 상태(Atom) 시스템 구조

### 전체 아키텍처

```js
┌──────────────────────────────────────────────────────────────┐
│                    Atom State System                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│    ┌─────────────┐      ┌─────────────┐    ┌─────────────┐   │
│    │    Atom     │      │ DerivedAtom │    │  AsyncAtom  │   │
│    └─────────────┘      └─────────────┘    └─────────────┘   │
│            │                  │                   │          │
│            └──────────────────┼───────────────────┘          │
│                               │                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Schedule Update System                     │ │
│  │  ┌─────────────────┐  ┌────────────────────────────────┐│ │
│  │  │ pendingUpdates  │  │     Microtask Queue            ││ │
│  │  │   (Set<fn>)     │  │     Promise.resolve()          ││ │
│  │  └─────────────────┘  └────────────────────────────────┘│ │
│  └─────────────────────────────────────────────────────────┘ │
│                              │                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                      React 연동                          │ │
│  │  ┌─────────────────┐  ┌────────────────────────────────┐│ │
│  │  │    useAtom      │  │    useSyncExternalStore        ││ │
│  │  │     (Hook)      │  │      (React 18)                ││ │
│  │  └─────────────────┘  └────────────────────────────────┘│ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 핵심 구성 요소

#### 1. Atom Interface

```typescript
interface Atom<T> {
  _getValue: () => T;
  _setValue: (newValue: T) => void;
  _subscribe: (listener: () => void) => () => void;
}
```

#### 2. 상태 흐름

```
사용자 액션 → set(atom, value) → atom._setValue() → scheduleUpdate() 
→ Batch Update (Microtask Queue) → 구독자 리렌더
```

## 🎯 라이브러리 설계 고려사항과 기술적 선택

### 1. 성능 최적화 우선 설계

#### 자동 배치 업데이트 시스템

- **문제**: 동기적 상태 업데이트로 인한 과도한 리렌더링  
- **해결책**: 마이크로태스크 큐를 활용한 배치 처리  
- **선택 이유**: 우선순위가 높은 `Promise.resolve()` 사용  

```typescript
// 리렌더링을 배치 처리하여 성능을 최적화하는 핵심 함수
const scheduleUpdate = (listener: () => void) => {
  pendingUpdates.add(listener);

  if (!isFlushingUpdates) {
    isFlushingUpdates = true;
    Promise.resolve().then(() => {
      const listeners = Array.from(pendingUpdates);
      pendingUpdates.clear();
      isFlushingUpdates = false;
      listeners.forEach(listener => listener());
    });
  }
};
```

#### 메모리 효율성
- **Set 자료구조**: 중복 리스너 제거  
- **약한 참조 패턴**: 구독 해제 시 자동 메모리 정리  
- **지연 계산**: 파생 상태 필요 시점에만 계산 수행  

### 2. useAtom의 React 18 호환성 고려

#### useSyncExternalStore 채택
- **이전 방식**: `useState` + `useEffect` 조합 (tearing 문제 발생)  
- **현재 방식**: React 18의 `useSyncExternalStore` 활용  
- **장점**: Concurrent Rendering 호환, 티어링 현상 방지  

```typescript
export const useAtom = <T>(atom: Atom<T>) => {
  const value = useSyncExternalStore(
    atom._subscribe,
    atom._getValue
  );
  return value;
};
```

### 3. 타입 안전성과 개발자 경험
- 내부 API는 `_` 접두사로 은닉  
- `get`, `set`, `subscribe`, `useAtom`만 공개  
- TypeScript 기반으로 컴파일 타임 안전성 보장  

### 4. 확장성 고려 설계
- **기본 Atom** → 핵심 상태 관리  
- **파생 Atom** → 계산된 상태 관리  
- **비동기 Atom** → Suspense/에러 처리 지원  

---

## 🔍 원자 상태의 값 비교 방식과 최적화
- `Object.is()`로 정확한 동등성 검사  
- 중복 업데이트 방지 (Set 자료구조 활용)  
- 파생 상태는 지연 평가(Lazy Evaluation)  
- 구독 해제 시 자동 메모리 정리  

---

## ⚛️ React와 연동 시 발생한 문제 및 해결 방법
- 기존 방식(`useState`+`useEffect`) → tearing, 동기화 불일치 문제 발생  
- `useSyncExternalStore`로 해결 → Concurrent Rendering 대응 및 tearing 방지  

---


