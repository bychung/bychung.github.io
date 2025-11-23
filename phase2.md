# Phase 2: Gamification Layer - 상세 구현 기획

## 1. 개요

### 1.1. Phase 2 목표

**"레벨과 PSR 시각화가 학생의 학습 동기를 높이고 재참여를 유도하는가?"**

Phase 2는 Phase 1에서 검증된 AI 피드백 위에 게이미피케이션 레이어를 추가하여, 다음을 구현한다:

1. **레벨 시스템**: XP 기반 노력 지표로 꾸준한 학습 유도
2. **PSR 시스템**: Elo 레이팅 기반 실력 측정으로 성장 가시화
3. **뱃지 시스템**: 학습 여정의 의미있는 순간 기념
4. **메인 대시보드**: 레벨, PSR, 일일 퀘스트, 스트릭을 한눈에

### 1.2. 성공 기준

- [ ] 레벨 시스템이 하루 5-10문제 풀이를 유도한다
- [ ] PSR이 학생의 실력을 적절히 반영한다 (시뮬레이션 테스트 통과)
- [ ] 일일 퀘스트가 학습 지속성을 높인다
- [ ] 뱃지 획득이 학습 목표 설정에 효과적이다
- [ ] 전체 시스템이 Phase 1 기능을 깨지 않고 통합된다

### 1.3. Phase 1과의 차이점

| 항목            | Phase 1               | Phase 2                          |
| --------------- | --------------------- | -------------------------------- |
| **풀이 제출**   | 단순 제출 + AI 피드백 | + XP 획득 + PSR 변동 + 뱃지 획득 |
| **메인 화면**   | 문제집 페이지 뷰어    | 대시보드 (레벨, PSR, 퀘스트)     |
| **피드백 화면** | AI 피드백만           | + XP 획득량 표시 + 레벨업 알림   |
| **데이터**      | 제출 기록만           | + 통계, 뱃지, PSR 히스토리       |

---

## 2. 시스템 아키텍처

### 2.1. 전체 구조

```
┌─────────────────────────────────────────────────────────┐
│                    React Native App (태블릿)             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ 대시보드     │  │ 문제 풀이    │  │ 뱃지 컬렉션   │  │
│  │ (New)        │  │ (Updated)    │  │ (New)        │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│                      NestJS Backend                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Stats API    │  │ Gamification │  │ Badge System │  │
│  │ (New)        │  │ Module (New) │  │ (New)        │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│                   Supabase (PostgreSQL)                  │
│  New Tables: user_stats, badges, user_badges,           │
│              problem_stats, daily_quests, xp_logs        │
└─────────────────────────────────────────────────────────┘
```

### 2.2. 데이터 플로우

#### 문제 풀이 제출 플로우 (Phase 2 확장)

```
[User] 문제 풀이 제출 (Phase 1 동일)
  ↓
[Backend] AI 파이프라인 실행 → PScore 산출
  ↓
┌─────────────────────────────────────────┐
│ Phase 2 추가: Gamification 처리          │
│                                         │
│ 1. XP 계산 및 부여                       │
│    - 기본 문제 풀이 XP                   │
│    - PSR 기반 차등 XP (N≥100)           │
│    - 미개척 문제 보너스                  │
│                                         │
│ 2. 레벨업 체크                           │
│    - 현재 XP가 레벨업 기준 도달?         │
│    - 레벨업 시 타이틀 변경               │
│                                         │
│ 3. PSR 변동 계산                        │
│    - 학생 PSR 업데이트                   │
│    - 문제 PSR 업데이트                   │
│    - K-Factor 동적 조절                 │
│    - 풀이 시간 반영 (N≥30)              │
│                                         │
│ 4. 뱃지 획득 체크                        │
│    - 누적 문제 수, PSR 마일스톤 등       │
│    - 새 뱃지 획득 시 user_badges 추가    │
│                                         │
│ 5. 일일 퀘스트 진행도 업데이트           │
│    - 오늘 풀이 수 카운트                 │
│    - 3/5/10문제 달성 시 보너스 XP       │
│                                         │
│ 6. 연속 달성(스트릭) 체크                │
│    - 오늘 5문제 이상 달성?               │
│    - 연속 일수 업데이트                  │
│    - 7일/30일 보너스 XP                 │
└─────────────────────────────────────────┘
  ↓
[Backend] feedback + gamification_result 반환
  ↓
[App] 피드백 화면 + XP 획득 표시 + 레벨업 모달
```

---

## 3. 데이터베이스 설계

### 3.1. Phase 2 마이그레이션 파일

Phase 2에서는 **1개의 마이그레이션 파일**을 생성합니다:

- `004_phase2_gamification.sql`: 레벨, PSR, 뱃지 관련 테이블 및 필드

#### 3.1.1. 004_phase2_gamification.sql

