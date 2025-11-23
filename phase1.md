# Phase 1: Core Learning Loop - 상세 구현 기획

## 1. 개요

### 1.1. Phase 1 목표

**"AI가 손글씨 풀이를 제대로 분석하고 유의미한 피드백을 줄 수 있는가?"**

Phase 1은 Math Grow의 핵심 가치를 검증하는 단계로, 다음 3가지 플로우를 구현한다:

1. **문제 선택 플로우**: 문제집 페이지 뷰어 → 문제 영역 터치 → 문제 풀이 화면 진입
2. **풀이 제출 플로우**: 손글씨 캔버스 작성 → 답안 입력 → 제출
3. **AI 피드백 플로우**: 3단계 프롬프트 체인 → PScore 분류 → 피드백 생성

### 1.2. 성공 기준

- [ ] 사용자가 문제집 페이지에서 문제를 선택하고 풀이를 제출할 수 있다
- [ ] AI가 손글씨를 85% 이상 정확도로 인식한다
- [ ] AI가 6가지 PScore Case 중 적절한 분류를 수행한다
- [ ] 전체 프로세스가 10초 이내에 완료된다
- [ ] 피드백이 중하위권 학생이 이해 가능한 수준이다

### 1.3. 제외 사항

- 레벨/PSR 시스템 (Phase 2)
- 뱃지 시스템 (Phase 2)
- 내 서재 UX (Phase 3)
- 여러 문제집 지원 (Phase 1에서는 단일 문제집만)

---

## 2. 시스템 아키텍처

### 2.1. 전체 구조

```
┌─────────────────────────────────────────────────────────┐
│                    React Native App (태블릿)             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ 페이지 뷰어  │→│ 문제 풀이 뷰  │→│ AI 피드백 뷰  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│                      NestJS Backend                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Auth Module  │  │ Problem API  │  │ AI Pipeline  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│                   Supabase (PostgreSQL)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │    Tables    │  │    Storage   │  │     Auth     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│                      Gemini API                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   STEP 1     │→│   STEP 2     │→│   STEP 3     │  │
│  │     OCR      │  │   PScore     │  │  Feedback    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 2.2. 데이터 플로우

#### 문제 선택 플로우

```
[App] GET /books/:bookId/pages
  ↓
[DB] pages 테이블 조회 → page_image_url 반환
  ↓
[App] 페이지 이미지 표시 + bbox 영역 오버레이
  ↓
[User] 문제 영역 터치
  ↓
[App] bbox 기반으로 이미지 크롭 (클라이언트)
  ↓
[App] 문제 풀이 화면 진입
```

#### 풀이 제출 플로우

```
[User] 손글씨 작성 + 답안 입력 + 제출
  ↓
[App] 손글씨 캔버스 → Base64 이미지 변환
  ↓
[App] POST /submissions
  ↓
[Backend] 손글씨 이미지 → Supabase Storage 업로드
  ↓
[Backend] submission 레코드 생성 (status: pending)
  ↓
[Backend] AI 파이프라인 트리거
  ↓
[Backend] STEP 1: OCR (문제 이미지 + 풀이 이미지)
  ↓
[Backend] STEP 2: PScore 분류
  ↓
[Backend] STEP 3: 피드백 생성
  ↓
[Backend] feedback 레코드 생성
  ↓
[Backend] submission 상태 업데이트 (status: completed)
  ↓
[App] 피드백 조회 GET /submissions/:id/feedback
  ↓
[App] 피드백 화면 표시
```

---

## 3. 데이터베이스 설계

### 3.1. 마이그레이션 전략 및 관리

**서버가 DB 스키마의 단일 소스 (Single Source of Truth)**

Phase 1부터 **NestJS 서버가 모든 DB 스키마 및 마이그레이션을 관리**합니다.

#### 3.1.1. 마이그레이션 관리 방식

**Supabase CLI + Prisma 조합 (추천 방식):**

```
backend/ (NestJS)
  ├── supabase/
  │   └── migrations/
  │       ├── 001_initial_schema.sql       # 기본 테이블 (books, chapters, problems)
  │       └── 002_phase1_additions.sql     # Phase 1 추가 (pages, profiles, submissions, feedbacks)
  ├── prisma/
  │   └── schema.prisma                    # DB → 자동 생성 (타입 생성용)
  └── src/
      ├── books/
      ├── problems/
      └── ...
```

**워크플로우:**

1. **SQL 마이그레이션 작성** (Supabase CLI)

   ```bash
   npx supabase migration new 001_initial_schema
   # supabase/migrations/001_initial_schema.sql 생성
   ```

2. **로컬 DB 적용**

   ```bash
   npx supabase db reset  # 마이그레이션 전체 재적용시 실행
   npx supabase migration up # 마이그레이션 증분 적용시 실행
   ```

3. **Prisma 타입 동기화**

   ```bash
   npx prisma db pull      # DB 스키마 → prisma/schema.prisma
   npx prisma generate     # TypeScript 타입 생성
   ```

4. **서버 코드에서 사용**

   ```typescript
   import { PrismaClient } from '@prisma/client';
   const prisma = new PrismaClient();

   const problems = await prisma.problem.findMany({
     include: { page: true },
   });
   ```

**프로덕션 배포:**

```bash
npx supabase db push  # 원격 Supabase 프로젝트에 마이그레이션 적용
```

#### 3.1.2. 마이그레이션 파일 네이밍

- 형식: `NNN_description.sql` (예: `001_initial_schema.sql`)
- 순차적 증가: 001, 002, 003, ...
- 명확한 설명: 무엇을 추가/수정하는지 명시

#### 3.1.3. 문제 업로더(본 기획에는 없으며, 별도의 프로젝트임)와의 관계

**문제 업로더는 서버 API에만 의존:**

- DB 스키마 직접 접근 ❌
- 서버 API 호출로 데이터 업로드 ✅
- 예: `POST /admin/books`, `POST /admin/problems`

**장점:**

- 스키마 변경 시 서버 API만 맞으면 됨
- 관심사 분리 (문제 업로더 = 데이터 입력, 서버 = 데이터 관리)

### 3.2. Phase 1 마이그레이션 파일

Phase 1에서는 **2개의 마이그레이션 파일**을 생성합니다:

- `001_initial_schema.sql`: 기본 테이블 (books, chapters, problems)
- `002_phase1_additions.sql`: Phase 1 추가 테이블 및 수정

#### 3.2.1. 001_initial_schema.sql (통합 초기 스키마)

**주의:** 이 파일은 문제 업로더의 001~006 마이그레이션을 모두 통합한 **최종 초기 스키마**입니다. 서버는 이 파일로 처음 DB를 생성합니다.

```sql
-- ============================================================================
-- Migration: 001 - Initial Schema (Consolidated from uploader migrations 001-006)
-- Description: 문제집, 단원, 페이지, 문제 테이블 및 Storage 초기 설정
-- ============================================================================

-- ============================================================================
-- TABLES
-- ============================================================================

