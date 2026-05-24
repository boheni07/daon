# Plan: 프로젝트 폴더 구조 설정

> **Feature:** project-folder-setup
> **작성일:** 2026-05-22
> **Phase:** Plan ✅
> **Status:** Completed

---

## Executive Summary

| 항목 | 내용 |
|---|---|
| **Feature** | Daon 프로젝트 bkit PDCA 폴더 구조 초기화 및 PRD 파일 정리 |
| **작성일** | 2026-05-22 |
| **소요 시간** | < 5분 |

### Value Delivered (4-Perspective)

| 관점 | 내용 |
|---|---|
| **Problem** | PRD 파일 2개가 루트에 혼재하고 폴더 구조 없음 — 향후 PDCA 개발 관리 불가 |
| **Solution** | bkit PDCA 표준 구조(`docs/00-pm ~ 04-report`, `src/`) 생성 및 파일 이동 |
| **Function & UX Effect** | 개발자가 어느 단계 문서가 어디 있는지 즉시 파악 가능, bkit 툴 자동 인식 |
| **Core Value** | 다온 프로젝트의 첫 번째 PDCA 사이클을 시작할 수 있는 기반 마련 |

---

## Context Anchor

| Anchor | 값 |
|---|---|
| **WHY** | bkit PDCA 워크플로우를 사용하기 위한 폴더 구조 초기화 — 모든 후속 Plan/Design/Do 단계의 전제 조건 |
| **WHO** | NUBiz AX 기획팀 / 개발팀 |
| **RISK** | 낮음 — 파일 이동 및 폴더 생성만 포함 |
| **SUCCESS** | bkit PDCA 경로(`docs/00-pm/*.prd.md` 등)가 올바르게 인식됨 |
| **SCOPE** | (IN) 폴더 생성, PRD 이동, 구 초안 삭제 / (OUT) 소스코드 개발, 시스템 설계 |

---

## 1. 요구사항

| # | 요구사항 | 우선순위 |
|---|---|---|
| R-01 | bkit PDCA 표준 폴더 구조 생성 | 필수 |
| R-02 | `Daon_PRD.md` → `docs/00-pm/daon.prd.md` 이동 | 필수 |
| R-03 | `PRD_자서전반려로봇.md` (v0.1 초안) 삭제 | 필수 |
| R-04 | 향후 소스코드를 위한 `src/` 폴더 생성 | 선택 |

---

## 2. 구현 결과

### 생성된 폴더 구조

```
Daon/
├── docs/
│   ├── 00-pm/              ← PRD 문서
│   │   └── daon.prd.md     ← Daon PRD v1.0 (이동 완료)
│   ├── 01-plan/
│   │   └── features/       ← Plan 문서 (이 파일 위치)
│   ├── 02-design/
│   │   └── features/       ← Design 문서
│   ├── 03-analysis/        ← Gap 분석 문서
│   └── 04-report/
│       └── features/       ← 완료 보고서
├── src/                    ← 향후 소스코드
└── .bkit/                  ← bkit 설정 (기존)
```

### 처리된 파일

| 파일 | 처리 | 결과 |
|---|---|---|
| `Daon_PRD.md` | `docs/00-pm/daon.prd.md`로 이동 | ✅ 완료 |
| `PRD_자서전반려로봇.md` | 삭제 (v0.1 구버전) | ✅ 완료 |

---

## 3. 다음 단계

- `/pdca plan daon` — 다온 서비스 개발 Plan 문서 작성 (PRD 자동 참조)
- `/pdca design daon` — 시스템 아키텍처 설계