```sql
-- ============================================================================
-- Migration: 004 - Phase 2 Gamification
-- Description: 레벨, PSR, 뱃지, 통계 테이블 및 기존 테이블 확장
-- ============================================================================

-- ============================================================================
-- 1. 기존 테이블 확장
-- ============================================================================

-- profiles 테이블에 게이미피케이션 필드 추가
ALTER TABLE profiles
ADD COLUMN xp INT DEFAULT 0 NOT NULL,
ADD COLUMN current_psr INT DEFAULT 1000 NOT NULL,
ADD COLUMN current_streak INT DEFAULT 0 NOT NULL,
ADD COLUMN longest_streak INT DEFAULT 0 NOT NULL,
ADD COLUMN last_active_date DATE;

-- problems 테이블에 PSR 및 통계 필드 추가
ALTER TABLE problems
ADD COLUMN psr INT DEFAULT 1000 NOT NULL,
ADD COLUMN total_attempts INT DEFAULT 0 NOT NULL,
ADD COLUMN total_correct INT DEFAULT 0 NOT NULL,
ADD COLUMN avg_solve_time_seconds INT;

-- submissions 테이블에 게이미피케이션 관련 필드 추가
ALTER TABLE submissions
ADD COLUMN xp_earned INT DEFAULT 0,
ADD COLUMN user_psr_before INT,
ADD COLUMN user_psr_after INT,
ADD COLUMN problem_psr_before INT,
ADD COLUMN problem_psr_after INT,
ADD COLUMN k_factor INT,
ADD COLUMN solve_time_seconds INT;

-- PSR 통계 조회를 위한 인덱스
CREATE INDEX idx_submissions_problem_solve_time ON submissions(problem_id, solve_time_seconds);
CREATE INDEX idx_submissions_user_psr_history ON submissions(user_id, submitted_at DESC);

-- ============================================================================
-- 2. 새 테이블 생성
-- ============================================================================

-- 레벨 디자인 마스터 테이블
CREATE TABLE IF NOT EXISTS level_design (
    level INT PRIMARY KEY,
    title VARCHAR(50) NOT NULL,
    min_xp INT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_level_design_min_xp ON level_design(min_xp);

-- 사용자 통계 스냅샷 (일별)
CREATE TABLE IF NOT EXISTS user_stats_daily (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
    stat_date DATE NOT NULL,

    -- 레벨 관련
    level INT NOT NULL,
    xp INT NOT NULL,
    xp_gained_today INT DEFAULT 0,

    -- PSR 관련
    psr INT NOT NULL,
    psr_change_today INT DEFAULT 0,

    -- 풀이 관련
    problems_solved_today INT DEFAULT 0,
    total_problems_solved INT NOT NULL,

    -- 스트릭 관련
    current_streak INT DEFAULT 0,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, stat_date)
);

CREATE INDEX idx_user_stats_daily_user_id ON user_stats_daily(user_id);
CREATE INDEX idx_user_stats_daily_date ON user_stats_daily(stat_date);

-- 뱃지 마스터 테이블
CREATE TABLE IF NOT EXISTS badges (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT NOT NULL,
    category VARCHAR(20) NOT NULL, -- intro, effort, consistency, skill, exploration, special
    emoji VARCHAR(10),
    condition_type VARCHAR(50) NOT NULL, -- total_problems, consecutive_days, psr_milestone, etc.
    condition_value INT,
    order_index INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 사용자 뱃지 획득 기록
CREATE TABLE IF NOT EXISTS user_badges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
    badge_id VARCHAR(50) REFERENCES badges(id) ON DELETE CASCADE,
    acquired_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, badge_id)
);

CREATE INDEX idx_user_badges_user_id ON user_badges(user_id);
CREATE INDEX idx_user_badges_badge_id ON user_badges(badge_id);

-- XP 획득 로그 (디버깅 및 분석용, submission과 무관한 XP 획득도 있음)
CREATE TABLE IF NOT EXISTS xp_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
    submission_id UUID REFERENCES submissions(id) ON DELETE SET NULL,
    xp_amount INT NOT NULL,
    xp_type VARCHAR(50) NOT NULL, -- daily_login, problem_solved, daily_quest, streak_bonus, etc.
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_xp_logs_user_id ON xp_logs(user_id);
CREATE INDEX idx_xp_logs_created_at ON xp_logs(created_at);

-- ============================================================================
-- 3. 레벨 디자인 초기 데이터 삽입
-- ============================================================================

-- Level 1~7: 각 레벨당 750 XP
INSERT INTO level_design (level, title, min_xp) VALUES
(1, '🌱 새싹 계산사', 0),
(2, '🌱 새싹 계산사', 750),
(3, '🌱 새싹 계산사', 1500),
(4, '🌱 새싹 계산사', 2250),
(5, '🌱 새싹 계산사', 3000),
(6, '🌱 새싹 계산사', 3750),
(7, '🌱 새싹 계산사', 4500),

-- Level 8~15: 900 + (n-8) × 150
(8, '🧮 방정식 탐험가', 5250),
(9, '🧮 방정식 탐험가', 6150),
(10, '🧮 방정식 탐험가', 7200),
(11, '🧮 방정식 탐험가', 8400),
(12, '🧮 방정식 탐험가', 9750),
(13, '🧮 방정식 탐험가', 11250),
(14, '🧮 방정식 탐험가', 12900),
(15, '🧮 방정식 탐험가', 14700),

-- Level 16~30: 2000 + (n-16) × 200
(16, '📐 기하학 모험가', 16650),
(17, '📐 기하학 모험가', 18650),
(18, '📐 기하학 모험가', 20850),
(19, '📐 기하학 모험가', 23250),
(20, '📐 기하학 모험가', 25850),
(21, '📐 기하학 모험가', 28650),
(22, '📐 기하학 모험가', 31650),
(23, '📐 기하학 모험가', 34850),
(24, '📐 기하학 모험가', 38250),
(25, '📐 기하학 모험가', 41850),
(26, '📐 기하학 모험가', 45650),
(27, '📐 기하학 모험가', 49650),
(28, '📐 기하학 모험가', 53850),
(29, '📐 기하학 모험가', 58250),
(30, '📐 기하학 모험가', 62850),

-- Level 31~50: 5000 + (n-31) × 300
(31, '🎯 함수 마스터', 67650),
(32, '🎯 함수 마스터', 72650),
(33, '🎯 함수 마스터', 77950),
(34, '🎯 함수 마스터', 83550),
(35, '🎯 함수 마스터', 89450),
(36, '🎯 함수 마스터', 95650),
(37, '🎯 함수 마스터', 102150),
(38, '🎯 함수 마스터', 108950),
(39, '🎯 함수 마스터', 116050),
(40, '🎯 함수 마스터', 123450),
(41, '🎯 함수 마스터', 131150),
(42, '🎯 함수 마스터', 139150),
(43, '🎯 함수 마스터', 147450),
(44, '🎯 함수 마스터', 156050),
(45, '🎯 함수 마스터', 164950),
(46, '🎯 함수 마스터', 174150),
(47, '🎯 함수 마스터', 183650),
(48, '🎯 함수 마스터', 193450),
(49, '🎯 함수 마스터', 203550),
(50, '🎯 함수 마스터', 213950),

-- 더미 레벨 (최고 레벨 처리용, 절대 도달 불가능)
(51, '🎯 함수 마스터', 999999999)
ON CONFLICT (level) DO NOTHING;

COMMENT ON TABLE level_design IS '레벨별 디자인 정보 (최소 XP, 타이틀 등)';
COMMENT ON COLUMN level_design.min_xp IS '해당 레벨의 시작 XP (inclusive). 다음 레벨까지 필요한 XP는 (다음 레벨 min_xp - 현재 레벨 min_xp)로 계산';

-- ============================================================================
-- 4. 뱃지 초기 데이터 삽입 (Phase 2 기본 뱃지)
-- ============================================================================

INSERT INTO badges (id, name, description, category, emoji, condition_type, condition_value, order_index) VALUES
-- 입문 뱃지 (3개)
('first_problem', '수학 여정의 시작', '첫 문제 풀이를 제출했어요', 'intro', '🎓', 'total_problems', 1, 1),
('first_feedback', '피드백 수령인', '첫 AI 피드백을 확인했어요', 'intro', '📝', 'first_feedback_check', NULL, 2),
('first_correct', '정확한 첫걸음', '첫 정답을 획득했어요', 'intro', '🎯', 'first_correct_answer', NULL, 3),

-- 노력 뱃지 (4개)
('problems_10', '열정의 씨앗', '누적 10문제를 풀었어요', 'effort', '🌟', 'total_problems', 10, 4),
('problems_50', '노력의 결실', '누적 50문제를 풀었어요', 'effort', '💎', 'total_problems', 50, 5),
('problems_100', '백문불여일견', '누적 100문제를 풀었어요', 'effort', '🏅', 'total_problems', 100, 6),
('problems_500', '천문불여일행', '누적 500문제를 풀었어요', 'effort', '👑', 'total_problems', 500, 7),

-- 꾸준함 뱃지 (1개, Phase 2에서는 7일만)
('streak_7', '불타는 일주일', '7일 연속 5문제 이상 달성', 'consistency', '🔥', 'consecutive_days', 7, 8),

-- 실력 성장 뱃지 (2개, Phase 2에서는 1100, 1200만)
('psr_1100', '성장의 증거', 'PSR 1100점을 돌파했어요', 'skill', '📈', 'psr_milestone', 1100, 9),
('psr_1200', '실력 도약', 'PSR 1200점을 돌파했어요', 'skill', '🚀', 'psr_milestone', 1200, 10),

-- 특별 뱃지 (2개)
('daily_10', '완벽한 하루', '하루에 10문제를 풀었어요', 'special', '🌈', 'daily_problems', 10, 11),
('level_10', '레벨 마스터', '레벨 10에 도달했어요', 'special', '🎪', 'level_milestone', 10, 12)
ON CONFLICT (id) DO NOTHING;

-- ============================================================================
-- 5. RLS (Row Level Security)
-- ============================================================================

ALTER TABLE level_design ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_stats_daily ENABLE ROW LEVEL SECURITY;
ALTER TABLE badges ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_badges ENABLE ROW LEVEL SECURITY;
ALTER TABLE xp_logs ENABLE ROW LEVEL SECURITY;

-- level_design: 모두 조회 가능
CREATE POLICY "Public read access for level_design"
ON level_design FOR SELECT
USING (true);

-- user_stats_daily: 본인만 조회 가능
CREATE POLICY "Users can view own stats"
ON user_stats_daily FOR SELECT
USING (auth.uid() = user_id);

-- badges: 모두 조회 가능
CREATE POLICY "Public read access for badges"
ON badges FOR SELECT
USING (true);

-- user_badges: 본인만 조회 가능
CREATE POLICY "Users can view own badges"
ON user_badges FOR SELECT
USING (auth.uid() = user_id);

-- xp_logs: 본인만 조회 가능
CREATE POLICY "Users can view own xp logs"
ON xp_logs FOR SELECT
USING (auth.uid() = user_id);

-- ============================================================================
-- 6. COMMENTS
-- ============================================================================

COMMENT ON COLUMN profiles.xp IS '누적 XP (절대 감소하지 않음, 레벨은 level_design 테이블과 JOIN하여 계산)';
COMMENT ON COLUMN profiles.current_psr IS '현재 PSR (Proficiency Skill Rating, 초기값 1000)';
COMMENT ON COLUMN profiles.current_streak IS '현재 연속 달성 일수 (5문제 이상 기준)';
COMMENT ON COLUMN profiles.longest_streak IS '최장 연속 달성 일수 기록';
COMMENT ON COLUMN profiles.last_active_date IS '마지막 활동 날짜 (스트릭 계산용)';

COMMENT ON COLUMN problems.psr IS '문제의 PSR (난이도 지표, 초기값 1000)';
COMMENT ON COLUMN problems.total_attempts IS '총 풀이 시도 횟수';
COMMENT ON COLUMN problems.total_correct IS '총 정답 횟수';
COMMENT ON COLUMN problems.avg_solve_time_seconds IS '평균 풀이 시간 (초)';

COMMENT ON TABLE user_stats_daily IS '사용자 일별 통계 스냅샷 (대시보드 및 그래프용)';
COMMENT ON TABLE badges IS '뱃지 마스터 데이터 (모든 뱃지 정의)';
COMMENT ON TABLE user_badges IS '사용자가 획득한 뱃지 기록';
COMMENT ON TABLE xp_logs IS 'XP 획득 로그 (디버깅 및 분석용, submission 없는 XP도 기록)';

COMMENT ON COLUMN submissions.k_factor IS 'PSR 계산 시 사용된 K-Factor';
COMMENT ON COLUMN submissions.solve_time_seconds IS '풀이 소요 시간 (초). started_at ~ submitted_at';
```