-- 문제집 정보 (작업 현황 관리)
CREATE TABLE IF NOT EXISTS books (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    publisher VARCHAR(100),
    published_year INT,

    -- 작업 상태 관리
    status VARCHAR(20) DEFAULT 'processing', -- processing, converting, analyzing, done, error
    total_pages INT,
    last_processed_page INT DEFAULT 0,

    -- 에러 처리 (문제 업로더용)
    error_message TEXT,
    failed_at_page INT,
    failed_analysis_pages INT[] DEFAULT ARRAY[]::INT[],

    -- 원본 PDF 위치
    original_pdf_url TEXT,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 단원 정보 (독립적인 계층 구조, book과 무관)
CREATE TABLE IF NOT EXISTS chapters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES chapters(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    order_index INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 페이지 정보 (books → pages → problems 계층)
CREATE TABLE IF NOT EXISTS pages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    book_id UUID REFERENCES books(id) ON DELETE CASCADE,
    page_number INT NOT NULL,
    page_image_url TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(book_id, page_number)
);

-- 문제 정보
CREATE TABLE IF NOT EXISTS problems (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID REFERENCES pages(id) ON DELETE CASCADE,

    -- 메타 데이터
    problem_number VARCHAR(20),
    problem_type VARCHAR(20) DEFAULT 'multiple_choice' NOT NULL
        CHECK (problem_type IN ('multiple_choice', 'subjective', 'true_false')),

    -- 텍스트 데이터
    problem_text TEXT NOT NULL,
    options JSONB,
    answer VARCHAR(255),
    solution_text TEXT,

    -- 단원 정보 (N:M 관계, 최대 3개)
    chapter_ids JSONB DEFAULT '[]'::jsonb,

    -- 이미지 관련
    requires_image BOOLEAN DEFAULT false,
    problem_image_url TEXT,
    solution_image_url TEXT,
    raw_bbox JSONB,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================================
-- VIEWS
-- ============================================================================

-- Leaf 단원 조회용 뷰
CREATE VIEW leaf_chapters AS
SELECT c.id, c.name, c.description
FROM chapters c
WHERE NOT EXISTS (
    SELECT 1 FROM chapters
    WHERE parent_id = c.id
);

-- ============================================================================
-- INDEXES
-- ============================================================================

CREATE INDEX idx_books_status ON books(status);
CREATE INDEX idx_chapters_parent_id ON chapters(parent_id);
CREATE INDEX idx_chapters_name ON chapters(name);
CREATE INDEX idx_pages_book_id ON pages(book_id);
CREATE INDEX idx_problems_page_id ON problems(page_id);
CREATE INDEX idx_problems_chapter_ids ON problems USING GIN (chapter_ids);
CREATE INDEX idx_problems_requires_image ON problems(requires_image);
CREATE INDEX idx_problems_problem_type ON problems(problem_type);

-- ============================================================================
-- TRIGGERS
-- ============================================================================

-- 1. Chapter IDs 검증 (존재 여부 + Leaf 노드만 허용)
CREATE OR REPLACE FUNCTION validate_chapter_ids()
RETURNS TRIGGER AS $$
DECLARE
  chapter_record JSONB;
  chapter_id_value UUID;
BEGIN
  IF NEW.chapter_ids IS NULL THEN
    RETURN NEW;
  END IF;

  FOR chapter_record IN
    SELECT * FROM jsonb_array_elements(NEW.chapter_ids)
  LOOP
    chapter_id_value := (chapter_record->>'id')::UUID;

    -- Chapter 존재 여부 확인
    IF NOT EXISTS (SELECT 1 FROM chapters WHERE id = chapter_id_value) THEN
      RAISE EXCEPTION 'Invalid chapter_id: %. Chapter does not exist.', chapter_id_value;
    END IF;

    -- Leaf 노드인지 확인
    IF EXISTS (SELECT 1 FROM chapters WHERE parent_id = chapter_id_value) THEN
      RAISE EXCEPTION 'Chapter % is not a leaf node. Only leaf nodes can be assigned to problems.', chapter_id_value;
    END IF;
  END LOOP;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_chapter_ids_validity
BEFORE INSERT OR UPDATE ON problems
FOR EACH ROW
EXECUTE FUNCTION validate_chapter_ids();

-- 2. 최대 3개 Chapter 제약
CREATE OR REPLACE FUNCTION enforce_max_3_chapters()
RETURNS TRIGGER AS $$
DECLARE
  chapter_count INT;
BEGIN
  IF NEW.chapter_ids IS NULL THEN
    RETURN NEW;
  END IF;

  chapter_count := jsonb_array_length(NEW.chapter_ids);

  IF chapter_count > 3 THEN
    RAISE EXCEPTION 'A problem can have at most 3 chapters. Current: %', chapter_count;
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_max_3_chapters
BEFORE INSERT OR UPDATE ON problems
FOR EACH ROW
EXECUTE FUNCTION enforce_max_3_chapters();

-- 3. Chapter 삭제 방지 (문제에서 참조 중인 경우)
CREATE OR REPLACE FUNCTION prevent_chapter_delete_if_referenced()
RETURNS TRIGGER AS $$
DECLARE
  problem_count INT;
BEGIN
  SELECT COUNT(*) INTO problem_count
  FROM problems
  WHERE chapter_ids @> jsonb_build_array(
    jsonb_build_object('id', OLD.id::text)
  );

  IF problem_count > 0 THEN
    RAISE EXCEPTION 'Cannot delete chapter %. It is referenced by % problem(s).', OLD.name, problem_count;
  END IF;

  RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER prevent_referenced_chapter_delete
BEFORE DELETE ON chapters
FOR EACH ROW
EXECUTE FUNCTION prevent_chapter_delete_if_referenced();

-- 4. Chapters updated_at 자동 업데이트
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_chapters_updated_at
BEFORE UPDATE ON chapters
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();

-- ============================================================================
-- ROW LEVEL SECURITY (RLS) - TABLES
-- ============================================================================

ALTER TABLE books ENABLE ROW LEVEL SECURITY;
ALTER TABLE chapters ENABLE ROW LEVEL SECURITY;
ALTER TABLE pages ENABLE ROW LEVEL SECURITY;
ALTER TABLE problems ENABLE ROW LEVEL SECURITY;

-- 공개 읽기
CREATE POLICY "Allow public read access" ON books FOR SELECT USING (true);
CREATE POLICY "Allow public read access" ON chapters FOR SELECT USING (true);
CREATE POLICY "Allow public read access" ON pages FOR SELECT USING (true);
CREATE POLICY "Allow public read access" ON problems FOR SELECT USING (true);

-- 인증된 사용자만 CUD (관리자용)
CREATE POLICY "Allow authenticated users to insert" ON books FOR INSERT WITH CHECK (auth.role() = 'authenticated');
CREATE POLICY "Allow authenticated users to update" ON books FOR UPDATE USING (auth.role() = 'authenticated');
CREATE POLICY "Allow authenticated users to delete" ON books FOR DELETE USING (auth.role() = 'authenticated');

CREATE POLICY "Allow authenticated users to insert chapters" ON chapters FOR INSERT WITH CHECK (auth.role() = 'authenticated');
CREATE POLICY "Allow authenticated users to update chapters" ON chapters FOR UPDATE USING (auth.role() = 'authenticated');
CREATE POLICY "Allow authenticated users to delete chapters" ON chapters FOR DELETE USING (auth.role() = 'authenticated');

CREATE POLICY "Allow authenticated users to insert" ON problems FOR INSERT WITH CHECK (auth.role() = 'authenticated');
CREATE POLICY "Allow authenticated users to update" ON problems FOR UPDATE USING (auth.role() = 'authenticated');
CREATE POLICY "Allow authenticated users to delete" ON problems FOR DELETE USING (auth.role() = 'authenticated');

-- ============================================================================
-- STORAGE BUCKETS
-- ============================================================================

INSERT INTO storage.buckets (id, name, public)
VALUES
    ('original-pdfs', 'original-pdfs', false),
    ('raw-pages', 'raw-pages', true),
    ('final-problems', 'final-problems', true)
ON CONFLICT (id) DO NOTHING;

-- ============================================================================
-- ROW LEVEL SECURITY (RLS) - STORAGE
-- ============================================================================

-- original-pdfs: 인증된 사용자만 접근
CREATE POLICY "Authenticated users can upload PDFs"
ON storage.objects FOR INSERT TO authenticated
WITH CHECK (bucket_id = 'original-pdfs');

CREATE POLICY "Authenticated users can read PDFs"
ON storage.objects FOR SELECT TO authenticated
USING (bucket_id = 'original-pdfs');

CREATE POLICY "Authenticated users can delete PDFs"
ON storage.objects FOR DELETE TO authenticated
USING (bucket_id = 'original-pdfs');

-- raw-pages: 인증된 사용자만 접근
CREATE POLICY "Authenticated users can upload raw pages"
ON storage.objects FOR INSERT TO authenticated
WITH CHECK (bucket_id = 'raw-pages');

CREATE POLICY "Authenticated users can read raw pages"
ON storage.objects FOR SELECT TO authenticated
USING (bucket_id = 'raw-pages');

CREATE POLICY "Authenticated users can delete raw pages"
ON storage.objects FOR DELETE TO authenticated
USING (bucket_id = 'raw-pages');

-- final-problems: 공개 읽기, 인증된 사용자만 쓰기
CREATE POLICY "Public read access for final problems"
ON storage.objects FOR SELECT TO public
USING (bucket_id = 'final-problems');

CREATE POLICY "Authenticated users can upload final problems"
ON storage.objects FOR INSERT TO authenticated
WITH CHECK (bucket_id = 'final-problems');

CREATE POLICY "Authenticated users can update final problems"
ON storage.objects FOR UPDATE TO authenticated
USING (bucket_id = 'final-problems');

CREATE POLICY "Authenticated users can delete final problems"
ON storage.objects FOR DELETE TO authenticated
USING (bucket_id = 'final-problems');

-- ============================================================================
-- COMMENTS
-- ============================================================================

COMMENT ON COLUMN books.status IS 'processing(초기), converting(이미지 변환 중), analyzing(AI 분석 중), done(완료), error(이미지 변환 실패)';
COMMENT ON COLUMN books.error_message IS '에러 발생 시 에러 메시지 저장';
COMMENT ON COLUMN books.failed_at_page IS '이미지 변환 실패한 페이지 번호';
COMMENT ON COLUMN books.failed_analysis_pages IS 'AI 분석 실패한 페이지 번호 목록';
COMMENT ON COLUMN problems.problem_type IS 'Type of problem: multiple_choice (객관식), subjective (주관식), true_false (진위형)';
COMMENT ON COLUMN problems.chapter_ids IS 'JSONB array of chapter objects: [{"id": "uuid"}, ...]. Max 3 items.';
COMMENT ON COLUMN problems.requires_image IS '문제 이미지가 필수인지 여부 (도형, 그래프 등 포함)';
```

**주요 변경사항 (문제 업로더 001-006 통합):**

- ✅ `pages` 테이블 추가 (books → pages → problems 계층)
- ✅ `problems.page_id` 사용 (기존 `book_id`, `original_page` 제거)
- ✅ `chapters` 테이블 독립화 (book_id 제거, 독립적인 교육과정 Tree)
- ✅ `problems.chapter_id` → `chapter_ids` (JSONB, N:M 관계)
- ✅ `problems.keywords` 제거 (chapter_ids로 대체)
- ✅ `problems.requires_image` 추가 (문제 이미지 필수 여부)
- ✅ `problems.problem_type` 추가 (객관식/주관식/진위형)
- ✅ `books` 테이블에 에러 처리 필드 추가
- ✅ Triggers 추가 (chapter 검증, 최대 3개 제약, 삭제 방지)

#### 3.2.2. 002_phase1_additions.sql (Phase 1 추가)

Phase 1 학습 루프에 필요한 테이블 및 Storage 버킷을 추가합니다.

```sql
-- ============================================================================
-- Migration: 002 - Phase 1 Additions
-- Description: 사용자, 제출, 피드백 테이블 및 Storage 버킷
-- ============================================================================

-- 사용자 프로필 (Supabase Auth 확장)
CREATE TABLE IF NOT EXISTS profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    nickname VARCHAR(50) NOT NULL,
    grade INT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_profiles_id ON profiles(id);

-- 풀이 제출
CREATE TABLE IF NOT EXISTS submissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
    problem_id UUID REFERENCES problems(id) ON DELETE CASCADE,

    solution_image_url TEXT NOT NULL,
    user_answer VARCHAR(255),

    status VARCHAR(20) DEFAULT 'pending',

    started_at TIMESTAMPTZ,
    submitted_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_submissions_user_id ON submissions(user_id);
CREATE INDEX idx_submissions_problem_id ON submissions(problem_id);
CREATE INDEX idx_submissions_status ON submissions(status);

-- AI 피드백
CREATE TABLE IF NOT EXISTS feedbacks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submission_id UUID REFERENCES submissions(id) ON DELETE CASCADE,

    pscore_case INT NOT NULL,
    pscore_value DECIMAL(3,2) NOT NULL,
    is_correct BOOLEAN NOT NULL,

    ocr_result JSONB,
    readability_score DECIMAL(3,2),
    analysis_result JSONB,
    feedback_json JSONB NOT NULL,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_feedbacks_submission_id ON feedbacks(submission_id);

-- RLS
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE submissions ENABLE ROW LEVEL SECURITY;
ALTER TABLE feedbacks ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile"
ON profiles FOR SELECT
USING (auth.uid() = id);

CREATE POLICY "Users can update own profile"
ON profiles FOR UPDATE
USING (auth.uid() = id);

CREATE POLICY "Users can insert own profile"
ON profiles FOR INSERT
WITH CHECK (auth.uid() = id);

CREATE POLICY "Users can view own submissions"
ON submissions FOR SELECT
USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own submissions"
ON submissions FOR INSERT
WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own submissions"
ON submissions FOR UPDATE
USING (auth.uid() = user_id);

CREATE POLICY "Users can view own feedback"
ON feedbacks FOR SELECT
USING (
    EXISTS (
        SELECT 1 FROM submissions
        WHERE submissions.id = feedbacks.submission_id
        AND submissions.user_id = auth.uid()
    )
);

-- Storage Bucket for user solutions
INSERT INTO storage.buckets (id, name, public)
VALUES ('user-solutions', 'user-solutions', false)
ON CONFLICT (id) DO NOTHING;

-- Storage RLS policies
CREATE POLICY "Users can upload own solutions"
ON storage.objects FOR INSERT
TO authenticated
WITH CHECK (
    bucket_id = 'user-solutions'
    AND auth.uid()::text = (storage.foldername(name))[1]
);

CREATE POLICY "Users can read own solutions"
ON storage.objects FOR SELECT
TO authenticated
USING (
    bucket_id = 'user-solutions'
    AND auth.uid()::text = (storage.foldername(name))[1]
);

CREATE POLICY "Users can delete own solutions"
ON storage.objects FOR DELETE
TO authenticated
USING (
    bucket_id = 'user-solutions'
    AND auth.uid()::text = (storage.foldername(name))[1]
);
```

**추가된 테이블:**

- ✅ `profiles`: 사용자 프로필 (nickname, grade)
- ✅ `submissions`: 풀이 제출 (solution_image_url, status)
- ✅ `feedbacks`: AI 피드백 (pscore, feedback_json)
- ✅ `user-solutions` Storage 버킷

**Storage 구조:**

```
user-solutions/
  └── {user_id}/
      └── {submission_id}.png
```

### 3.3. 최종 프로젝트 구조

```
backend/ (NestJS)
  ├── supabase/
  │   ├── config.toml
  │   └── migrations/
  │       ├── 001_initial_schema.sql
  │       └── 002_phase1_additions.sql
  ├── prisma/
  │   ├── schema.prisma              # 자동 생성 (prisma db pull)
  │   └── .gitignore
  ├── src/
  │   ├── main.ts
  │   ├── app.module.ts
  │   ├── auth/
  │   ├── books/
  │   ├── problems/
  │   ├── submissions/
  │   ├── feedbacks/
  │   ├── ai/
  │   └── common/
  ├── package.json
  └── .env

problem-uploader/
  ├── src/
  │   └── api-client.ts             # 서버 API 호출
  └── package.json
```

**개발 시작 명령어:**

```bash
# 1. 로컬 Supabase 시작
npx supabase start

# 2. 마이그레이션 적용
npx supabase db reset

# 3. Prisma 타입 생성
npx prisma db pull
npx prisma generate

# 4. NestJS 서버 시작
npm run start:dev
```

### 3.3. 데이터 관계도

```
books (1) ──< (N) pages (1) ──< (N) problems
chapters (tree) ──< (N:M) problems (via chapter_ids JSONB)

profiles (1) ──< (N) submissions
problems (1) ──< (N) submissions
submissions (1) ──< (1) feedbacks
```

**주요 특징:**

- ✅ `books → pages → problems` 계층 구조 (물리적 페이지 관계)
- ✅ `chapters`는 독립적인 Tree 구조 (교육과정 체계, book과 무관)
- ✅ `problems ─ chapters`: N:M 관계 (하나의 문제가 최대 3개 단원에 속함)
- 문제에서 책 정보가 필요하면 `problems → pages → books` JOIN
- 문제에서 단원 정보가 필요하면 `chapter_ids` JSONB 배열 사용

---

## 4. API 설계

### 4.1. 인증 API

#### POST /auth/signup

사용자 회원가입 (Supabase Auth 활용)

**Request:**

```json
{
  "email": "student@example.com",
  "password": "password123",
  "nickname": "수학왕",
  "grade": 2
}
```

**Response:**

```json
{
  "user": {
    "id": "uuid",
    "email": "student@example.com"
  },
  "session": {
    "access_token": "jwt_token",
    "refresh_token": "jwt_token"
  }
}
```

**구현:**

1. Supabase Auth의 `signUp()` 호출
2. `profiles` 테이블에 nickname, grade 저장

#### POST /auth/login

로그인

**Request:**

```json
{
  "email": "student@example.com",
  "password": "password123"
}
```

**Response:**

```json
{
  "user": {
    "id": "uuid",
    "email": "student@example.com"
  },
  "session": {
    "access_token": "jwt_token",
    "refresh_token": "jwt_token"
  },
  "profile": {
    "nickname": "수학왕",
    "grade": 2
  }
}
```

#### GET /auth/me

현재 사용자 정보 조회

**Headers:**

```
Authorization: Bearer {access_token}
```

**Response:**

```json
{
  "id": "uuid",
  "email": "student@example.com",
  "nickname": "수학왕",
  "grade": 2
}
```

---

### 4.2. 문제집 & 문제 API

#### GET /books/:bookId

문제집 정보 조회 (Phase 1에서는 단일 문제집)

**Response:**

```json
{
  "id": "uuid",
  "title": "수학의 정석 수학 (하)",
  "publisher": "성지출판",
  "published_year": 2023,
  "status": "done",
  "total_pages": 320
}
```

#### GET /books/:bookId/pages

문제집의 페이지 목록 조회

**Query Parameters:**

- `page`: 페이지 번호 (선택사항, 특정 페이지만 조회)
- `limit`: 한 번에 불러올 페이지 수 (기본: 10)
- `offset`: 시작 오프셋 (페이징용)

**Response:**

```json
{
  "book_id": "uuid",
  "total_pages": 320,
  "pages": [
    {
      "id": "uuid",
      "page_number": 1,
      "page_image_url": "https://supabase.co/storage/raw-pages/book-id/page-1.png"
    },
    {
      "id": "uuid",
      "page_number": 2,
      "page_image_url": "https://supabase.co/storage/raw-pages/book-id/page-2.png"
    }
  ]
}
```

#### GET /pages/:pageId/problems

특정 페이지의 문제 목록 및 bbox 조회

**Response:**

```json
{
  "page_id": "uuid",
  "page_number": 15,
  "problems": [
    {
      "id": "uuid",
      "problem_number": "1",
      "raw_bbox": {
        "problem": { "x1": 100, "y1": 200, "x2": 500, "y2": 400 },
        "solution": { "x1": 100, "y1": 450, "x2": 500, "y2": 700 }
      },
      "answer": "3",
      "options": [
        { "num": 1, "text": "1" },
        { "num": 2, "text": "2" },
        { "num": 3, "text": "3" },
        { "num": 4, "text": "4" },
        { "num": 5, "text": "5" }
      ]
    },
    {
      "id": "uuid",
      "problem_number": "2",
      "raw_bbox": {
        "problem": { "x1": 100, "y1": 750, "x2": 500, "y2": 950 }
      },
      "answer": "12",
      "options": null
    }
  ]
}
```

#### GET /problems/:problemId

문제 상세 조회

**Response:**

```json
{
  "id": "uuid",
  "page_id": "uuid",
  "problem_number": "1",
  "problem_type": "multiple_choice",
  "problem_text": "다음 방정식을 풀어라. $x^2 - 5x + 6 = 0$",
  "options": [
    { "num": 1, "text": "x = 1, 2" },
    { "num": 2, "text": "x = 2, 3" },
    { "num": 3, "text": "x = 3, 4" },
    { "num": 4, "text": "x = 4, 5" },
    { "num": 5, "text": "x = 5, 6" }
  ],
  "answer": "2",
  "chapter_ids": [
    { "id": "uuid-1" },
    { "id": "uuid-2" }
  ],
  "requires_image": false,
  "problem_image_url": "https://supabase.co/storage/final-problems/problem-uuid.png",
  "solution_image_url": "https://supabase.co/storage/final-problems/solution-uuid.png",
  "raw_bbox": {
    "problem": { "x1": 100, "y1": 200, "x2": 500, "y2": 400 },
    "solution": { "x1": 100, "y1": 450, "x2": 500, "y2": 700 }
  },
  "page": {
    "id": "uuid",
    "page_number": 15,
    "book_id": "uuid"
  }
}
```

**주요 변경:**

- ✅ `book_id`, `original_page` 제거
- ✅ `page_id` 추가 (pages 테이블 참조)
- ✅ `chapter_id` → `chapter_ids` (JSONB 배열, N:M 관계)
- ✅ `keywords` 제거 (chapter_ids로 대체)
- ✅ `problem_type` 추가 (multiple_choice, subjective, true_false)
- ✅ `requires_image` 추가 (문제 이미지 필수 여부)
- ✅ `page` 객체 추가 (페이지 정보 포함, 선택사항: JOIN 결과)

---

### 4.3. 풀이 제출 API

#### POST /submissions

풀이 제출

**Headers:**

```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request:**

```json
{
  "problem_id": "uuid",
  "solution_image_base64": "data:image/png;base64,iVBORw0KG...",
  "user_answer": "2",
  "started_at": "2025-11-18T10:30:00Z"
}
```

**Response:**

```json
{
  "submission_id": "uuid",
  "status": "pending",
  "submitted_at": "2025-11-18T10:35:00Z"
}
```

**처리 흐름:**

1. Base64 이미지 디코딩
2. Supabase Storage에 업로드 (`user-solutions/{user_id}/{submission_id}.png`)
3. `submissions` 테이블에 레코드 생성 (status: pending)
4. **비동기로** AI 파이프라인 트리거 (Background Job)
5. 즉시 submission_id 반환

#### GET /submissions/:submissionId

제출 상태 조회

**Response:**

```json
{
  "id": "uuid",
  "status": "completed", // pending, processing, completed, failed
  "submitted_at": "2025-11-18T10:35:00Z",
  "problem_id": "uuid"
}
```

#### GET /submissions/:submissionId/feedback

피드백 조회

**Response:**

```json
{
  "submission_id": "uuid",
  "pscore_case": 2,
  "pscore_value": 0.7,
  "is_correct": false,
  "feedback": {
    "summary": {
      "emoji": "😊",
      "title": "아까운 실수!",
      "subtitle": "풀이는 완벽했는데, 마지막에 부호 실수가 있었어요"
    },
    "strengths": [
      {
        "title": "인수분해 정확",
        "detail": "x² - 5x + 6을 (x-2)(x-3)으로 완벽하게 인수분해했어요. 이차방정식의 인수분해 공식을 잘 이해하고 있네요!"
      }
    ],
    "improvements": [
      {
        "title": "부호 실수",
        "what_went_wrong": "x - 3 = 0에서 x = -3으로 계산했어요",
        "why_wrong": "x - 3 = 0을 풀 때, 양변에 3을 더하면 x = 3이 되어야 해요",
        "how_to_fix": "x - a = 0 형태에서는 x = a가 됩니다",
        "severity": "low"
      }
    ],
    "concept_explanation": {
      "concept_name": "이차방정식의 해 구하기",
      "explanation": "인수분해로 (x-a)(x-b) = 0 형태가 되면, x = a 또는 x = b가 해가 됩니다. 이는 곱셈이 0이 되려면 둘 중 하나는 0이어야 한다는 원리를 이용한 것입니다.",
      "example": "예를 들어, (x-5)(x+2)=0이면 x-5=0 또는 x+2=0이므로 x=5 또는 x=-2입니다."
    },
    "correct_solution": {
      "title": "올바른 풀이 과정",
      "steps": [
        { "step": 1, "content": "x² - 5x + 6 = 0", "explanation": "주어진 식" },
        {
          "step": 2,
          "content": "(x - 2)(x - 3) = 0",
          "explanation": "인수분해"
        },
        {
          "step": 3,
          "content": "x - 2 = 0 또는 x - 3 = 0",
          "explanation": "각 인수를 0으로"
        },
        { "step": 4, "content": "x = 2 또는 x = 3", "explanation": "최종 답" }
      ]
    },
    "next_steps": {
      "advice": "부호 실수를 줄이려면, 풀이 과정을 한 번 더 검토하세요",
      "practice_topics": ["이차방정식의 해", "인수분해 확인"],
      "difficulty_adjust": "similar"
    }
  }
}
```

**에러 처리:**

- 제출이 아직 처리 중이면 `{ "status": "processing" }` 반환
- 제출이 실패하면 `{ "status": "failed", "error": "..." }` 반환

---

### 4.4. AI 파이프라인 (내부 모듈)

AI 파이프라인은 백엔드 내부 모듈로 구현되며, 직접적인 API 엔드포인트는 없음.

#### 4.4.1. STEP 1: OCR 및 구조화

**입력:**

- 문제 정보 (DB에서 이미 추출된 텍스트)
- 문제 이미지 URL (DB의 `requires_image` 플래그가 true인 경우만 전달)
- 손글씨 풀이 이미지 URL

**Gemini API 프롬프트:**

```
당신은 고등학교 수학 문제 풀이를 분석하는 전문가입니다.
다음 정보가 주어집니다:
- 문제 내용: {problem_text} (이미 텍스트로 변환됨)
- [requires_image=true인 경우만] 문제 이미지 (도형, 그래프 등이 포함됨)
- 학생의 손글씨 풀이 이미지

작업:
1. 학생의 손글씨 풀이를 단계별로 읽고 구조화된 형태로 추출하세요.
2. 이미지 가독성을 평가하세요 (0.0 ~ 1.0).

주의사항:
- 학생이 적은 내용을 절대 수정하지 말고 있는 그대로 추출하세요.
- 읽을 수 없는 부분은 [읽을 수 없음]으로 표시하세요.
- 문제 이미지가 제공된 경우, 도형/그래프 정보를 참고하여 학생의 풀이를 이해하세요.

출력 형식 (JSON):
{
  "solution_steps": [
    {"step": 1, "content": "학생이 적은 내용", "is_readable": true},
    {"step": 2, "content": "...", "is_readable": true}
  ],
  "readability": 0.8,
  "has_work": true,
  "work_completeness": 0.8
}
```

**출력:**

- OCR 결과 JSON
- 가독성 점수가 0.5 미만이면 조기 실패 처리

**참고:** `requires_image` 플래그는 문제 이미지가 풀이에 필수적인지 판단합니다 (도형, 그래프 등). false인 경우 문제 이미지를 AI에 전달하지 않아 토큰 비용을 절약합니다.

#### 4.4.2. STEP 2: 논리 분석 및 PScore 분류

**입력:**

- STEP 1의 OCR 결과
- 학생의 답안
- 문제 정보 (텍스트)
- 문제 이미지 (DB의 `requires_image` 플래그가 true인 경우만 전달)
- 문제의 정답
- 문제의 해설 (있는 경우)

**Gemini API 프롬프트:**

```
당신은 고등학교 수학 풀이를 논리적으로 검증하는 전문가입니다.
다음 정보가 주어집니다:
- 문제 정보: {problem_text}
- [requires_image=true인 경우만] 문제 이미지 (도형, 그래프 등이 포함됨)
- 학생의 풀이 과정: {solution_steps}
- 학생의 답안: {user_answer}
- 정답: {correct_answer}
- 참고 해설: {solution_text} (제공된 경우만, 이는 참고용이며 가장 좋은 풀이 방법이 아닐 수 있습니다. 학생의 풀이가 해설과 다르더라도 논리적으로 타당하다면 인정해주세요.)

작업:
1. 학생이 사용한 개념이 올바른지 분석하세요.
2. 계산 과정에서 오류를 찾으세요 (단계별).
3. 논리적 연결성을 평가하세요.
4. 다음 결정 트리를 따라 PScore Case를 분류하세요:

[결정 트리]
- has_work == false OR work_completeness < 0.1? → Case 6 (백지, 0.0)
- 최종 답안 정답?
  - YES
    - 개념 적용 올바름?
      - YES
        - 풀이 효율적? → YES: Case 1 (완벽, 1.0) / NO: Case 3 (돌아온 길, 0.6)
      - NO → Case 5 (우연, 0.2)
  - NO
    - 개념 적용 올바름?
      - YES
        - 단순 계산 실수만? → YES: Case 2 (아차 실수, 0.7)
      - NO → Case 4 (개념 부족, 0.3)

출력 형식 (JSON):
{
  "pscore_case": 2,
  "pscore_value": 0.7,
  "is_correct": false,
  "concept_analysis": {
    "used_concepts": ["인수분해"],
    "correct_concepts": ["인수분해"],
    "incorrect_concepts": [],
    "concept_score": 0.95
  },
  "calculation_analysis": {
    "errors": [
      {
        "step": 3,
        "type": "sign_error",
        "detail": "x - 3 = 0에서 x = -3으로 잘못 계산",
        "severity": "low"
      }
    ]
  },
  "logic_analysis": {
    "flow": "coherent",
    "completeness": 0.9
  },
  "key_strengths": ["인수분해 정확", "단계 명확"],
  "key_weaknesses": ["부호 실수"],
  "confidence": 0.95
}
```

**출력:**

- PScore 분류 결과 JSON

#### 4.4.3. STEP 3: 피드백 생성

**입력:**

- STEP 2의 분석 결과

**Gemini API 프롬프트:**

```
당신은 중하위권 고등학생을 위한 따뜻하고 전문적인 수학 튜터입니다.
다음 분석 결과를 바탕으로 교육적이고 격려적인 피드백을 생성하세요.

분석 결과: {analysis_result}

피드백 원칙:
- 톤: 친근하고 격려적 ("~해요" 말투)
- 구조: 전체 요약 → 잘한 점 → 개선할 점 → 개념 설명 → 올바른 풀이 → 다음 단계
- 길이: 요약 30자, 잘한 점 각 80자, 개선점 각 150자, 개념 설명 200자
- 긍정 우선: 잘한 점을 먼저 언급 (Case 6 제외)

PScore Case별 전략:
- Case 1 (완벽): 축하 + 더 어려운 문제 추천
- Case 2 (아차 실수): 격려 + 실수 방지 팁
- Case 3 (돌아온 길): 인정 + 더 효율적인 방법 소개
- Case 4 (개념 부족): 개념 재설명 + 기초 복습 권장
- Case 5 (우연): 논리적 풀이 중요성 강조 + 체계적 풀이 안내
- Case 6 (백지): 따뜻한 격려 + 쉬운 문제부터 시작 권장

출력 형식: [4.3 API의 feedback JSON과 동일]
```

**출력:**

- 최종 피드백 JSON

---

## 5. 앱 화면 및 기능 (React Native - 태블릿 타겟)

### 5.1. 화면 구조

```
App
├── AuthStack (인증 전)
│   ├── LoginScreen
│   └── SignupScreen
└── MainStack (인증 후)
    ├── PageViewerScreen (문제집 페이지 뷰어)
    ├── ProblemSolvingScreen (문제 풀이)
    └── FeedbackScreen (AI 피드백)
```

### 5.2. 온보딩 화면

#### 5.2.1. LoginScreen

- 이메일/비밀번호 입력
- 로그인 버튼
- 회원가입 링크

**기술 구현:**

- Supabase Auth의 `signInWithPassword()` 사용
- JWT 토큰을 AsyncStorage에 저장
- Context API로 인증 상태 관리

#### 5.2.2. SignupScreen

- 이메일/비밀번호 입력
- 닉네임 입력
- 학년 선택 (1~3)
- 회원가입 버튼

**기술 구현:**

- Supabase Auth의 `signUp()` 사용
- 회원가입 성공 시 `POST /auth/signup` 호출하여 프로필 생성

---

### 5.3. PageViewerScreen (문제집 페이지 뷰어)

#### 5.3.1. 화면 구성

```
┌─────────────────────────────────────┐
│  ← [문제집 제목]              [페이지] │ ← Header
├─────────────────────────────────────┤
│                                     │
│     [문제집 페이지 이미지]           │
│                                     │
│     ┌─────────────┐  ← 문제 bbox   │
│     │  문제 1     │     오버레이    │
│     └─────────────┘                 │
│                                     │
│     ┌─────────────┐                 │
│     │  문제 2     │                 │
│     └─────────────┘                 │
│                                     │
├─────────────────────────────────────┤
│  [이전 페이지]          [다음 페이지]  │ ← Footer
└─────────────────────────────────────┘
```

#### 5.3.2. 기능

**페이지 로딩:**

1. 컴포넌트 마운트 시 `GET /books/:bookId/pages?page=1` 호출
2. 페이지 이미지를 `react-native-fast-image`로 표시
3. `GET /pages/:pageId/problems` 호출하여 문제 목록 및 bbox 조회

**문제 영역 표시:**

1. 각 문제의 `raw_bbox.problem`을 기반으로 터치 가능한 영역 오버레이
2. 오버레이는 완전 투명 (시각적 표시 없음)
3. 터치 시 즉시 해당 문제 ID, bbox 정보, 페이지 이미지 URL을 `ProblemSolvingScreen`으로 전달

**페이지 네비게이션:**

1. 좌우 스와이프로 페이지 이동
2. 또는 하단 버튼으로 페이지 이동
3. 현재 페이지 번호 표시

**기술 구현:**

- `react-native-fast-image`: 이미지 로딩 최적화
- `react-native-gesture-handler`: 스와이프 제스처
- Bbox 오버레이는 `<View>` + `absolute` 포지셔닝
- Bbox 좌표를 이미지 크기에 맞게 스케일링 필요

#### 5.3.3. 문제 영역 표시 방식 (크롭 없이)

사용자가 문제 영역을 터치하면:

1. 전체 페이지 이미지 URL과 `raw_bbox.problem` 좌표를 `ProblemSolvingScreen`으로 전달
2. 실제 이미지 크롭은 수행하지 않음

**`ProblemSolvingScreen`에서 문제 영역만 표시하는 방법:**

```jsx
<View
  style={{
    width: bbox.x2 - bbox.x1,
    height: bbox.y2 - bbox.y1,
    overflow: 'hidden',
  }}
>
  <Image
    source={{ uri: pageImageUrl }}
    style={{
      position: 'absolute',
      left: -(bbox.x1 * scaleFactor),
      top: -(bbox.y1 * scaleFactor),
      width: originalImageWidth * scaleFactor,
      height: originalImageHeight * scaleFactor,
    }}
  />
</View>
```

**장점:**

- 추가 라이브러리 불필요
- 이미지 크롭 처리 시간 제로
- 메모리 효율적 (원본 이미지 한 번만 로드)

---

### 5.4. ProblemSolvingScreen (문제 풀이)

#### 5.4.1. 화면 구성

```
┌─────────────────────────────────────┐
│  ← [문제 번호]                   [?] │ ← Header
├─────────────────────────────────────┤
│                                     │
│     [크롭된 문제 이미지]             │ ← 상단 고정
│                                     │
│                                     │
├─────────────────────────────────────┤
│                                     │
│     [손글씨 캔버스]                  │ ← 하단 스크롤
│     (펜, 지우개, 실행 취소 등)        │
│                                     │
│                                     │
├─────────────────────────────────────┤
│  답안: [____] (객관식 선택/주관식)    │
│                                     │
│           [제출하기]                 │ ← Footer
└─────────────────────────────────────┘
```

#### 5.4.2. 기능

**문제 이미지 표시:**

- 상단에 문제 영역만 표시 (전체 페이지 이미지의 bbox 영역, 실제 크롭 없이 CSS/View로 처리)
- 줌 인/아웃 가능 (Pinch Gesture)

**손글씨 캔버스:**

- 펜 3색 (검정, 빨강, 파랑)
- 지우개
- 실행 취소 / 다시 실행
- 전체 지우기
- 캔버스 배경: 흰색 (또는 노트 줄무늬)

**답안 입력:**

- 객관식: 선택지 버튼 (문제 정보에서 `options` 가져옴)
- 주관식: 텍스트 입력 필드

**제출:**

1. 손글씨 캔버스를 PNG 이미지로 변환 (Base64)
2. `POST /submissions` 호출
3. 로딩 화면으로 전환

**기술 구현:**

- 손글씨 캔버스: `react-native-sketch-canvas` 또는 `@shopify/react-native-skia`
- 이미지 변환: 캔버스의 `toDataURL()` 또는 `getSnapshot()`
- 시간 기록:
  - `started_at`: 문제 진입 시 타임스탬프
  - `submitted_at`: 제출 버튼 클릭 시 타임스탬프 (풀이 소요 시간 측정용)

---

### 5.5. FeedbackScreen (AI 피드백)

#### 5.5.1. 로딩 화면

제출 직후 로딩 화면 표시:

```
┌─────────────────────────────────────┐
│                                     │
│        [로딩 애니메이션]             │
│                                     │
│   📖 풀이를 꼼꼼히 읽고 있어요...    │ ← STEP 1
│                                     │
│   (2-3초 후)                        │
│   🧮 개념과 계산을 확인 중이에요...  │ ← STEP 2
│                                     │
│   (2-3초 후)                        │
│   💬 맞춤 피드백을 준비하고 있어요...│ ← STEP 3
│                                     │
└─────────────────────────────────────┘
```

**기술 구현:**

- `GET /submissions/:submissionId` 폴링 (1초마다)
- `status: "processing"` → 로딩 계속
- `status: "completed"` → 피드백 화면으로 전환
- `status: "failed"` → 에러 메시지 표시

#### 5.5.2. 피드백 화면

```
┌─────────────────────────────────────┐
│  ← [피드백]                           │ ← Header
├─────────────────────────────────────┤
│  😊 아까운 실수!                     │ ← Summary
│  풀이는 완벽했는데, 마지막에 부호     │
│  실수가 있었어요                      │
├─────────────────────────────────────┤
│  ✅ 잘한 점                           │ ← Strengths
│  ─────────────────────────────────  │
│  • 인수분해 정확                      │
│    x² - 5x + 6을 (x-2)(x-3)으로...  │
├─────────────────────────────────────┤
│  ⚠️ 개선할 점                         │ ← Improvements
│  ─────────────────────────────────  │
│  • 부호 실수                          │
│    - 무엇이 잘못됐나요?               │
│      x - 3 = 0에서 x = -3으로...    │
│    - 왜 틀렸나요?                     │
│      x - 3 = 0을 풀 때...           │
│    - 어떻게 고치나요?                 │
│      x - a = 0 형태에서는...        │
├─────────────────────────────────────┤
│  📚 개념 설명                         │ ← Concept
│  ─────────────────────────────────  │
│  이차방정식의 해 구하기               │
│  인수분해로 (x-a)(x-b) = 0...       │
├─────────────────────────────────────┤
│  ✏️ 올바른 풀이 과정                  │ ← Solution
│  ─────────────────────────────────  │
│  1. x² - 5x + 6 = 0 (주어진 식)     │
│  2. (x - 2)(x - 3) = 0 (인수분해)   │
│  3. x = 2 또는 x = 3 (최종 답)      │
├─────────────────────────────────────┤
│  💡 다음 단계                         │ ← Next Steps
│  ─────────────────────────────────  │
│  부호 실수를 줄이려면...              │
│                                     │
│       [다른 문제 풀기]                │
└─────────────────────────────────────┘
```

**기술 구현:**

- 피드백 JSON을 파싱하여 섹션별로 표시
- 스크롤 가능한 View
- 각 섹션은 아코디언 형태로 접고 펼 수 있음 (선택사항)

---

## 6. 백엔드 구조 (NestJS)

### 6.1. 모듈 구조

```
src/
├── main.ts
├── app.module.ts
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   └── auth.guard.ts
├── books/
│   ├── books.module.ts
│   ├── books.controller.ts
│   └── books.service.ts
├── problems/
│   ├── problems.module.ts
│   ├── problems.controller.ts
│   └── problems.service.ts
├── submissions/
│   ├── submissions.module.ts
│   ├── submissions.controller.ts
│   ├── submissions.service.ts
│   └── dto/
│       ├── create-submission.dto.ts
│       └── submission-response.dto.ts
├── ai/
│   ├── ai.module.ts
│   ├── ai.service.ts
│   ├── gemini.service.ts
│   └── prompts/
│       ├── step1-ocr.prompt.ts
│       ├── step2-analysis.prompt.ts
│       └── step3-feedback.prompt.ts
├── feedbacks/
│   ├── feedbacks.module.ts
│   ├── feedbacks.controller.ts
│   └── feedbacks.service.ts
└── common/
    ├── supabase.service.ts
    └── types/
        └── feedback.types.ts
```

### 6.2. 주요 모듈 설명

#### 6.2.1. AuthModule

- Supabase Auth 연동
- JWT 가드 구현
- 회원가입/로그인/프로필 관리

#### 6.2.2. BooksModule & ProblemsModule

- 문제집/페이지/문제 조회 API
- Supabase DB 쿼리

#### 6.2.3. SubmissionsModule

- 풀이 제출 처리
- 이미지 업로드 (Supabase Storage)
- AI 파이프라인 트리거 (비동기 Job)

#### 6.2.4. AIModule (핵심)

- Gemini API 연동
- 3단계 프롬프트 체인 실행
- 에러 핸들링 및 재시도 로직

**GeminiService 메서드:**

- `ocrAndStructure(problemImage, solutionImage)`: STEP 1
- `analyzeAndClassify(ocrResult, userAnswer, correctAnswer)`: STEP 2
- `generateFeedback(analysisResult)`: STEP 3

**프롬프트 관리:**

- 각 STEP의 프롬프트를 별도 파일로 관리
- 프롬프트 버전 관리 (향후 A/B 테스트용)

#### 6.2.5. FeedbacksModule

- 피드백 조회 API
- 피드백 저장

---

### 6.3. AI 파이프라인 처리 (비동기 Job)

**문제:**

- AI 분석은 6-9초가 걸리므로 동기 처리 시 HTTP 타임아웃 발생 가능

**해결:**

- 제출 즉시 `submission_id` 반환
- 백그라운드에서 AI 파이프라인 실행
- 앱에서 폴링으로 상태 확인

**구현 방법 : 단순 비동기 (간소화)**

```typescript
async createSubmission(userId, data) {
  const submission = await this.supabase
    .from('submissions')
    .insert({ user_id: userId, ...data, status: 'pending' })
    .single();

  // 비동기로 AI 파이프라인 실행 (await 없이)
  this.runAIPipeline(submission.id).catch(error => {
    this.logger.error(`AI pipeline failed for ${submission.id}`, error);
  });

  return submission;
}
```

---

## 7. 기술 스택 및 라이브러리

### 7.1. 앱 (React Native)

| 카테고리          | 라이브러리                                | 용도                              |
| ----------------- | ----------------------------------------- | --------------------------------- |
| **프레임워크**    | React Native                              | 앱 기본 프레임워크                |
| **내비게이션**    | React Navigation 6                        | 화면 전환                         |
| **상태 관리**     | React Context API                         | 인증 상태 관리 (Phase 1은 간소화) |
| **네트워킹**      | Axios                                     | HTTP 요청                         |
| **이미지**        | react-native-fast-image                   | 이미지 로딩 최적화                |
| **손글씨 캔버스** | react-native-sketch-canvas                | 손글씨 입력                       |
| **제스처**        | react-native-gesture-handler              | 스와이프 등 제스처                |
| **스토리지**      | @react-native-async-storage/async-storage | JWT 토큰 저장                     |
| **인증**          | @supabase/supabase-js                     | Supabase Auth                     |

**손글씨 캔버스 라이브러리 선택:**

1. **react-native-sketch-canvas** (추천)
   - 장점: 간단한 API, 안정적
   - 단점: 커스터마이징 제한
2. **@shopify/react-native-skia**
   - 장점: 고성능, 커스터마이징 자유로움
   - 단점: 학습 곡선 높음

**Phase 1 권장:** react-native-sketch-canvas

### 7.2. 백엔드 (NestJS)

| 카테고리          | 라이브러리                         | 용도                                |
| ----------------- | ---------------------------------- | ----------------------------------- |
| **프레임워크**    | NestJS                             | 백엔드 프레임워크                   |
| **DB 클라이언트** | @supabase/supabase-js              | Supabase 연동                       |
| **타입 생성**     | @prisma/client, prisma             | DB 스키마 기반 TypeScript 타입 생성 |
| **인증**          | @nestjs/passport, passport-jwt     | JWT 인증                            |
| **AI API**        | @google/generative-ai              | Gemini API                          |
| **이미지 처리**   | sharp (선택사항)                   | 이미지 리사이징                     |
| **환경 변수**     | @nestjs/config                     | .env 관리                           |
| **검증**          | class-validator, class-transformer | DTO 검증                            |

### 7.3. 데이터베이스 & 스토리지

| 항목        | 기술                  |
| ----------- | --------------------- |
| **DB**      | Supabase (PostgreSQL) |
| **Storage** | Supabase Storage      |
| **인증**    | Supabase Auth         |

### 7.4. AI

| 항목                   | 기술                             |
| ---------------------- | -------------------------------- |
| **LLM 및 이미지 분석** | Google Gemini 1.5 Pro (멀티모달) |

---

## 8. 개발 우선순위 및 마일스톤

### 8.1. Milestone 1: 인증 및 기본 구조 (1주)

**목표:** 로그인/회원가입 + 기본 네비게이션

**앱:**

- [ ] 프로젝트 초기 설정 (React Native CLI 또는 Expo)
- [ ] 인증 화면 (로그인/회원가입)
- [ ] Supabase Auth 연동
- [ ] JWT 토큰 저장 및 인증 Context
- [ ] 기본 네비게이션 구조

**백엔드:**

- [ ] NestJS 프로젝트 초기 설정
- [ ] Supabase 연동 (DB + Auth + Storage)
- [ ] AuthModule 구현
- [ ] JWT Guard 구현

**DB:**

- [ ] 001_initial_schema.sql 실행 (이미 완료)
- [ ] profiles, pages 테이블 추가

### 8.2. Milestone 2: 문제집 페이지 뷰어 (1주)

**목표:** 페이지 이미지 표시 + 문제 영역 선택

**앱:**

- [ ] PageViewerScreen 구현
- [ ] 페이지 이미지 로딩 및 표시
- [ ] 문제 bbox 오버레이 렌더링
- [ ] 문제 영역 터치 → 이미지 크롭
- [ ] 페이지 네비게이션 (좌우 스와이프)

**백엔드:**

- [ ] BooksModule 구현 (GET /books/:bookId)
- [ ] ProblemsModule 구현 (GET /pages/:pageId/problems)

**테스트:**

- [ ] 실제 문제집 데이터로 페이지 표시 확인
- [ ] bbox 크롭 정확도 확인

### 8.3. Milestone 3: 문제 풀이 화면 (1-2주)

**목표:** 손글씨 캔버스 + 답안 입력 + 제출

**앱:**

- [ ] ProblemSolvingScreen 구현
- [ ] 크롭된 문제 이미지 표시
- [ ] 손글씨 캔버스 구현 (펜, 지우개, 실행 취소)
- [ ] 답안 입력 (객관식/주관식)
- [ ] 제출 버튼 → 이미지 Base64 변환
- [ ] 제출 API 호출

**백엔드:**

- [ ] SubmissionsModule 구현 (POST /submissions)
- [ ] 이미지 Base64 → Supabase Storage 업로드
- [ ] submissions 테이블 레코드 생성

**DB:**

- [ ] submissions 테이블 추가
- [ ] user-solutions 스토리지 버킷 추가

**테스트:**

- [ ] 손글씨 입력 → 제출 플로우 확인
- [ ] 이미지 업로드 성공 여부 확인

### 8.4. Milestone 4: AI 파이프라인 (2-3주)

**목표:** 3단계 프롬프트 체인 + PScore 분류

**백엔드:**

- [ ] AIModule 구현
- [ ] GeminiService 구현 (Gemini API 연동)
- [ ] STEP 1: OCR 프롬프트 작성 및 테스트
- [ ] STEP 2: PScore 분류 프롬프트 작성 및 테스트
- [ ] STEP 3: 피드백 생성 프롬프트 작성 및 테스트
- [ ] 3단계 파이프라인 통합
- [ ] 에러 핸들링 및 재시도 로직

**DB:**

- [ ] feedbacks 테이블 추가

**테스트:**

- [ ] 다양한 풀이 케이스로 AI 분석 정확도 테스트
- [ ] PScore 분류 정확도 검증 (최소 10개 샘플)
- [ ] 전체 프로세스 시간 측정 (목표: 10초 이내)

### 8.5. Milestone 5: 피드백 화면 (1주)

**목표:** AI 피드백 표시

**앱:**

- [ ] FeedbackScreen 구현
- [ ] 로딩 화면 (3단계 메시지)
- [ ] 폴링으로 제출 상태 확인
- [ ] 피드백 JSON 파싱 및 UI 렌더링
- [ ] 피드백 섹션별 표시 (요약, 잘한 점, 개선점 등)

**백엔드:**

- [ ] FeedbacksModule 구현 (GET /submissions/:id/feedback)

**테스트:**

- [ ] 전체 플로우 E2E 테스트 (문제 선택 → 풀이 → 제출 → 피드백)
- [ ] 다양한 PScore Case별 피드백 확인

### 8.6. Milestone 6: 통합 테스트 및 최적화 (1주)

**목표:** 버그 수정 + 성능 최적화 + 사용자 테스트

**앱:**

- [ ] 이미지 로딩 최적화
- [ ] 캐싱 전략 구현
- [ ] 에러 핸들링 개선
- [ ] UX 개선 (로딩 상태, 에러 메시지 등)

**백엔드:**

- [ ] API 응답 시간 최적화
- [ ] 로깅 및 모니터링 설정

**테스트:**

- [ ] 최소 10개 문제로 E2E 테스트
- [ ] 베타 테스터 3-5명 초대하여 피드백 수집
- [ ] AI 정확도 재검증 (목표: 85% 이상)

---

## 9. 주요 기술 과제 및 해결 방안

### 9.1. 문제집 페이지 이미지 로딩

**Phase 1 접근:**

- 페이지 이동 시마다 해당 페이지 이미지 로드
- 이미지는 문제 업로더가 이미 압축하여 업로드하므로 추가 처리 불필요
- CDN 활용: Supabase Storage의 CDN 기능으로 빠른 로딩

**향후 최적화 (Phase 2+):**

- Lazy Loading, 프리로딩 등의 최적화는 필요 시 추가

### 9.2. 문제 영역 bbox 렌더링

**과제:**

- bbox 좌표가 절대 좌표인데, 화면 크기에 따라 스케일링 필요

**해결:**

- 이미지 원본 크기와 화면에 표시되는 크기의 비율 계산
- bbox 좌표를 화면 좌표로 변환
- `onLayout` 이벤트로 이미지 크기 동적 계산

**예시 코드:**

```typescript
const scaleFactor = displayedImageWidth / originalImageWidth;
const scaledBbox = {
  x1: bbox.x1 * scaleFactor,
  y1: bbox.y1 * scaleFactor,
  x2: bbox.x2 * scaleFactor,
  y2: bbox.y2 * scaleFactor,
};
```

### 9.3. Gemini API 3단계 프롬프트 체인

**Phase 1 접근:**

- 3단계 순차 실행 (STEP 1 → STEP 2 → STEP 3)
- 타임아웃 설정: 각 STEP별 타임아웃 (예: 30초)
- API 호출 실패 시 사용자에게 오류 안내

**향후 최적화 (Phase 2+):**

- 재시도 로직, 캐싱 등은 필요 시 추가

---

## 10. 에러 처리 및 예외 케이스

### 10.1. AI API 호출 실패

**케이스:**

- Gemini API 타임아웃, 할당량 초과, 네트워크 오류

**처리:**

- submission 상태를 `failed`로 업데이트
- 사용자에게 "일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요" 안내

### 10.2. 빈 캔버스 제출

**케이스:**

- 사용자가 아무것도 그리지 않고 제출

**처리:**

- 클라이언트에서 캔버스가 비어있는지 확인 (선택사항)
- 또는 AI가 STEP 1에서 `has_work: false` 감지 → Case 6 (백지)로 분류

### 10.3. 정답 정보 없음

**케이스:**

- DB에 정답이 누락된 문제

**처리:**

- STEP 2에서 정답 없이 분석 (풀이 과정만 평가)
- 피드백에서 "정답 확인이 어려워요" 안내
- PScore는 풀이 과정 품질만으로 판단

---

## 부록: 참고 자료

### A. Gemini API 문서

- https://ai.google.dev/docs

### B. Supabase 문서

- https://supabase.com/docs

### C. React Native 손글씨 캔버스 라이브러리

- react-native-sketch-canvas: https://github.com/terrylinla/react-native-sketch-canvas
- @shopify/react-native-skia: https://shopify.github.io/react-native-skia/
