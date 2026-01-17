# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

Jekyll 기반 개인 기술 블로그 (jekyll-theme-chirpy 테마)

## 명령어

```bash
bundle exec jekyll serve  # 로컬 서버 실행
bundle exec jekyll build  # 빌드
```

## 커스텀 명령어 (테크니컬 라이팅)

학습/트러블슈팅 후 블로그 포스트를 작성하는 워크플로우:

| 명령어 | 설명 |
|--------|------|
| `/learn [주제]` | 학습 세션 시작. 주제에 대해 리서치하고 핵심 개념 설명 |
| `/troubleshoot [문제]` | 트러블슈팅 세션 시작. 원인 분석 및 해결 경험 정리 |
| `/experience [경험]` | 경험 세션 시작. 프로젝트 경험을 질문을 통해 정리 |
| `/write-post` | 현재 대화 내용을 블로그 포스트로 변환 (3단계 프로세스) |

### 사용 플로우

1. `/learn`, `/troubleshoot`, `/experience` 중 하나로 세션 시작
2. 대화를 통해 학습 또는 문제 해결 진행
3. `/write-post` 입력 시 Claude가 3단계로 문서화 진행:
   - 1단계: 문서 유형 제안 (학습/문제해결/경험회고)
   - 2단계: 정보 구조 초안 작성
   - 3단계: 최종 문장 다듬기
4. 최종 검토 후 커밋

## 포스트 작성

### 파일 구조
- **위치**: `_posts/`
- **파일명**: `YYYY-MM-DD-제목.md`
- **템플릿**: `template/post.md`

### Front Matter
```yaml
---
title: "[카테고리] 제목"
date: YYYY-MM-DD HH:MM +0900
categories:
  - 상위 카테고리
  - 하위 카테고리
tags:
  - 태그1
  - 태그2
---
```

### 작성 컨벤션
- **제목**: `[약어] 제목` 형식 (예: `[Swift]디자인시스템 구조 개선하기`, `[SwiftUI]네비게이션 라우터`)
- **섹션**: ## 이모지 + 제목 (예: `## 🤔 제목`)
- **레퍼런스**: 마지막에 `## 참고` 섹션 포함
- **Prompt 블록**: `{: .prompt-info }`, `{: .prompt-tip }` 사용 가능

## 이미지 관리

**경로**: `assets/img/post/연도/`

## 카테고리 구조

| 상위 카테고리 | 하위 카테고리 |
|-------------|-------------|
| 🍎 iOS | Swift, UIKit, SwiftUI, Xcode, AVFoundation, Architecture |
| 📚 CS | Computer Science, Problem Solving |
| ⛅️ Log | 회고 |
| ⚙️ Automation | CI/CD, Workflow, Plugin |

## Git 워크플로우

**커밋 메시지**: `docs: 설명` 형식

예시:
- `docs: "제목" 문서 추가`
- `docs: "제목" 문서 수정`
- `docs: 문서 카테고리 리네이밍`