### 3.2. 데이터 관계도 (Phase 2 추가)

```
[Phase 1 관계도]
books (1) ──< (N) pages (1) ──< (N) problems
chapters (tree) ──< (N:M) problems (via chapter_ids JSONB)
profiles (1) ──< (N) submissions
problems (1) ──< (N) submissions
submissions (1) ──< (1) feedbacks

[Phase 2 추가 관계도]
level_design (1) ──< (N) profiles (via xp lookup)
profiles (1) ──< (N) user_stats_daily
profiles (1) ──< (N) user_badges (N) ──> (1) badges
profiles (1) ──< (N) xp_logs
submissions (1) ──> (N) xp_logs
```

**주요 특징 (Phase 2):**

- ✅ `level_design`: 레벨별 최소 XP, 타이틀 마스터 데이터
- ✅ `max_xp` 없이 다음 레벨의 `min_xp`로 계산 (데이터 중복 제거)
- ✅ `profiles.xp`로 `level_design` 조회하여 현재 레벨 계산
- ✅ 더미 레벨 (51) 추가로 최고 레벨 처리 간소화
- ✅ `total_problems_solved`는 저장하지 않고 submissions에서 COUNT
- ✅ `user_stats_daily`에 일별 스냅샷 저장 (그래프, 대시보드용)
- ✅ **PSR 히스토리**: submissions 테이블에 직접 저장 (별도 로그 테이블 불필요)
- ✅ **풀이 시간 통계**: submissions에서 직접 조회 (별도 테이블 불필요)
- ✅ **계산 가능한 값은 저장 안함**: psr_change, problem_psr_change는 before/after로 계산

---

## 4. API 설계

### 4.1. 통계 API

#### GET /users/me/stats

현재 사용자의 게이미피케이션 통계 조회

**Headers:**

```
Authorization: Bearer {access_token}
```

**Response:**

```json
{
  "user_id": "uuid",
  "level": {
    "current": 8,
    "title": "🧮 방정식 탐험가",
    "xp_current": 5700,
    "xp_for_level": 6150,
    "xp_for_next_level": 900,
    "xp_progress": 450,
    "xp_progress_percent": 50
  },
  "psr": {
    "current": 1150,
    "change_today": 15,
    "all_time_high": 1200
  },
  "problems": {
    "total_solved": 87,
    "solved_today": 7,
    "correct_rate": 0.78
  },
  "streak": {
    "current": 5,
    "longest": 12
  },
  "daily_quest": {
    "problems_solved_today": 7,
    "goal_3": true,
    "goal_5": true,
    "goal_10": false
  }
}
```

#### GET /users/me/stats/history

통계 히스토리 (그래프용)

**Query Parameters:**

- `period`: 기간 (7d, 30d, 90d, all)
- `metric`: 지표 (level, psr, problems)

**Response:**

```json
{
  "metric": "psr",
  "period": "30d",
  "data": [
    { "date": "2025-11-01", "value": 1050 },
    { "date": "2025-11-02", "value": 1065 },
    { "date": "2025-11-03", "value": 1070 }
    // ...
  ]
}
```

### 4.2. 뱃지 API

#### GET /badges

모든 뱃지 목록 (마스터 데이터)

**Response:**

```json
{
  "badges": [
    {
      "id": "first_problem",
      "name": "수학 여정의 시작",
      "description": "첫 문제 풀이를 제출했어요",
      "category": "intro",
      "emoji": "🎓",
      "condition_type": "total_problems",
      "condition_value": 1
    }
    // ...
  ]
}
```

#### GET /users/me/badges

현재 사용자가 획득한 뱃지 + 진행도

**Response:**

```json
{
  "acquired_badges": [
    {
      "badge_id": "first_problem",
      "name": "수학 여정의 시작",
      "emoji": "🎓",
      "acquired_at": "2025-11-18T10:30:00Z"
    },
    {
      "badge_id": "problems_10",
      "name": "열정의 씨앗",
      "emoji": "🌟",
      "acquired_at": "2025-11-20T15:45:00Z"
    }
  ],
  "progress": [
    {
      "badge_id": "problems_50",
      "name": "노력의 결실",
      "emoji": "💎",
      "current": 23,
      "target": 50,
      "progress_percent": 46
    }
  ]
}
```

**구현 참고:**

- `current` 값 계산 시:
  - `total_problems` 조건: submissions 테이블에서 COUNT
  - `psr_milestone` 조건: profiles.current_psr
  - `level_milestone` 조건: getUserLevel()로 현재 레벨 조회
  - `consecutive_days` 조건: profiles.current_streak

### 4.3. 풀이 제출 API (Phase 2 확장)

#### POST /submissions (수정)

**Request:** (Phase 1과 동일)

```json
{
  "problem_id": "uuid",
  "solution_image_base64": "data:image/png;base64,iVBORw0KG...",
  "user_answer": "2",
  "started_at": "2025-11-18T10:30:00Z"
}
```

**Response:** (Phase 2 확장)

```json
{
  "submission_id": "uuid",
  "status": "pending",
  "submitted_at": "2025-11-18T10:35:00Z"
}
```

#### GET /submissions/:submissionId/feedback (수정)

**Response:** (기존 피드백 + gamification_result 추가)

```json
{
  "submission_id": "uuid",
  "pscore_case": 1,
  "pscore_value": 1.0,
  "is_correct": true,
  "feedback": {
    // ... (Phase 1과 동일)
  },
  "gamification_result": {
    "xp_earned": {
      "base": 120,
      "daily_quest_bonus": 100,
      "total": 220
    },
    "level_up": {
      "happened": true,
      "old_level": 7,
      "new_level": 8,
      "new_title": "🧮 방정식 탐험가"
    },
    "psr_change": {
      "user_psr_before": 1135,
      "user_psr_after": 1152,
      "change": 17,
      "problem_psr_before": 1100,
      "problem_psr_after": 1097,
      "problem_psr_change": -3,
      "k_factor": 32
    },
    "badges_earned": [
      {
        "badge_id": "level_10",
        "name": "레벨 마스터",
        "emoji": "🎪"
      }
    ],
    "daily_quest": {
      "problems_solved_today": 5,
      "goal_achieved": "goal_5"
    }
  }
}
```

---

## 5. 백엔드 로직 상세

### 5.1. 게이미피케이션 처리 파이프라인

제출 처리 시 다음 순서로 게이미피케이션 로직 실행:

```typescript
// SubmissionsService.ts
async processSubmissionGamification(submission, feedback) {
  const result = {
    xp_earned: {},
    level_up: {},
    psr_change: {},
    badges_earned: [],
    daily_quest: {}
  };

  // 1. XP 계산 및 부여
  result.xp_earned = await this.gamificationService.calculateAndAwardXP(
    submission.user_id,
    submission.problem_id,
    feedback.pscore_value
  );

  // 2. 레벨업 체크
  result.level_up = await this.gamificationService.checkLevelUp(
    submission.user_id
  );

  // 3. PSR 변동 계산
  result.psr_change = await this.gamificationService.updatePSR(
    submission.user_id,
    submission.problem_id,
    feedback.pscore_value,
    submission.solve_time_seconds
  );

  // 4. 뱃지 획득 체크
  result.badges_earned = await this.badgeService.checkAndAwardBadges(
    submission.user_id
  );

  // 5. 일일 퀘스트 업데이트
  result.daily_quest = await this.gamificationService.updateDailyQuest(
    submission.user_id
  );

  // 6. 스트릭 체크
  await this.gamificationService.updateStreak(submission.user_id);

  return result;
}
```

### 5.2. XP 계산 로직

#### 5.2.1. calculateAndAwardXP()

