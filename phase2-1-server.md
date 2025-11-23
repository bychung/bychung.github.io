# Phase 2-1: 문제 재풀이(복습) 시스템 구현 기획서

## 목차

1. [개요](#1-개요)
2. [DB 스키마 설계](#2-db-스키마-설계)
3. [서버 구현 사항](#3-서버-구현-사항)
4. [API 명세](#4-api-명세)
5. [마이그레이션 파일](#5-마이그레이션-파일)
6. [추가 고려 사항](#6-추가-고려-사항)
7. [참고 자료](#7-참고-자료)
8. [체크리스트](#8-체크리스트)
9. [핵심 타입 정의](#9-핵심-타입-정의)
10. [FAQ](#10-faq)
11. [구현 순서](#11-구현-순서)

---

## 1. 개요

### 1.1. 목표

사용자가 이미 푼 문제를 다시 풀었을 때(Re-solving), 학습 효과(복습)는 독려하되 무의미한 반복(Abusing)을 통한 점수 파밍을 방지하는 시스템 구축.

### 1.2. 설계 철학

**Single Source of Truth**: 별도의 집계 테이블 없이 `submissions` + `feedbacks` 테이블만으로 모든 풀이 이력을 관리합니다.

**이유**:

- ✅ **데이터 일관성**: 동기화 로직 불필요, 버그 발생 가능성 감소
- ✅ **유지보수성**: 집계 로직 변경이 쉬움
- ✅ **단순성**: 추가 테이블 없이 기존 구조 활용

**성능 최적화**: 인덱스 + 배치 조회 + Redis 캐싱

### 1.3. 핵심 정책 요약

| 구분         | 정책                                                                          |
| :----------- | :---------------------------------------------------------------------------- |
| **PSR**      | 이미 맞힌 문제는 다시 맞혀도 점수 상승 없음. 틀리면 하락, 오답을 맞히면 상승. |
| **XP**       | 복습 회차별로 감소 (100% → 30% → 10% → 0%)                                    |
| **Cooldown** | 24시간 이내 재풀이 시 XP 지급 0%                                              |

---

## 2. DB 스키마 설계

### 2.1. 설계 철학: Single Source of Truth

**핵심 원칙**: `submissions` + `feedbacks` 테이블만으로 모든 풀이 이력 정보를 관리합니다.

#### 왜 별도 테이블을 만들지 않나요?

1. **데이터 일관성**: 단일 진실 공급원(Single Source of Truth) 유지
2. **유지보수성**: 동기화 로직 불필요, 버그 발생 가능성 감소
3. **유연성**: 집계 로직 변경이 쉬움
4. **YAGNI 원칙**: 실제 성능 문제 발생 전까지 오버엔지니어링 방지

#### 풀이 이력 정보를 얻는 방법

```sql
-- submissions + feedbacks 테이블에서 집계
SELECT
  s.problem_id,
  COUNT(*) as total_attempts,
  COUNT(*) FILTER (WHERE f.is_correct) as correct_attempts,
  COUNT(*) FILTER (WHERE f.is_correct) as solve_count,
  MAX(s.submitted_at) as last_attempted_at,
  MIN(s.submitted_at) FILTER (WHERE f.is_correct) as first_correct_at,
  (array_agg(f.is_correct ORDER BY s.submitted_at DESC))[1] as last_result
FROM submissions s
LEFT JOIN feedbacks f ON s.id = f.submission_id
WHERE s.user_id = $1 AND s.problem_id = $2
GROUP BY s.problem_id;
```

---

### 2.2. 기존 테이블 변경 사항

#### 2.2.1. `submissions` 테이블

```sql
ALTER TABLE submissions
ADD COLUMN is_retry BOOLEAN DEFAULT FALSE,              -- 재풀이 여부
ADD COLUMN attempt_count INT DEFAULT 1,                 -- N번째 시도
ADD COLUMN solve_count_at_submission INT DEFAULT 0,     -- 제출 당시 정답 제출 횟수
ADD COLUMN xp_multiplier DECIMAL(3, 2) DEFAULT 1.00;    -- XP 배율 (1.00 / 0.30 / 0.10 / 0.00)
```

#### 컬럼 설명

- `is_retry`: 이 제출이 재풀이인지 여부 (최초 풀이 = false, 재풀이 = true)
- `attempt_count`: 해당 문제를 몇 번째 시도하는 것인지 (1, 2, 3, ...)
- `solve_count_at_submission`: **제출 당시** 정답을 제출한 횟수 (XP 배율 계산용, 캐싱)
- `xp_multiplier`: 적용된 XP 배율 (1.00 = 100%, 0.30 = 30%, 0.10 = 10%, 0.00 = 0%)

#### 인덱스 추가

```sql
-- 사용자별 문제별 풀이 이력 조회 최적화
CREATE INDEX idx_submissions_user_problem_time
ON submissions(user_id, problem_id, submitted_at DESC);

-- 문제 풀이 상태 배치 조회 최적화 (커버링 인덱스)
CREATE INDEX idx_submissions_user_status
ON submissions(user_id, problem_id, submitted_at DESC)
INCLUDE (id, status);
```

---

## 3. 서버 구현 사항

### 3.1. 아키텍처 개요

```
submissions.service.ts (제출 진입점)
  ↓
  1. 풀이 이력 조회 (problem_attempts)
  2. 재풀이 여부 판단
  3. XP 배율 결정 (회차, 쿨타임)
  ↓
submissions.service.ts → runAIPipeline()
  ↓ AI 분석 완료 후
  ↓
gamification.service.ts → processGamification()
  ├─ calculateAndAwardXP() → 배율 적용
  ├─ updatePSR() → 이전 상태 고려
  └─ 풀이 이력 업데이트 (problem_attempts)
```

---

### 3.2. `submissions.service.ts` 수정

#### 3.2.1. `createSubmission()` 로직 추가

```typescript
async createSubmission(
  userId: string,
  createSubmissionDto: CreateSubmissionDto
) {
  const supabase = this.supabaseService.getAdminClient();
  const problemId = createSubmissionDto.problem_id;

  // ========== 1. 풀이 이력 조회 (submissions 기반) ==========
  const attemptHistory = await this.getProblemAttemptHistory(userId, problemId);

  let isRetry = false;
  let attemptCount = 1;
  let solveCountAtSubmission = 0;
  let xpMultiplier = 1.0;

  if (attemptHistory) {
    // 재풀이
    isRetry = true;
    attemptCount = attemptHistory.total_attempts + 1;
    solveCountAtSubmission = attemptHistory.solve_count;

    // ========== 2. XP 배율 결정 ==========
    // (1) 쿨타임 체크 (24시간 이내면 배율 0)
    const lastAttemptTime = new Date(attemptHistory.last_attempted_at);
    const now = new Date();
    const hoursSinceLastAttempt =
      (now.getTime() - lastAttemptTime.getTime()) / (1000 * 60 * 60);

    if (hoursSinceLastAttempt < 24) {
      xpMultiplier = 0.0;
    } else {
      // (2) 회차별 배율 (solve_count 기준)
      const solveCount = attemptHistory.solve_count;
      if (solveCount === 0) {
        xpMultiplier = 1.0;  // 최초 정답
      } else if (solveCount === 1) {
        xpMultiplier = 0.3;  // 2회차
      } else if (solveCount === 2) {
        xpMultiplier = 0.1;  // 3회차
      } else {
        xpMultiplier = 0.0;  // 4회차 이상
      }
    }
  }

  // ========== 3. Submission 생성 (기존 로직 + 추가 필드) ==========
  const { data: submission, error: submissionError } = await supabase
    .from('submissions')
    .insert({
      id: submissionId,
      user_id: userId,
      problem_id: problemId,
      solution_image_url: solutionImageUrl,
      user_answer: createSubmissionDto.user_answer,
      started_at: createSubmissionDto.started_at,
      status: 'pending',

      // 추가 필드
      is_retry: isRetry,
      attempt_count: attemptCount,
      solve_count_at_submission: solveCountAtSubmission,
      xp_multiplier: xpMultiplier,
    })
    .select()
    .single();

  // ... (이미지 업로드, AI 파이프라인 트리거 등 기존 로직)
}

/**
 * 문제별 사용자 풀이 이력 조회 (submissions 집계)
 */
private async getProblemAttemptHistory(
  userId: string,
  problemId: string
): Promise<{
  total_attempts: number;
  solve_count: number;
  last_attempted_at: string;
  last_result: boolean | null;
} | null> {
  const supabase = this.supabaseService.getAdminClient();

  // submissions + feedbacks 조인하여 집계
  const { data } = await supabase
    .from('submissions')
    .select(`
      id,
      submitted_at,
      feedbacks!inner(is_correct)
    `)
    .eq('user_id', userId)
    .eq('problem_id', problemId)
    .eq('status', 'completed')
    .order('submitted_at', { ascending: false });

  if (!data || data.length === 0) {
    return null;
  }

  const totalAttempts = data.length;
  const solveCount = data.filter(
    (s: any) => s.feedbacks?.is_correct === true
  ).length;
  const lastAttemptedAt = data[0].submitted_at;
  const lastResult = data[0].feedbacks?.is_correct ?? null;

  return {
    total_attempts: totalAttempts,
    solve_count: solveCount,
    last_attempted_at: lastAttemptedAt,
    last_result: lastResult,
  };
}
```

---

#### 3.2.2. `runAIPipeline()` 내부 수정

AI 피드백 생성 시 재풀이 정보를 전달:

```typescript
private async runAIPipeline(
  submissionId: string,
  problemId: string,
  solutionImageUrl: string,
  userAnswer?: string
) {
  // ... (문제 정보 조회)

  const { data: submission } = await supabase
    .from('submissions')
    .select('user_id, started_at, submitted_at, is_retry, solve_count_at_submission')
    .eq('id', submissionId)
    .single();

  // 이전 풀이 결과 조회 (재풀이인 경우만)
  let prevResult = null;
  if (submission.is_retry) {
    const prevHistory = await this.getProblemAttemptHistory(
      submission.user_id,
      problemId
    );
    prevResult = prevHistory?.last_result;
  }

  // AI 파이프라인 호출 (재풀이 정보 추가)
  const result = await this.aiService.runPipeline(
    problem.problem_text,
    problem.requires_image ? problem.problem_image_url : null,
    solutionImageUrl,
    userAnswer || '',
    problem.answer || '',
    problem.solution_text,

    // 추가 파라미터
    {
      is_retry: submission.is_retry,
      prev_result: prevResult,
    }
  );

  // ... (피드백 저장, 게이미피케이션 처리)
}
```

---

### 3.3. `ai.service.ts` 수정

#### `runPipeline()` 메서드 시그니처 변경

```typescript
async runPipeline(
  problemText: string,
  problemImageUrl: string | null,
  solutionImageUrl: string,
  userAnswer: string,
  correctAnswer: string,
  solutionText: string,
  retryContext?: {
    is_retry: boolean;
    prev_result: string | null;
  }
): Promise<AIPipelineResult> {
  // ... (기존 로직)

  // 피드백 생성 시 재풀이 컨텍스트 전달
  const feedback = await this.geminiService.generateFeedback(
    problemText,
    userAnswer,
    correctAnswer,
    analysisResult,
    retryContext
  );

  // ...
}
```

---

#### `gemini.service.ts` - 프롬프트 조정

```typescript
async generateFeedback(
  problemText: string,
  userAnswer: string,
  correctAnswer: string,
  analysisResult: any,
  retryContext?: {
    is_retry: boolean;
    prev_result: string | null;
  }
): Promise<any> {
  let contextPrompt = '';

  // 재풀이 상황에 따른 프롬프트 조정
  if (retryContext?.is_retry) {
    if (retryContext.prev_result === 'wrong' && analysisResult.is_correct) {
      contextPrompt = `
이 학생은 이전에 이 문제를 틀렸지만, 이번에는 정답을 맞혔습니다.
약점을 극복한 것에 대해 칭찬하고, 이전 실수를 어떻게 개선했는지 언급해 주세요.
`;
    } else if (retryContext.prev_result === 'correct' && analysisResult.is_correct) {
      contextPrompt = `
이 학생은 이전에도 이 문제를 맞혔고, 이번에도 맞혔습니다.
개념을 잘 기억하고 있다는 점을 긍정적으로 평가해 주세요. 단, 너무 길지 않게.
`;
    } else if (retryContext.prev_result === 'correct' && !analysisResult.is_correct) {
      contextPrompt = `
이 학생은 이전에는 이 문제를 맞혔지만, 이번에는 틀렸습니다.
깜빡했거나 실수한 것으로 보이니, 부드럽게 개념을 상기시켜 주세요.
`;
    }
  }

  const prompt = `
${BASE_FEEDBACK_PROMPT}

${contextPrompt}

문제: ${problemText}
정답: ${correctAnswer}
학생 답안: ${userAnswer}

...
`;

  // ... (나머지 로직)
}
```

---

### 3.4. `gamification.service.ts` 수정

#### 3.4.1. `calculateAndAwardXP()` 로직 수정

```typescript
async calculateAndAwardXP(
  userId: string,
  problemId: string,
  pscore: number,
  submissionId?: string
): Promise<XPEarned> {
  const client = this.supabase.getAdminClient();
  let totalXP = 0;
  const logs: XPLog[] = [];

  // 1. Submission 정보 조회 (XP 배율 포함)
  const { data: submission } = await client
    .from('submissions')
    .select('xp_multiplier, is_retry')
    .eq('id', submissionId)
    .single();

  const xpMultiplier = submission?.xp_multiplier ?? 1.0;

  // 정답이 아니면 XP 지급 안 함
  if (pscore <= 0.7) {
    return { base: 0, total: 0, logs: [] };
  }

  // 2. 기본 XP 계산 (기존 로직)
  const { data: problem } = await client
    .from('problems')
    .select('psr, total_attempts')
    .eq('id', problemId)
    .single();

  let baseXP = 100;
  if (problem.total_attempts >= 100) {
    baseXP = Math.floor(problem.psr / 10);
  }

  // ========== XP 배율 적용 ==========
  const adjustedBaseXP = Math.floor(baseXP * xpMultiplier);
  totalXP += adjustedBaseXP;
  logs.push({ type: 'problem_solved', amount: adjustedBaseXP });

  // 3. 미개척 보너스 (최초 풀이만)
  if (!submission.is_retry && problem.total_attempts <= 10) {
    const bonusXP = Math.floor(300 * xpMultiplier);
    totalXP += bonusXP;
    logs.push({ type: 'unexplored_bonus', amount: bonusXP });
  }

  // 4. 일일 퀘스트 보너스 (배율 미적용)
  const todayCount = await this.getTodayProblemsCount(userId);
  if (todayCount === 3) {
    totalXP += 100;
    logs.push({ type: 'daily_quest_3', amount: 100 });
  } else if (todayCount === 5) {
    totalXP += 150;
    logs.push({ type: 'daily_quest_5', amount: 150 });
  } else if (todayCount === 10) {
    totalXP += 200;
    logs.push({ type: 'daily_quest_10', amount: 200 });
  }

  // 5. XP 부여 및 로그 기록
  await client.rpc('increment_xp', { user_id: userId, amount: totalXP });

  for (const log of logs) {
    await client.from('xp_logs').insert({
      user_id: userId,
      submission_id: submissionId,
      xp_amount: log.amount,
      xp_type: log.type,
    });
  }

  return { base: adjustedBaseXP, total: totalXP, logs };
}
```

---

#### 3.4.2. `updatePSR()` 로직 수정

```typescript
async updatePSR(
  userId: string,
  problemId: string,
  pscore: number,
  solveTimeSeconds?: number
): Promise<PSRChange> {
  const client = this.supabase.getAdminClient();

  // 1. 현재 PSR 조회
  const { data: user } = await client
    .from('profiles')
    .select('current_psr')
    .eq('id', userId)
    .single();

  const { data: problem } = await client
    .from('problems')
    .select('psr, total_attempts')
    .eq('id', problemId)
    .single();

  const userPSR = user.current_psr || 1000;
  const problemPSR = problem.psr || 1000;
  const N = problem.total_attempts || 0;

  // 2. 이전 풀이 결과 조회 (submissions 기반)
  const { data: prevSubmissions } = await client
    .from('submissions')
    .select(`
      id,
      submitted_at,
      feedbacks!inner(is_correct)
    `)
    .eq('user_id', userId)
    .eq('problem_id', problemId)
    .eq('status', 'completed')
    .order('submitted_at', { ascending: false })
    .limit(2);  // 최근 2개 (현재 제출 제외하고 이전 것 확인)

  let prevResult: boolean | null = null;
  if (prevSubmissions && prevSubmissions.length > 1) {
    // 두 번째 항목이 이전 결과 (첫 번째는 현재 진행 중이거나 아직 완료 안 됨)
    prevResult = prevSubmissions[1].feedbacks?.is_correct ?? null;
  }

  const isCorrect = pscore > 0.7;

  // ========== 3. PSR 변동 결정 ==========
  let kFactorUser = this.calculateKFactor(N, 'user');
  let shouldUpdateUserPSR = true;

  // 변동 매트릭스 적용
  if (prevResult === true && isCorrect) {
    // 정답 -> 정답: PSR 변동 없음 (Lock)
    shouldUpdateUserPSR = false;
  } else if (prevResult === true && !isCorrect) {
    // 정답 -> 오답: 하락
    shouldUpdateUserPSR = true;
  } else if (prevResult === false && isCorrect) {
    // 오답 -> 정답: 상승 (K-Factor 0.8배)
    kFactorUser = Math.floor(kFactorUser * 0.8);
    shouldUpdateUserPSR = true;
  } else if (prevResult === false && !isCorrect) {
    // 오답 -> 오답: 하락 (점진적 감소)
    // 연속 오답 횟수 계산
    const { data: recentSubmissions } = await client
      .from('submissions')
      .select(`feedbacks!inner(is_correct)`)
      .eq('user_id', userId)
      .eq('problem_id', problemId)
      .eq('status', 'completed')
      .order('submitted_at', { ascending: false })
      .limit(5);

    const wrongAttempts = recentSubmissions?.filter(
      (s: any) => s.feedbacks?.is_correct === false
    ).length || 1;

    kFactorUser = Math.floor(kFactorUser * Math.max(0.5, 1 - wrongAttempts * 0.1));
    shouldUpdateUserPSR = true;
  }
  // prevResult === null: 최초 풀이는 정상 적용

  // 4. 예상 결과 (E) 계산
  const expectedUser = 1 / (1 + Math.pow(10, (problemPSR - userPSR) / 400));
  const expectedProblem = 1 / (1 + Math.pow(10, (userPSR - problemPSR) / 400));

  // 5. PScore 조정 (풀이 시간 반영, 기존 로직 유지)
  let adjustedPScore = pscore;
  // ... (기존 로직)

  // 6. PSR 변동 계산
  let userPSRChange = 0;
  let newUserPSR = userPSR;

  if (shouldUpdateUserPSR) {
    userPSRChange = Math.round(kFactorUser * (adjustedPScore - expectedUser));
    newUserPSR = userPSR + userPSRChange;
  }

  const kFactorProblem = this.calculateKFactor(N, 'problem');
  const problemPSRChange = Math.round(
    kFactorProblem * (1 - adjustedPScore - expectedProblem)
  );
  const newProblemPSR = problemPSR + problemPSRChange;

  // 7. DB 업데이트
  if (shouldUpdateUserPSR) {
    await client
      .from('profiles')
      .update({ current_psr: newUserPSR })
      .eq('id', userId);
  }

  await client
    .from('problems')
    .update({ psr: newProblemPSR })
    .eq('id', problemId);

  return {
    user_psr_before: userPSR,
    user_psr_after: newUserPSR,
    change: userPSRChange,
    problem_psr_before: problemPSR,
    problem_psr_after: newProblemPSR,
    problem_psr_change: problemPSRChange,
    k_factor: kFactorUser,
  };
}
```

---

#### 3.4.3. 배치 조회 최적화

문제집 단위로 여러 문제의 풀이 이력을 한 번에 조회하는 메서드:

```typescript
/**
 * 문제집의 모든 문제에 대한 사용자 풀이 이력 배치 조회
 * (성능 최적화: N+1 쿼리 방지)
 */
async getProblemsAttemptHistoryBatch(
  userId: string,
  problemIds: string[]
): Promise<Map<string, AttemptHistory>> {
  const supabase = this.supabaseService.getAdminClient();

  // 한 번의 쿼리로 모든 문제의 풀이 이력 조회
  const { data } = await supabase
    .from('submissions')
    .select(`
      problem_id,
      submitted_at,
      feedbacks!inner(is_correct)
    `)
    .eq('user_id', userId)
    .in('problem_id', problemIds)
    .eq('status', 'completed')
    .order('submitted_at', { ascending: false });

  if (!data || data.length === 0) {
    return new Map();
  }

  // 문제별로 그룹화
  const grouped = data.reduce((acc: any, submission: any) => {
    const problemId = submission.problem_id;
    if (!acc[problemId]) {
      acc[problemId] = [];
    }
    acc[problemId].push(submission);
    return acc;
  }, {});

  // 각 문제별 집계
  const resultMap = new Map<string, AttemptHistory>();

  for (const [problemId, submissions] of Object.entries(grouped)) {
    const subs = submissions as any[];
    const totalAttempts = subs.length;
    const solveCount = subs.filter(
      (s: any) => s.feedbacks?.is_correct === true
    ).length;
    const lastAttemptedAt = subs[0].submitted_at;
    const lastResult = subs[0].feedbacks?.is_correct ?? null;

    resultMap.set(problemId, {
      total_attempts: totalAttempts,
      solve_count: solveCount,
      last_attempted_at: lastAttemptedAt,
      last_result: lastResult,
    });
  }

  return resultMap;
}
```

#### 사용 예시

```typescript
// 문제 목록 조회 시
const problemIds = problems.map(p => p.id);
const historyMap = await this.getProblemsAttemptHistoryBatch(
  userId,
  problemIds
);

// 각 문제에 이력 정보 첨부
const problemsWithStatus = problems.map(problem => ({
  ...problem,
  attempt_history: historyMap.get(problem.id) || null,
}));
```

---

### 3.5. 새 API 엔드포인트 추가

#### 3.5.1. `problems.controller.ts` - 문제 상태 조회

```typescript
@Get(':id/status')
@UseGuards(AuthGuard)
async getProblemStatus(
  @Param('id') problemId: string,
  @Req() req: any
) {
  return this.problemsService.getProblemStatus(problemId, req.user.id);
}
```

#### 3.5.2. `problems.service.ts`

```typescript
/**
 * 특정 문제의 사용자 풀이 상태 조회
 */
async getProblemStatus(problemId: string, userId: string) {
  const supabase = this.supabase.getAdminClient();

  // 1. 문제 기본 정보
  const { data: problem } = await supabase
    .from('problems')
    .select('id, total_correct')
    .eq('id', problemId)
    .single();

  if (!problem) {
    throw new NotFoundException('Problem not found');
  }

  // 2. 사용자 풀이 이력 (submissions 기반)
  const { data: submissions } = await supabase
    .from('submissions')
    .select(`
      id,
      submitted_at,
      feedbacks!inner(is_correct)
    `)
    .eq('user_id', userId)
    .eq('problem_id', problemId)
    .eq('status', 'completed')
    .order('submitted_at', { ascending: false });

  let attemptHistory = null;
  if (submissions && submissions.length > 0) {
    const totalAttempts = submissions.length;
    const solveCount = submissions.filter(
      (s: any) => s.feedbacks?.is_correct === true
    ).length;
    const lastAttemptedAt = submissions[0].submitted_at;
    const lastResult = submissions[0].feedbacks?.is_correct;

    attemptHistory = {
      total_attempts: totalAttempts,
      solve_count: solveCount,
      last_attempted_at: lastAttemptedAt,
      last_result: lastResult,
    };
  }

  // 3. 상태 뱃지 정보 계산
  let solveStatusBadge = {
    label: '미도전',
    color: 'gray',
    style: 'outlined',
  };

  if (attemptHistory) {
    if (attemptHistory.last_result === true) {
      solveStatusBadge = {
        label: '정복 ✓',
        color: 'green',
        style: 'filled',
      };
    } else if (attemptHistory.last_result === false) {
      solveStatusBadge = {
        label: '취약 ✗',
        color: 'red',
        style: 'filled',
      };
    }
  }

  // 4. 미정복 문제 뱃지
  const unconqueredBadge =
    problem.total_correct < 10
      ? {
          label: '🔥 미정복',
          color: 'orange',
          style: 'filled',
        }
      : null;

  // 5. 복습 모드 정보
  let retryInfo = null;
  if (attemptHistory) {
    const lastAttemptTime = new Date(attemptHistory.last_attempted_at);
    const now = new Date();
    const hoursSinceLastAttempt =
      (now.getTime() - lastAttemptTime.getTime()) / (1000 * 60 * 60);

    let expectedXpRate = 1.0;
    if (hoursSinceLastAttempt < 24) {
      expectedXpRate = 0.0;
    } else {
      const solveCount = attemptHistory.solve_count;
      if (solveCount === 1) {
        expectedXpRate = 0.3;
      } else if (solveCount === 2) {
        expectedXpRate = 0.1;
      } else if (solveCount >= 3) {
        expectedXpRate = 0.0;
      }
    }

    retryInfo = {
      is_retry: true,
      attempt_count: attemptHistory.total_attempts + 1,
      solve_count: attemptHistory.solve_count,
      last_attempted_at: attemptHistory.last_attempted_at,
      expected_xp_rate: expectedXpRate,
      cooldown_active: hoursSinceLastAttempt < 24,
    };
  }

  return {
    problem_id: problemId,
    solve_status_badge: solveStatusBadge,
    unconquered_badge: unconqueredBadge,
    retry_info: retryInfo,
  };
}
```

---

## 4. API 명세

### 4.1. 문제 상태 조회

**Endpoint**: `GET /problems/:id/status`

**Request Headers**:

```
Authorization: Bearer {access_token}
```

**Response** (200 OK):

```json
{
  "problem_id": "uuid",
  "solve_status_badge": {
    "label": "정복 ✓",
    "color": "green",
    "style": "filled"
  },
  "unconquered_badge": {
    "label": "🔥 미정복",
    "color": "orange",
    "style": "filled"
  },
  "retry_info": {
    "is_retry": true,
    "attempt_count": 3,
    "solve_count": 1,
    "last_attempted_at": "2025-11-20T12:00:00Z",
    "expected_xp_rate": 0.3,
    "cooldown_active": false
  }
}
```

---

### 4.2. 문제 제출 (기존 API 응답 변경 없음)

**Endpoint**: `POST /submissions`

기존과 동일하게 동작하되, 내부 로직에서 재풀이 여부를 자동 판단.

---

### 4.3. 피드백 조회 (기존 API 응답 변경 없음)

**Endpoint**: `GET /submissions/:id/feedback`

피드백 JSON에 이미 재풀이 컨텍스트가 반영되어 있으므로 응답 구조 변경 없음.

---

## 5. 마이그레이션 파일

### 5.1. `supabase/migrations/006_alter_submissions_for_retry.sql`

```sql
-- submissions 테이블 컬럼 추가
ALTER TABLE submissions
ADD COLUMN IF NOT EXISTS is_retry BOOLEAN DEFAULT FALSE,
ADD COLUMN IF NOT EXISTS attempt_count INT DEFAULT 1,
ADD COLUMN IF NOT EXISTS solve_count_at_submission INT DEFAULT 0,
ADD COLUMN IF NOT EXISTS xp_multiplier DECIMAL(3, 2) DEFAULT 1.00;

-- 성능 최적화 인덱스
-- 1. 사용자별 문제별 풀이 이력 조회 최적화
CREATE INDEX IF NOT EXISTS idx_submissions_user_problem_time
ON submissions(user_id, problem_id, submitted_at DESC);

-- 2. 문제 풀이 상태 배치 조회 최적화 (커버링 인덱스)
CREATE INDEX IF NOT EXISTS idx_submissions_user_status
ON submissions(user_id, problem_id, submitted_at DESC)
INCLUDE (id, status);

-- 3. 재풀이 분석용 인덱스
CREATE INDEX IF NOT EXISTS idx_submissions_retry
ON submissions(user_id, problem_id, is_retry)
WHERE status = 'completed';

-- 4. feedbacks와 조인 최적화를 위한 복합 인덱스
CREATE INDEX IF NOT EXISTS idx_submissions_join_feedback
ON submissions(id, user_id, problem_id, status, submitted_at DESC);
```

**참고**: 기존 `idx_submissions_user_problem_time` 인덱스가 있다면 먼저 삭제 후 생성:

```sql
DROP INDEX IF EXISTS idx_submissions_user_problem_time;
```

---

## 6. 추가 고려 사항

### 6.1. 어뷰징 방지

- 서버 시간 기준으로 쿨타임 검증
- 짧은 시간 내 대량 제출 감지
- XP 로그 모니터링

### 6.2. 성능 최적화

- **인덱스**: 마이그레이션 파일에 정의된 4개 인덱스 사용
- **배치 조회**: N+1 방지를 위해 `getProblemsAttemptHistoryBatch()` 사용
- **성능 목표**: 단일 조회 < 50ms, 배치 조회 < 200ms

---

## 7. 참고 자료

- **기획 문서**: `mds/index2.md`
- **Prisma Schema**: `prisma/schema.prisma`
- **서버 코드**: `src/submissions/submissions.service.ts`, `src/gamification/gamification.service.ts`

---

## 8. 체크리스트

### 서버 구현

- [ ] `submissions` 테이블 컬럼 추가 마이그레이션
- [ ] 성능 최적화 인덱스 추가 마이그레이션
- [ ] `submissions.service.ts`: `getProblemAttemptHistory()` 메서드 (submissions 기반)
- [ ] `submissions.service.ts`: `getProblemsAttemptHistoryBatch()` 메서드 (배치 조회)
- [ ] `submissions.service.ts`: XP 배율 결정 로직
- [ ] `gamification.service.ts`: XP 배율 적용 로직
- [ ] `gamification.service.ts`: PSR 변동 로직 수정 (submissions 기반)
- [ ] `ai.service.ts`: 재풀이 컨텍스트 전달
- [ ] `gemini.service.ts`: 프롬프트 조정 로직
- [ ] `problems.controller.ts`: `/problems/:id/status` 엔드포인트
- [ ] `problems.service.ts`: `getProblemStatus()` 메서드 (submissions 기반)

### 테스트

- [ ] 최초 풀이 → 재풀이 플로우
- [ ] XP 배율 적용 검증 (100% / 30% / 10% / 0%)
- [ ] 쿨타임 체크 (24시간 이내)
- [ ] PSR 변동 매트릭스 검증
- [ ] AI 피드백 컨텍스트 검증
- [ ] API 응답 검증
- [ ] 성능 테스트 (단일 조회 < 50ms, 배치 조회 < 200ms)
- [ ] 배치 조회 N+1 방지 검증
- [ ] 동시 제출 테스트 (Race condition)

---

## 9. 핵심 타입 정의

```typescript
// src/common/types/retry.types.ts

export interface AttemptHistory {
  total_attempts: number;
  solve_count: number;
  last_attempted_at: string;
  last_result: boolean | null;
}

export interface RetryContext {
  is_retry: boolean;
  prev_result: boolean | null;
}
```

---

j## 10. FAQ

### Q1. 왜 문제별로 매번 집계 쿼리를 실행하나요?

실제로는 **배치 조회**를 사용하여 N+1 쿼리 문제를 방지합니다. 문제집 단위로 한 번에 조회합니다.

### Q2. 동시에 같은 문제를 제출하면 어떻게 되나요?

submission 생성 시점에 `solve_count_at_submission`을 스냅샷으로 저장하므로, 이후 다른 제출과 독립적으로 처리됩니다.

---

## 11. 구현 순서

1. DB 마이그레이션 (submissions 테이블 수정 + 인덱스)
2. `getProblemAttemptHistory()` 메서드 구현
3. `getProblemsAttemptHistoryBatch()` 메서드 구현 (배치 조회)
4. XP 배율 로직 구현
5. PSR 변동 로직 수정
6. AI 피드백 재풀이 컨텍스트 추가
7. 문제 상태 조회 API 구현
8. 테스트 및 검증

---

**문서 작성일**: 2025-11-23  
**최종 수정일**: 2025-11-23  
**작성자**: AI Assistant  
**버전**: 2.3 (간결화)
