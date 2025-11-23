# Phase 2-1: 문제 재풀이(복습) 시스템 - 앱 구현 기획서

## 목차

1. [개요](#1-개요)
2. [UI/UX 설계](#2-uiux-설계)
3. [API 연동](#3-api-연동)
4. [화면별 구현 사항](#4-화면별-구현-사항)
5. [타입 정의](#5-타입-정의)
6. [구현 순서](#6-구현-순서)
7. [체크리스트](#7-체크리스트)

---

## 1. 개요

### 1.1. 목표

문제 재풀이(복습) 시스템의 앱 측 UI/UX를 구현하여 사용자에게 다음 정보를 제공:

- 문제 풀이 상태 (정복/취약/미도전)
- 미정복 문제 여부
- 복습 모드 안내 및 예상 XP
- 재풀이 컨텍스트가 반영된 AI 피드백

### 1.2. 핵심 정책 요약

| 항목         | 내용                                        |
| :----------- | :------------------------------------------ |
| **PSR**      | 이미 맞힌 문제는 다시 맞혀도 점수 상승 없음 |
| **XP**       | 복습 회차별 감소 (100% → 30% → 10% → 0%)    |
| **Cooldown** | 24시간 이내 재풀이 시 XP 지급 0%            |

---

## 2. UI/UX 설계

### 2.1. 문제 상태 뱃지

#### A. 풀이 상태 뱃지

| 상태   | 라벨   | 색상/스타일     | 조건                  |
| :----- | :----- | :-------------- | :-------------------- |
| 미도전 | 미도전 | Gray / Outlined | 풀이 기록 없음        |
| 정복   | 정복 ✓ | Green / Filled  | 최근 풀이 결과가 정답 |
| 취약   | 취약 ✗ | Red / Filled    | 최근 풀이 결과가 오답 |

#### B. 미정복 문제 뱃지

| 조건                      | 라벨      | 색상/스타일     |
| :------------------------ | :-------- | :-------------- |
| 문제의 total_correct < 10 | 🔥 미정복 | Orange / Filled |

#### C. 배치 위치

```
┌─────────────────────────────┐
│ [정복 ✓] [🔥 미정복]        │
│                             │
│    문제 내용...             │
└─────────────────────────────┘
```

---

### 2.2. 복습 모드 알림

문제 풀이 화면 진입 시, 재풀이인 경우 상단에 안내 메시지 표시:

```
┌─────────────────────────────┐
│ ℹ️ 복습을 시작합니다!        │
│ 이번엔 30 XP를 획득할 수     │
│ 있어요.                      │
└─────────────────────────────┘
```

#### 메시지 표시 조건

| XP Rate         | 메시지                                                |
| :-------------- | :---------------------------------------------------- |
| 100%            | (표시 안 함 - 최초 풀이)                              |
| 30%             | "복습을 시작합니다! 이번엔 30 XP를 획득할 수 있어요." |
| 10%             | "복습을 시작합니다! 이번엔 10 XP를 획득할 수 있어요." |
| 0% (4회차 이상) | "복습을 시작합니다! 개념 복습에 집중해 보세요."       |
| 0% (쿨타임)     | "너무 빨리 다시 풀었습니다. 잠시 후 복습해 주세요."   |

---

### 2.3. AI 피드백 표시

재풀이 컨텍스트가 반영된 피드백은 서버에서 제공하므로, 앱에서는 기존 피드백 화면 그대로 표시하면 됩니다.

---

## 3. API 연동

### 3.1. 문제 상태 조회 API

**Endpoint**: `GET /problems/:id/status`

**Request Headers**:

```
Authorization: Bearer {access_token}
```

**Response**:

```typescript
{
  problem_id: string;
  solve_status_badge: {
    label: string;      // "미도전" | "정복 ✓" | "취약 ✗"
    color: string;      // "gray" | "green" | "red"
    style: string;      // "outlined" | "filled"
  };
  unconquered_badge: {
    label: string;      // "🔥 미정복"
    color: string;      // "orange"
    style: string;      // "filled"
  } | null;
  retry_info: {
    is_retry: boolean;
    attempt_count: number;
    solve_count: number;
    last_attempted_at: string;
    expected_xp_rate: number;     // 0.0 | 0.1 | 0.3 | 1.0
    cooldown_active: boolean;
  } | null;
}
```

---

### 3.2. API 서비스 메서드 추가

`src/services/api.ts`에 추가:

```typescript
// 문제 상태 조회
async getProblemStatus(problemId: string): Promise<ProblemStatus> {
  const response = await this.get(`/problems/${problemId}/status`);
  return response.data;
}
```

---

## 4. 화면별 구현 사항

### 4.1. 문제 목록 화면 (PageViewerScreen)

#### 변경 사항

1. **문제 렌더링 시 뱃지 표시**
   - 문제 카드 좌상단에 뱃지 2개 표시 (풀이 상태 + 미정복)
2. **데이터 로딩**
   - 문제 목록 조회 시, 각 문제에 대해 상태 정보 배치 조회
   - 성능 최적화: 문제 목록 로딩 시 한 번에 상태 조회 (N+1 방지)

#### 구현 포인트

```typescript
// 문제 목록 로딩 시
const problems = await api.getProblems(workbookId);

// 각 문제의 상태 조회 (배치)
// 옵션 1: 서버에서 문제 목록 API에 상태 정보 포함
// 옵션 2: 클라이언트에서 병렬 요청
const statuses = await Promise.all(
  problems.map(p => api.getProblemStatus(p.id)),
);

// 문제와 상태 정보 매핑
const problemsWithStatus = problems.map((problem, index) => ({
  ...problem,
  status: statuses[index],
}));
```

**참고**: 성능을 위해 서버 API 수정을 고려할 수 있습니다. (문제 목록 조회 시 상태 정보 포함)

---

### 4.2. 문제 풀이 화면 (ProblemSolvingScreen)

#### 변경 사항

1. **화면 진입 시 상태 조회**

   - `GET /problems/:id/status` 호출
   - `retry_info` 확인

2. **복습 모드 알림 표시**

   - `retry_info`가 있으면 상단에 안내 메시지 표시
   - `expected_xp_rate`에 따라 메시지 내용 결정
   - `cooldown_active`이면 경고 메시지 표시

3. **UI 컴포넌트**
   - 토스트 또는 배너 형태로 표시
   - 자동으로 사라지거나 닫기 버튼 제공

#### 구현 포인트

```typescript
useEffect(() => {
  const loadProblemStatus = async () => {
    const status = await api.getProblemStatus(problemId);

    if (status.retry_info) {
      const { expected_xp_rate, cooldown_active } = status.retry_info;

      if (cooldown_active) {
        showToast('너무 빨리 다시 풀었습니다. 잠시 후 복습해 주세요.');
      } else if (expected_xp_rate === 0.3) {
        showToast('복습을 시작합니다! 이번엔 30 XP를 획득할 수 있어요.');
      } else if (expected_xp_rate === 0.1) {
        showToast('복습을 시작합니다! 이번엔 10 XP를 획득할 수 있어요.');
      } else if (expected_xp_rate === 0.0) {
        showToast('복습을 시작합니다! 개념 복습에 집중해 보세요.');
      }
    }
  };

  loadProblemStatus();
}, [problemId]);
```

---

### 4.3. 피드백 화면 (FeedbackScreen)

#### 변경 사항

**없음** - 서버에서 재풀이 컨텍스트가 반영된 피드백을 제공하므로, 기존 피드백 표시 로직 그대로 사용.

---

## 5. 타입 정의

`src/types/index.ts`에 추가:

```typescript
// 문제 상태 조회 응답
export interface ProblemStatus {
  problem_id: string;
  solve_status_badge: StatusBadge;
  unconquered_badge: StatusBadge | null;
  retry_info: RetryInfo | null;
}

export interface StatusBadge {
  label: string;
  color: 'gray' | 'green' | 'red' | 'orange';
  style: 'outlined' | 'filled';
}

export interface RetryInfo {
  is_retry: boolean;
  attempt_count: number;
  solve_count: number;
  last_attempted_at: string;
  expected_xp_rate: number; // 0.0 | 0.1 | 0.3 | 1.0
  cooldown_active: boolean;
}
```

---

## 6. 구현 순서

1. **타입 정의** (`src/types/index.ts`)

   - `ProblemStatus`, `StatusBadge`, `RetryInfo` 추가

2. **API 서비스 메서드** (`src/services/api.ts`)

   - `getProblemStatus()` 추가

3. **공통 컴포넌트 - 뱃지** (`src/components/StatusBadge.tsx`)

   - 풀이 상태 뱃지, 미정복 뱃지 렌더링 컴포넌트

4. **문제 목록 화면** (`src/screens/PageViewer/PageViewerScreen.tsx`)

   - 문제 상태 조회 및 뱃지 표시

5. **문제 풀이 화면** (`src/screens/ProblemSolving/ProblemSolvingScreen.tsx`)

   - 복습 모드 알림 표시

6. **테스트**
   - 각 시나리오별 UI/UX 검증

---

## 7. 체크리스트

### 타입 정의

- [ ] `ProblemStatus` 타입 정의
- [ ] `StatusBadge` 타입 정의
- [ ] `RetryInfo` 타입 정의

### API 연동

- [ ] `api.ts`에 `getProblemStatus()` 메서드 추가

### 공통 컴포넌트

- [ ] `StatusBadge` 컴포넌트 생성
- [ ] 뱃지 스타일링 (색상, outlined/filled)

### 화면 구현

- [ ] **PageViewerScreen**: 문제 상태 조회 로직
- [ ] **PageViewerScreen**: 뱃지 렌더링
- [ ] **ProblemSolvingScreen**: 화면 진입 시 상태 조회
- [ ] **ProblemSolvingScreen**: 복습 모드 알림 표시
- [ ] **ProblemSolvingScreen**: 쿨타임 경고 표시

### 테스트

- [ ] 최초 풀이 → 뱃지 "미도전" 표시
- [ ] 정답 제출 후 → 뱃지 "정복 ✓" 표시
- [ ] 오답 제출 후 → 뱃지 "취약 ✗" 표시
- [ ] 미정복 문제 → "🔥 미정복" 뱃지 표시
- [ ] 재풀이 진입 → 복습 모드 알림 표시 (XP Rate에 따라)
- [ ] 24시간 이내 재풀이 → 쿨타임 경고 표시
- [ ] AI 피드백 컨텍스트 반영 확인

---

## 8. 추가 고려 사항

### 8.1. 성능 최적화

**문제**: 문제 목록에서 각 문제마다 상태 조회 시 N+1 API 호출 발생

**해결 방안**:

1. **클라이언트 캐싱**: 상태 정보를 로컬에 캐싱 (React Query, SWR 등)
2. **서버 API 개선**: 문제 목록 조회 시 상태 정보 함께 반환
   - `GET /workbooks/:id/problems` 응답에 각 문제의 상태 포함
   - 서버에서 배치 조회 로직 활용 (이미 구현됨)

**권장**: 서버 API 개선 (2번) - 더 깔끔하고 성능이 좋음

---

### 8.2. UX 개선 아이디어

1. **복습 추천**

   - 대시보드에서 "복습하면 좋을 문제" 섹션 추가
   - 조건: 정답을 맞혔지만 오래된 문제 (예: 7일 이상 경과)

2. **통계 화면**

   - 전체 문제 수 / 정복한 문제 수 / 취약한 문제 수
   - 복습 필요한 문제 수

3. **필터링**
   - 문제 목록에서 상태별 필터 (미도전/정복/취약)

---

**문서 작성일**: 2025-11-23  
**최종 수정일**: 2025-11-23  
**작성자**: AI Assistant  
**버전**: 1.0