```typescript
async calculateAndAwardXP(userId, problemId, pscore) {
  let totalXP = 0;
  const logs = [];

  // 1. 기본 문제 풀이 XP
  const problem = await this.problemsService.findOne(problemId);
  let baseXP = 100;

  if (problem.total_attempts >= 100) {
    // PSR 기반 차등 XP (N≥100)
    const problemPSR = problem.psr;
    baseXP = Math.floor(problemPSR / 10); // PSR 1000~1100: 100~110 XP
  }

  // 미개척 문제 보너스 (N≤10)
  if (problem.total_attempts <= 10) {
    totalXP += 300;
    logs.push({ type: 'unexplored_bonus', amount: 300 });
  }

  totalXP += baseXP;
  logs.push({ type: 'problem_solved', amount: baseXP });

  // 2. 일일 퀘스트 보너스 체크
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

  // 3. XP 부여
  await this.supabase
    .from('profiles')
    .update({ xp: raw('xp + ?', [totalXP]) })
    .eq('id', userId);

  // 4. XP 로그 기록
  for (const log of logs) {
    await this.supabase.from('xp_logs').insert({
      user_id: userId,
      xp_amount: log.amount,
      xp_type: log.type
    });
  }

  return { base: baseXP, bonus: totalXP - baseXP, total: totalXP, logs };
}
```

#### 5.2.2. 일일 첫 로그인 XP (별도 엔드포인트)

```typescript
// AuthService.ts 또는 별도 Daily Login 체크
async checkDailyLoginBonus(userId) {
  const today = new Date().toISOString().split('T')[0];

  // 오늘 이미 로그인 XP를 받았는지 확인
  const log = await this.supabase
    .from('xp_logs')
    .select()
    .eq('user_id', userId)
    .eq('xp_type', 'daily_login')
    .gte('created_at', `${today}T00:00:00Z`)
    .single();

  if (!log) {
    // 첫 로그인 XP 부여
    await this.gamificationService.awardXP(userId, 50, 'daily_login');
    return { awarded: true, amount: 50 };
  }

  return { awarded: false };
}
```

#### 5.2.3. user_stats_daily 업데이트

문제 풀이 시마다 `user_stats_daily`를 업데이트하여 통계 스냅샷을 저장합니다.

```typescript
async updateDailyStats(userId) {
  const today = new Date().toISOString().split('T')[0];

  // 현재 사용자 정보 조회
  const profile = await this.supabase
    .from('profiles')
    .select('xp, current_psr, current_streak')
    .eq('id', userId)
    .single();

  // 현재 레벨 조회
  const levelInfo = await this.getUserLevel(userId);

  // 오늘 풀이 수
  const { count: todayCount } = await this.supabase
    .from('submissions')
    .select('*', { count: 'exact', head: true })
    .eq('user_id', userId)
    .gte('submitted_at', `${today}T00:00:00Z`);

  // 총 문제 수
  const { count: totalCount } = await this.supabase
    .from('submissions')
    .select('*', { count: 'exact', head: true })
    .eq('user_id', userId);

  // UPSERT
  await this.supabase
    .from('user_stats_daily')
    .upsert({
      user_id: userId,
      stat_date: today,
      level: levelInfo.level,
      xp: profile.xp,
      psr: profile.current_psr,
      problems_solved_today: todayCount,
      total_problems_solved: totalCount,
      current_streak: profile.current_streak
    }, {
      onConflict: 'user_id,stat_date'
    });
}
```

### 5.3. 레벨 조회 및 레벨업 로직

#### 5.3.1. 사용자 현재 레벨 조회

```typescript
async getUserLevel(userId) {
  const profile = await this.supabase
    .from('profiles')
    .select('xp')
    .eq('id', userId)
    .single();

  // XP를 기반으로 현재 레벨 찾기: min_xp <= profile.xp인 가장 높은 레벨
  const currentLevel = await this.supabase
    .from('level_design')
    .select('*')
    .lte('min_xp', profile.xp)
    .order('level', { ascending: false })
    .limit(1)
    .single();

  // 다음 레벨 정보 조회
  const nextLevel = await this.supabase
    .from('level_design')
    .select('min_xp')
    .eq('level', currentLevel.level + 1)
    .single();

  // 다음 레벨까지 필요한 XP 계산 (max_xp가 없으므로 다음 레벨의 min_xp 사용)
  const xpForNextLevel = nextLevel.min_xp - currentLevel.min_xp;
  const xpProgress = profile.xp - currentLevel.min_xp;

  return {
    level: currentLevel.level,
    title: currentLevel.title,
    xp_current: profile.xp,
    xp_for_level: nextLevel.min_xp, // 다음 레벨 시작 XP
    xp_for_next_level: xpForNextLevel,
    xp_progress: xpProgress,
    xp_progress_percent: Math.floor((xpProgress / xpForNextLevel) * 100)
  };
}
```

#### 5.3.2. checkLevelUp()

```typescript
async checkLevelUp(userId) {
  const profile = await this.supabase
    .from('profiles')
    .select('xp')
    .eq('id', userId)
    .single();

  // 현재 레벨 조회
  const currentLevelInfo = await this.getUserLevel(userId);
  const oldLevel = currentLevelInfo.level;

  // 다음 레벨의 min_xp 조회
  const nextLevel = await this.supabase
    .from('level_design')
    .select('level, title, min_xp')
    .eq('level', oldLevel + 1)
    .single();

  // 레벨업 체크: 간단한 비교!
  const leveledUp = profile.xp >= nextLevel.min_xp;

  return {
    happened: leveledUp,
    old_level: oldLevel,
    new_level: leveledUp ? nextLevel.level : oldLevel,
    new_title: leveledUp ? nextLevel.title : currentLevelInfo.title
  };
}
```

### 5.4. PSR 계산 로직

#### 5.4.1. updatePSR()

```typescript
async updatePSR(userId, problemId, pscore, solveTimeSeconds) {
  // 1. 현재 PSR 조회
  const user = await this.supabase.from('profiles').select('current_psr').eq('id', userId).single();
  const problem = await this.supabase.from('problems').select('psr, total_attempts').eq('id', problemId).single();

  const userPSR = user.current_psr;
  const problemPSR = problem.psr;
  const N = problem.total_attempts;

  // 2. K-Factor 동적 조절
  const kFactorUser = this.calculateKFactor(N, 'user');
  const kFactorProblem = this.calculateKFactor(N, 'problem');

  // 3. 예상 결과 (E) 계산
  const expectedUser = 1 / (1 + Math.pow(10, (problemPSR - userPSR) / 400));
  const expectedProblem = 1 / (1 + Math.pow(10, (userPSR - problemPSR) / 400));

  // 4. PScore 조정 (풀이 시간 반영)
  let adjustedPScore = pscore;

  if (N >= 30 && solveTimeSeconds) {
    // 정규 분로 기반 PScore 조정
    const stats = await this.getProblemSolveTimeStats(problemId);
    if (stats.stdDev > 0) {
      const zScore = (solveTimeSeconds - stats.mean) / stats.stdDev;

      // 표준편차 ±0.5σ 밖이면 조정
      if (zScore < -0.5) {
        // 빠르게 풀었음 (상위 30% 이내) → PScore +0.1~0.2
        adjustedPScore = Math.min(1.0, pscore + 0.1 + Math.abs(zScore) * 0.05);
      } else if (zScore > 0.5) {
        // 느리게 풀었음 (하위 30% 이내) → PScore -0.1~0.2
        adjustedPScore = Math.max(0.0, pscore - 0.1 - zScore * 0.05);
      }
    }
  } else if (N >= 10 && N < 30 && solveTimeSeconds) {
    // 초기 데이터: 상위/하위 10% 기준
    const submissions = await this.supabase
      .from('submissions')
      .select('solve_time_seconds')
      .eq('problem_id', problemId)
      .not('solve_time_seconds', 'is', null);
    const times = submissions.map(s => s.solve_time_seconds);
    const percentile10 = this.percentile(times, 10);
    const percentile90 = this.percentile(times, 90);

    if (solveTimeSeconds < percentile10) {
      adjustedPScore = Math.min(1.0, pscore + 0.15);
    } else if (solveTimeSeconds > percentile90) {
      adjustedPScore = Math.max(0.0, pscore - 0.15);
    }
  }

  // 5. PSR 변동 계산
  const userPSRChange = Math.round(kFactorUser * (adjustedPScore - expectedUser));
  const problemPSRChange = Math.round(kFactorProblem * ((1 - adjustedPScore) - expectedProblem));

  const newUserPSR = userPSR + userPSRChange;
  const newProblemPSR = problemPSR + problemPSRChange;

  // 6. DB 업데이트
  await this.supabase
    .from('profiles')
    .update({ current_psr: newUserPSR })
    .eq('id', userId);

  await this.supabase
    .from('problems')
    .update({ psr: newProblemPSR })
    .eq('id', problemId);

  // 7. submissions 테이블에 PSR 정보 기록 (별도 로그 테이블 불필요)
  // 이미 제출 시 생성된 submission 레코드 업데이트
  // (이 메서드는 submission 생성 후 호출됨)

  return {
    user_psr_before: userPSR,
    user_psr_after: newUserPSR,
    change: userPSRChange,
    problem_psr_before: problemPSR,
    problem_psr_after: newProblemPSR,
    k_factor: kFactorUser
  };
}

calculateKFactor(N, type) {
  if (type === 'problem') {
    if (N < 10) return 20;
    if (N < 30) return 30;
    if (N < 100) return 40;
    return 20;
  }
  // User K-Factor는 추후 유저별 PSR 안정도에 따라 조절 가능
  return 32;
}

async getProblemSolveTimeStats(problemId) {
  // submissions 테이블에서 직접 조회 (별도 테이블 불필요)
  const times = await this.supabase
    .from('submissions')
    .select('solve_time_seconds')
    .eq('problem_id', problemId)
    .not('solve_time_seconds', 'is', null);

  const values = times.map(t => t.solve_time_seconds);
  const mean = values.reduce((a, b) => a + b, 0) / values.length;
  const variance = values.reduce((sum, val) => sum + Math.pow(val - mean, 2), 0) / values.length;
  const stdDev = Math.sqrt(variance);

  return { mean, stdDev };
}
```

