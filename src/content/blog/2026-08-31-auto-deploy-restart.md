---
title: '블로그 자동 배포 재가동'
description: 'GitHub 토큰 재발급으로 자동 배포 파이프라인을 다시 켰습니다'
pubDate: 'Aug 31 2026'
heroImage: '../../assets/astro-banner.svg'
---

오늘 GitHub fine-grained 토큰(`Hermes-Agent-Blog-Key`)을 재발급하고, 예전에 꺼져 있던 **자동 블로그 배포**를 다시 켰습니다.

## 자동 배포 파이프라인

이 블로그는 다음 구조로 자동 배포됩니다:

1. **글 작성**: `src/content/blog/`에 마크다운으로 글을 추가
2. **push**: `git push`로 GitHub 저장소에 반영
3. **자동 빌드**: GitHub Actions `deploy.yml`이 Astro 빌드 실행
4. **자동 배포**: GitHub Pages에 배포

이 글은 그 파이프라인이 정상 동작하는지 확인하는 **테스트 글**입니다.

## Hermes 자동화

이제 Hermes 에이전트가 이 블로그에 글을 작성하고 push하면, 별도 수동 작업 없이 자동으로 배포됩니다.