### 5.5. 뱃지 획득 로직

#### 5.5.1. checkAndAwardBadges()

```typescript
async checkAndAwardBadges(userId) {
  const newBadges = [];

  // 사용자 통계 조회
  const profile = await this.supabase
    .from('profiles')
    .select('xp, current_psr, current_streak')
    .eq('id', userId)
    .single();

  // 현재 레벨 조회
  const levelInfo = await this.getUserLevel(userId);

  // 총 문제 풀이 수 (submissions에서 카운트)
  const { count: totalProblems } = await this.supabase
    .from('submissions')
    .select('*', { count: 'exact', head: true })
    .eq('user_id', userId);

  // 이미 획득한 뱃지 조회
  const acquiredBadges = await this.supabase
    .from('user_badges')
    .select('badge_id')
    .eq('user_id', userId);

  const acquiredIds = new Set(acquiredBadges.map(b => b.badge_id));

  // 모든 뱃지 조회
  const allBadges = await this.supabase.from('badges').select();

  for (const badge of allBadges) {
    if (acquiredIds.has(badge.id)) continue;

    let shouldAward = false;

    switch (badge.condition_type) {
      case 'total_problems':
        shouldAward = totalProblems >= badge.condition_value;
        break;

      case 'psr_milestone':
        shouldAward = profile.current_psr >= badge.condition_value;
        break;

      case 'level_milestone':
        shouldAward = levelInfo.level >= badge.condition_value;
        break;

      case 'first_feedback_check':
        // 제출 → 피드백 조회가 최소 1회 이상
        const feedbackCheck = await this.supabase
          .from('feedbacks')
          .select('id')
          .limit(1);
        shouldAward = feedbackCheck.length > 0;
        break;

      case 'first_correct_answer':
        const correctSubmission = await this.supabase
          .from('feedbacks')
          .select('id')
          .eq('is_correct', true)
          .in('submission_id', (await this.supabase
            .from('submissions')
            .select('id')
            .eq('user_id', userId)).map(s => s.id))
          .limit(1);
        shouldAward = correctSubmission.length > 0;
        break;

      case 'consecutive_days':
        shouldAward = profile.current_streak >= badge.condition_value;
        break;

      case 'daily_problems':
        const todayCount = await this.getTodayProblemsCount(userId);
        shouldAward = todayCount >= badge.condition_value;
        break;
    }

    if (shouldAward) {
      await this.supabase.from('user_badges').insert({
        user_id: userId,
        badge_id: badge.id
      });
      newBadges.push({
        badge_id: badge.id,
        name: badge.name,
        emoji: badge.emoji
      });
    }
  }

  return newBadges;
}
```

### 5.6. 스트릭 (연속 달성) 로직

#### 5.6.1. updateStreak()

```typescript
async updateStreak(userId) {
  const profile = await this.supabase
    .from('profiles')
    .select('current_streak, longest_streak, last_active_date')
    .eq('id', userId)
    .single();

  const today = new Date().toISOString().split('T')[0];
  const todayCount = await this.getTodayProblemsCount(userId);

  // 오늘 5문제 이상 달성했는지 확인
  if (todayCount < 5) return;

  const lastActiveDate = profile.last_active_date;
  let newStreak = profile.current_streak;

  if (!lastActiveDate) {
    // 첫 활동
    newStreak = 1;
  } else {
    const lastDate = new Date(lastActiveDate);
    const todayDate = new Date(today);
    const diffDays = Math.floor((todayDate - lastDate) / (1000 * 60 * 60 * 24));

    if (diffDays === 0) {
      // 오늘 이미 처리됨
      return;
    } else if (diffDays === 1) {
      // 연속 달성
      newStreak++;
    } else {
      // 연속 끊김
      newStreak = 1;
    }
  }

  const longestStreak = Math.max(profile.longest_streak, newStreak);

  await this.supabase
    .from('profiles')
    .update({
      current_streak: newStreak,
      longest_streak: longestStreak,
      last_active_date: today
    })
    .eq('id', userId);

  // 스트릭 보너스 체크
  if (newStreak === 7 || newStreak === 30) {
    const bonusXP = newStreak === 7 ? 1000 : 5000;
    await this.gamificationService.awardXP(userId, bonusXP, `streak_${newStreak}`);
  }
}
```

---

## 6. 앱 화면 및 기능 (React Native)

### 6.1. 화면 구조 (Phase 2 추가)

```
App
├── AuthStack (인증 전)
│   ├── LoginScreen
│   └── SignupScreen
└── MainStack (인증 후)
    ├── [NEW] DashboardScreen (메인 대시보드)
    ├── PageViewerScreen
    ├── ProblemSolvingScreen
    ├── FeedbackScreen (Updated)
    ├── [NEW] BadgeCollectionScreen (뱃지 컬렉션)
    └── [NEW] ProfileScreen (프로필 및 통계)
```

### 6.2. DashboardScreen (메인 대시보드) - 새로 추가

#### 6.2.1. 화면 구성

```
┌─────────────────────────────────────┐
│  Math Grow               [🎓 프로필]  │ ← Header
├─────────────────────────────────────┤
│                                     │
│  🧮 방정식 탐험가                    │ ← 타이틀
│  Level 8                            │
│  █████░░░░░ 50% (450/900)           │ ← XP 바 (현재 레벨 내 진행도)
│                                     │
├─────────────────────────────────────┤
│  📊 내 실력 점수                     │
│  1150 PSR        +15 (오늘)         │
│                                     │
│  [PSR 그래프 보기 →]                │
├─────────────────────────────────────┤
│  🎯 오늘의 목표                      │
│  ─────────────────────────────────  │
│  ✅ 3문제 풀기    (보너스 +100 XP)  │
│  ✅ 5문제 풀기    (보너스 +150 XP)  │
│  ⬜ 10문제 풀기   (보너스 +200 XP)  │
│                                     │
│  진행: 7 / 10                       │
├─────────────────────────────────────┤
│  🔥 연속 달성                        │
│  현재: 5일 연속 (최장: 12일)         │
│  7일 연속 달성까지 2일 남았어요!     │
├─────────────────────────────────────┤
│  🏅 최근 획득 뱃지                   │
│  [🎓] [🌟] [💎] [+2 더보기]        │
├─────────────────────────────────────┤
│                                     │
│        [📚 문제 풀러 가기]           │ ← CTA 버튼
│                                     │
└─────────────────────────────────────┘
```

#### 6.2.2. 기능

**데이터 로딩:**

- 컴포넌트 마운트 시 `GET /users/me/stats` 호출
  - 서버가 `profiles.xp`를 조회하고 `level_design`과 JOIN하여 현재 레벨, 타이틀 반환
- `GET /users/me/badges` 호출하여 최근 뱃지 표시

**XP 바:**

- 현재 레벨 내 XP 진행도 시각화
- 예: Level 8 (min_xp=5250, 다음 레벨 min_xp=6150)
  - 필요 XP: 6150 - 5250 = 900 (계산으로 구함)
  - 현재 5700이면 진행도: (5700 - 5250) / 900 = 50%
- 애니메이션 효과

**일일 퀘스트:**

- 3/5/10문제 체크박스 형태
- 실시간 진행도 업데이트 (문제 풀이 후 돌아올 때)

**CTA 버튼:**

- "문제 풀러 가기" → PageViewerScreen으로 이동

**기술 구현:**

- React Native Reanimated: XP 바 애니메이션
- Refresh Control: 스와이프로 새로고침

### 6.3. FeedbackScreen (Phase 2 업데이트)

#### 6.3.1. 화면 구성 (하단에 추가)

```
[Phase 1 피드백 내용]
...
├─────────────────────────────────────┤
│  🎉 보상 획득!                       │ ← Phase 2 추가
│  ─────────────────────────────────  │
│  💰 XP 획득: +220                   │
│    • 문제 풀이: +120                │
│    • 5문제 달성 보너스: +100        │
│                                     │
│  📈 PSR 변동: 1135 → 1152 (+17)    │
│                                     │
│  [레벨업 모달이 떴다면 여기 표시]     │
│                                     │
│  🏅 새 뱃지 획득!                    │
│  🎪 레벨 마스터                     │
│  레벨 10에 도달했어요!               │
│                                     │
└─────────────────────────────────────┘
```

#### 6.3.2. 레벨업 모달

레벨업이 발생한 경우 피드백 화면 위에 모달 표시:

```
┌─────────────────────────────────────┐
│                                     │
│         ✨ 레벨 업! ✨               │
│                                     │
│          Level 7 → 8                │
│                                     │
│      🧮 방정식 탐험가                │
│                                     │
│   다양한 유형에 도전하는 숙련자가     │
│   되었습니다!                        │
│                                     │
│          [계속하기]                  │
│                                     │
└─────────────────────────────────────┘
```

**애니메이션:**

- 레벨업 숫자가 커지는 효과
- 타이틀 변경 강조
- 축하 파티클 효과 (선택사항)

### 6.4. BadgeCollectionScreen (뱃지 컬렉션) - 새로 추가

#### 6.4.1. 화면 구성

```
┌─────────────────────────────────────┐
│  ← 뱃지 컬렉션                   12/12│ ← Header (획득/전체)
├─────────────────────────────────────┤
│  입문 뱃지                           │
│  ────────────────────────────────── │
│  [🎓]  [📝]  [🎯]                   │ ← 획득 (컬러)
│                                     │
├─────────────────────────────────────┤
│  노력 뱃지                           │
│  ────────────────────────────────── │
│  [🌟]  [💎]  [🏅]  [⬜]            │ ← 미획득 (회색)
│                                     │
│  노력의 결실 (💎)                    │ ← 탭 시 상세
│  누적 50문제를 풀었어요              │
│  진행: 23 / 50 (46%)                │
│  ▓▓▓▓▓░░░░░                         │
│                                     │
├─────────────────────────────────────┤
│  꾸준함 뱃지                         │
│  ────────────────────────────────── │
│  [🔥]  [⬜]                         │
│                                     │
├─────────────────────────────────────┤
│  실력 성장 뱃지                      │
│  ────────────────────────────────── │
│  [📈]  [🚀]  [⬜]                   │
│                                     │
├─────────────────────────────────────┤
│  특별 뱃지                           │
│  ────────────────────────────────── │
│  [🌈]  [🎪]                         │
│                                     │
└─────────────────────────────────────┘
```

#### 6.4.2. 기능

**뱃지 표시:**

- 획득: 컬러 이모지 + 이름
- 미획득: 회색 + 자물쇠 아이콘

**진행도 표시:**

- 미획득 뱃지를 탭하면 진행도 표시
- 프로그레스 바

**뱃지 상세:**

- 뱃지 탭 시 모달 또는 확장 뷰
- 획득 날짜 (획득한 경우)
- 달성 조건 설명

**기술 구현:**

- FlatList 또는 ScrollView
- 카테고리별 Section
- Animated 라이브러리로 뱃지 획득 효과

### 6.5. ProfileScreen (프로필 및 통계) - 새로 추가

#### 6.5.1. 화면 구성

```
┌─────────────────────────────────────┐
│  ← 프로필                            │
├─────────────────────────────────────┤
│  🧮 방정식 탐험가                    │
│  닉네임: 수학왕 | 학년: 2학년        │
│                                     │
│  Level 8  |  PSR 1150               │
├─────────────────────────────────────┤
│  📊 통계                             │
│  ────────────────────────────────── │
│  총 문제 풀이: 87문제                │
│  정답률: 78%                         │
│  최장 스트릭: 12일                   │
│                                     │
│  [PSR 변동 그래프]                   │
│  (30일)                             │
│  1200 ┤     ╱╲                      │
│  1150 ┤   ╱    ╲  ╱                │
│  1100 ┤ ╱        ╲╱                 │
│  1050 ┼────────────────→            │
│       11/1      11/15    11/30      │
│                                     │
├─────────────────────────────────────┤
│  🏅 뱃지 (12개 획득)                 │
│  [전체 보기 →]                       │
│                                     │
├─────────────────────────────────────┤
│  [로그아웃]                          │
└─────────────────────────────────────┘
```

#### 6.5.2. 기능

**프로필 정보:**

- 닉네임, 학년, 타이틀, 레벨, PSR

**통계:**

- 총 문제 풀이 (submissions 테이블에서 COUNT 또는 user_stats_daily의 최신 레코드)
- 정답률 (feedbacks 테이블에서 is_correct 비율)
- 최장 스트릭 (profiles.longest_streak)

**PSR 그래프:**

- `GET /users/me/stats/history?period=30d&metric=psr` 호출
- 선 그래프로 시각화

**기술 구현:**

- React Native Chart Kit 또는 Victory Native
- 그래프 인터랙션 (줌, 기간 선택)

---

## 7. 백엔드 모듈 구조 (NestJS)

### 7.1. 모듈 구조 (Phase 2 추가)

```
src/
├── main.ts
├── app.module.ts
├── auth/                        [Phase 1]
├── books/                       [Phase 1]
├── problems/                    [Phase 1]
├── submissions/                 [Phase 1, Updated]
│   ├── submissions.service.ts   (게이미피케이션 통합)
│   └── ...
├── ai/                          [Phase 1]
├── feedbacks/                   [Phase 1]
├── [NEW] gamification/
│   ├── gamification.module.ts
│   ├── gamification.service.ts  (XP, 레벨, PSR, 스트릭)
│   └── psr.service.ts           (PSR 전용 로직)
├── [NEW] badges/
│   ├── badges.module.ts
│   ├── badges.service.ts        (뱃지 획득 체크)
│   └── badges.controller.ts     (GET /badges)
├── [NEW] stats/
│   ├── stats.module.ts
│   ├── stats.service.ts         (통계 조회)
│   └── stats.controller.ts      (GET /users/me/stats)
└── common/
    ├── supabase.service.ts
    └── types/
```

### 7.2. GamificationModule

**책임:**

- XP 계산 및 부여
- 레벨 조회 및 레벨업 체크
- PSR 계산 및 업데이트
- 일일 퀘스트 업데이트
- 스트릭 업데이트

**주요 메서드:**

- `getUserLevel(userId)` - profiles.xp로 level_design 조회 (2 쿼리: 현재 레벨 + 다음 레벨)
- `calculateAndAwardXP()`
- `checkLevelUp()` - 다음 레벨의 min_xp와 비교
- `updatePSR()`
- `updateDailyQuest()`
- `updateStreak()`
- `updateDailyStats()` - user_stats_daily UPSERT

### 7.3. BadgesModule

**책임:**

- 뱃지 획득 조건 체크
- 뱃지 부여
- 뱃지 진행도 계산

**주요 메서드:**

- `checkAndAwardBadges()`
- `getBadgeProgress()`
- `getAllBadges()` (마스터 데이터)

### 7.4. StatsModule

**책임:**

- 통계 조회 API
- 일별 통계 스냅샷 생성

**주요 메서드:**

- `getUserStats()` - profiles.xp와 level_design JOIN하여 현재 레벨 정보 반환
- `getUserStatsHistory()` - user_stats_daily에서 히스토리 조회
- `getPSRHistory()` - submissions 테이블에서 PSR 변동 히스토리 조회
- `createDailySnapshot()` (Cron Job) - 매일 자정 user_stats_daily 생성

**PSR 히스토리 조회 예시:**

```typescript
async getPSRHistory(userId, period = '30d') {
  const startDate = this.getStartDate(period);

  const history = await this.supabase
    .from('submissions')
    .select(`
      submitted_at,
      user_psr_before,
      user_psr_after,
      problem_psr_before,
      problem_psr_after,
      k_factor,
      feedbacks (pscore_value)
    `)
    .eq('user_id', userId)
    .gte('submitted_at', startDate)
    .order('submitted_at', { ascending: true });

  return history.map(h => ({
    date: h.submitted_at,
    user_psr_before: h.user_psr_before,
    user_psr_after: h.user_psr_after,
    user_psr_change: h.user_psr_after - h.user_psr_before, // 계산
    problem_psr_change: h.problem_psr_after - h.problem_psr_before, // 계산
    pscore: h.feedbacks?.pscore_value,
    k_factor: h.k_factor
  }));
}
```

### 7.5. SubmissionsService 수정

Phase 2에서 제출 처리 시 게이미피케이션 로직 추가:

```typescript
async processSubmission(submissionId) {
  // 1. AI 파이프라인 실행 (Phase 1)
  const feedback = await this.aiService.runPipeline(submissionId);

  // 2. 게이미피케이션 처리 (Phase 2)
  const gamificationResult = await this.processSubmissionGamification(
    submission,
    feedback
  );

  // 3. submission 상태 업데이트 (PSR 정보 포함)
  await this.supabase
    .from('submissions')
    .update({
      status: 'completed',
      xp_earned: gamificationResult.xp_earned.total,
      user_psr_before: gamificationResult.psr_change.user_psr_before,
      user_psr_after: gamificationResult.psr_change.user_psr_after,
      problem_psr_before: gamificationResult.psr_change.problem_psr_before,
      problem_psr_after: gamificationResult.psr_change.problem_psr_after,
      k_factor: gamificationResult.psr_change.k_factor
    })
    .eq('id', submissionId);

  return { feedback, gamificationResult };
}
```

---

## 8. 개발 우선순위 및 마일스톤

### 8.1. Milestone 1: DB 및 백엔드 기초 (1주)

**목표:** DB 마이그레이션 + 게이미피케이션 모듈 구조

**백엔드:**

- [ ] `004_phase2_gamification.sql` 마이그레이션 작성 및 적용
- [ ] level_design 테이블 초기 데이터 삽입 (Level 1~50)
- [ ] Prisma 스키마 동기화
- [ ] GamificationModule 생성
- [ ] BadgesModule 생성
- [ ] StatsModule 생성

**테스트:**

- [ ] 마이그레이션 정상 적용 확인
- [ ] level_design 데이터 삽입 확인 (Level 1~50 + 더미 레벨 51)
- [ ] 뱃지 초기 데이터 삽입 확인
- [ ] XP로 레벨 조회 쿼리 테스트
- [ ] 다음 레벨 min_xp 조회 쿼리 테스트

### 8.2. Milestone 2: XP 및 레벨 시스템 (1-2주)

**목표:** XP 부여 + 레벨업 로직

**백엔드:**

- [ ] `calculateAndAwardXP()` 구현
- [ ] `checkLevelUp()` 구현
- [ ] 일일 로그인 XP 체크 구현
- [ ] 일일 퀘스트 XP 보너스 구현
- [ ] SubmissionsService에 XP 로직 통합

**앱:**

- [ ] DashboardScreen 구현 (기본 레이아웃)
- [ ] XP 바 표시
- [ ] 일일 퀘스트 표시

**테스트:**

- [ ] 문제 풀이 시 XP 정상 부여 확인
- [ ] XP 기반 레벨 조회 정확도 확인
- [ ] `xp_for_next_level` 계산 정확도 확인 (다음 레벨 min_xp - 현재 레벨 min_xp)
- [ ] 레벨업 정상 작동 확인 (경계값 테스트: 750, 1500, 5250 XP 등)
  - XP 749 → Level 1 유지
  - XP 750 → Level 2로 레벨업
- [ ] 더미 레벨 51 처리 확인 (Level 50에서 레벨업 시도 시)
- [ ] 일일 퀘스트 보너스 확인
- [ ] level_design 조회 쿼리 성능 확인 (2 쿼리: 현재 레벨 + 다음 레벨)

### 8.3. Milestone 3: PSR 시스템 (2주)

**목표:** PSR 계산 + 풀이 시간 반영

**백엔드:**

- [ ] `updatePSR()` 구현
- [ ] K-Factor 동적 조절 로직
- [ ] 풀이 시간 정규 분포 계산 (submissions 테이블에서 직접 조회)
- [ ] PScore 조정 로직 (시간 반영)
- [ ] SubmissionsService에 PSR 로직 통합 (k_factor 포함)
- [ ] submissions 테이블에 PSR 정보 저장 (별도 로그 테이블 불필요)

**앱:**

- [ ] DashboardScreen에 PSR 표시
- [ ] FeedbackScreen에 PSR 변동 표시

**테스트:**

- [ ] PSR 계산 정확도 검증 (수동 계산과 비교)
- [ ] K-Factor 동적 조절 확인
- [ ] 풀이 시간 반영 로직 테스트 (N≥30 문제)
- [ ] submissions 테이블에서 PSR 히스토리 조회 테스트
- [ ] psr_change, problem_psr_change 계산 정확도 확인

### 8.4. Milestone 4: 뱃지 시스템 (1주)

**목표:** 뱃지 획득 + 진행도 표시

**백엔드:**

- [ ] `checkAndAwardBadges()` 구현
- [ ] 뱃지 진행도 계산
- [ ] SubmissionsService에 뱃지 체크 통합
- [ ] `GET /badges` API 구현
- [ ] `GET /users/me/badges` API 구현

**앱:**

- [ ] BadgeCollectionScreen 구현
- [ ] 뱃지 획득 모달/애니메이션
- [ ] FeedbackScreen에 새 뱃지 표시

**테스트:**

- [ ] 뱃지 획득 조건 테스트 (각 뱃지별)
- [ ] 진행도 계산 정확도 확인

### 8.5. Milestone 5: 스트릭 및 통계 (1주)

**목표:** 연속 달성 + 통계 API

**백엔드:**

- [ ] `updateStreak()` 구현
- [ ] 스트릭 보너스 XP 로직
- [ ] `GET /users/me/stats` API 구현
- [ ] `GET /users/me/stats/history` API 구현
- [ ] 일별 통계 스냅샷 Cron Job (선택사항)

**앱:**

- [ ] DashboardScreen에 스트릭 표시
- [ ] ProfileScreen 구현 (기본)
- [ ] PSR 그래프 (간단한 버전)

**테스트:**

- [ ] 스트릭 계산 정확도 확인
- [ ] 연속 끊김 시나리오 테스트
- [ ] 7일/30일 보너스 XP 확인

### 8.6. Milestone 6: UI/UX 개선 및 통합 테스트 (1주)

**목표:** 레벨업 애니메이션 + 전체 플로우 테스트

**앱:**

- [ ] 레벨업 모달 구현
- [ ] 레벨업 애니메이션 (축하 효과)
- [ ] DashboardScreen 애니메이션 (XP 바)
- [ ] 뱃지 획득 애니메이션
- [ ] 전체 네비게이션 흐름 개선

**백엔드:**

- [ ] 성능 최적화 (PSR 계산, 뱃지 체크)
- [ ] 에러 핸들링 강화

**테스트:**

- [ ] E2E 테스트 (문제 풀이 → 피드백 → 대시보드)
- [ ] 레벨업 시나리오 테스트
- [ ] PSR 변동 시나리오 테스트
- [ ] 베타 테스터 피드백 수집

---

## 9. 주요 기술 과제 및 해결 방안

### 9.1. PSR 계산 복잡도

**과제:**

- PSR 계산 시 풀이 시간 통계를 조회해야 함 (N≥30)
- DB 쿼리 부하 가능성

**해결:**

- **Phase 2 접근:** submissions 테이블에서 직접 조회 (별도 테이블 불필요)
  - `idx_submissions_problem_solve_time` 인덱스로 최적화
  - 통계 계산은 매번 실행하지만 빈도가 높지 않음
- **향후 최적화:** Redis 캐싱, 통계 사전 계산 (Cron Job)

### 9.2. 뱃지 체크 성능

**과제:**

- 매 제출마다 모든 뱃지 조건을 체크하면 느려질 수 있음

**해결:**

- **Phase 2 접근:** 간단한 조건만 체크 (총 12개 뱃지)
- **향후 최적화:** 뱃지 조건별 인덱싱, 이벤트 기반 체크

### 9.3. 일일 퀘스트 실시간 업데이트

**과제:**

- 앱에서 일일 퀘스트 진행도를 실시간으로 보여줘야 함

**해결:**

- **Phase 2 접근:** 피드백 화면에서 돌아올 때 대시보드 새로고침
- **향후 최적화:** WebSocket 또는 Supabase Realtime 구독

### 9.4. XP 밸런스 조정

**과제:**

- XP 획득량과 레벨업 기준이 적절한지 검증 필요

**해결:**

- **Phase 2 접근:** 시뮬레이션 스크립트 작성 (하루 5/10문제 시나리오)
- 필요시 XP 값 조정: `level_design` 테이블 UPDATE만으로 가능 (코드 재배포 불필요)

### 9.5. 레벨 조회 성능

**과제:**

- 레벨 조회 시 2개 쿼리 필요 (현재 레벨 + 다음 레벨 min_xp)

**해결:**

- **Phase 2 접근:**
  - 2개 쿼리 모두 Primary Key 또는 인덱스 사용 (매우 빠름, < 2ms)
  - `level_design`은 작은 테이블 (수십 개 레코드)
  - 실제 병목은 네트워크 레이턴시 (쿼리 성능은 무시 가능)
- **향후 최적화:**
  - 애플리케이션 레벨에서 `level_design` 전체를 메모리 캐싱 (변경 빈도 낮음)
  - 캐싱 시 max_xp 없이도 다음 레벨 참조로 충분
  - Redis 캐싱, 또는 필요시 `profiles.current_level` 컬럼 추가 (비정규화)

---

## 10. 에러 처리 및 예외 케이스

### 10.1. PSR 계산 실패

**케이스:**

- 문제 또는 유저의 PSR 데이터가 없음

**처리:**

- 초기값 1000으로 처리
- 로그 기록 후 계속 진행

### 10.2. 레벨업 중복 처리

**케이스:**

- 동시에 여러 제출이 처리되어 레벨업이 중복 발생

**처리:**

- DB 트랜잭션 사용
- 또는 레벨업 체크를 동기화 (Lock)

### 10.3. 뱃지 중복 획득

**케이스:**

- 같은 뱃지를 여러 번 획득하려고 시도

**처리:**

- `user_badges` 테이블의 `UNIQUE(user_id, badge_id)` 제약으로 방지
- 앱에서는 `ON CONFLICT DO NOTHING` 처리

### 10.4. 스트릭 계산 오류

**케이스:**

- 타임존 차이로 날짜 계산 오류

**처리:**

- 서버 시간 기준으로 통일 (UTC)
- 또는 사용자 타임존 저장하여 계산 (Phase 3+)

### 10.5. 레벨 조회 실패

**케이스:**

- XP가 음수이거나 `level_design` 테이블에 해당 범위가 없음

**처리:**

- XP가 0 미만이면 0으로 보정
- `level_design` 조회 실패 시 기본값 (Level 1) 반환
- 로그 기록 후 계속 진행

---

## 11. 테스트 계획

### 11.1. 단위 테스트

**GamificationService:**

- [ ] `calculateAndAwardXP()`: 다양한 시나리오 (기본, 미개척, 일일 퀘스트)
- [ ] `checkLevelUp()`: 레벨업 경계 케이스
- [ ] `updatePSR()`: PSR 계산 정확도

**BadgesService:**

- [ ] `checkAndAwardBadges()`: 각 뱃지 조건별 테스트

### 11.2. 통합 테스트

- [ ] 제출 → XP 부여 → 레벨업 전체 플로우
- [ ] 제출 → PSR 변동 → 뱃지 획득 전체 플로우

### 11.3. E2E 테스트

- [ ] 문제 풀이 → 피드백 → 대시보드 확인
- [ ] 레벨업 시나리오
- [ ] 일일 퀘스트 달성 시나리오

---

## 12. 부록

### A. XP 시뮬레이션 스크립트

레벨업 밸런스 검증용:

```typescript
// xp-simulation.ts

// level_design 데이터를 시뮬레이션용으로 하드코딩
const LEVEL_DESIGN = [
  { level: 1, min_xp: 0 },
  { level: 2, min_xp: 750 },
  { level: 3, min_xp: 1500 },
  { level: 4, min_xp: 2250 },
  { level: 5, min_xp: 3000 },
  { level: 6, min_xp: 3750 },
  { level: 7, min_xp: 4500 },
  { level: 8, min_xp: 5250 },
  { level: 9, min_xp: 6150 },
  { level: 10, min_xp: 7200 },
  // ... 필요한 만큼 추가
  { level: 51, min_xp: 999999999 }, // 더미 레벨
];

function getLevelFromXP(xp: number): number {
  // min_xp <= xp인 가장 높은 레벨 찾기
  for (let i = LEVEL_DESIGN.length - 1; i >= 0; i--) {
    if (xp >= LEVEL_DESIGN[i].min_xp) {
      // 다음 레벨 확인 (더미 레벨 제외)
      if (i < LEVEL_DESIGN.length - 1 && LEVEL_DESIGN[i].level < 51) {
        return LEVEL_DESIGN[i].level;
      }
    }
  }
  return 1;
}

function simulateXP(dailyProblems: number, days: number) {
  let xp = 0;

  for (let day = 1; day <= days; day++) {
    const prevLevel = getLevelFromXP(xp);

    // 일일 로그인
    xp += 50;

    // 문제 풀이
    for (let i = 0; i < dailyProblems; i++) {
      xp += 100; // 기본 XP
    }

    // 일일 퀘스트
    if (dailyProblems >= 3) xp += 100;
    if (dailyProblems >= 5) xp += 150;
    if (dailyProblems >= 10) xp += 200;

    // 레벨업 체크
    const currentLevel = getLevelFromXP(xp);
    if (currentLevel > prevLevel) {
      console.log(
        `Day ${day}: Level Up! ${prevLevel} → ${currentLevel} (XP: ${xp})`
      );
    }
  }

  const finalLevel = getLevelFromXP(xp);
  console.log(`\nFinal: Level ${finalLevel}, XP ${xp}`);
}

simulateXP(5, 30); // 하루 5문제, 30일
simulateXP(10, 30); // 하루 10문제, 30일
```

### B. PSR 계산 검증 스크립트

```typescript
// psr-verification.ts
function calculatePSR(
  userPSR: number,
  problemPSR: number,
  pscore: number,
  kFactor: number
) {
  const expected = 1 / (1 + Math.pow(10, (problemPSR - userPSR) / 400));
  const change = Math.round(kFactor * (pscore - expected));
  return { newPSR: userPSR + change, change };
}

// 테스트 케이스
console.log(calculatePSR(1000, 1000, 1.0, 32)); // 예상: +16
console.log(calculatePSR(1000, 1100, 1.0, 32)); // 예상: +22 (어려운 문제 정답)
console.log(calculatePSR(1000, 900, 0.0, 32)); // 예상: -25 (쉬운 문제 오답)
```

### C. Phase 2 체크리스트

**기능 완성도:**

- [ ] 문제 풀이 시 XP 정상 부여
- [ ] 레벨업 정상 작동
- [ ] `level_design` 테이블에서 max_xp 없이 다음 레벨 min_xp로 계산
- [ ] 더미 레벨 51 정상 작동
- [ ] PSR 계산 정확도 85% 이상 (시뮬레이션)
- [ ] submissions 테이블에 PSR 정보 저장 (별도 로그 테이블 제거)
- [ ] psr_change, problem_psr_change 계산으로 처리 (저장 안함)
- [ ] 뱃지 12개 모두 구현
- [ ] 일일 퀘스트 작동
- [ ] 스트릭 계산 정확

**UI/UX:**

- [ ] 대시보드 표시
- [ ] 레벨업 애니메이션
- [ ] 뱃지 컬렉션 표시
- [ ] 피드백 화면에 XP/PSR 표시

**성능:**

- [ ] 레벨 조회 2 쿼리 (현재 + 다음) 성능 확인
- [ ] PSR 계산 2초 이내
- [ ] PSR 히스토리 조회 (submissions + feedbacks JOIN) 성능 확인
- [ ] 풀이 시간 통계 (submissions에서 직접 조회) 성능 확인
- [ ] 뱃지 체크 1초 이내

**안정성:**

- [ ] 에러 핸들링
- [ ] DB 트랜잭션
- [ ] 로그 기록
