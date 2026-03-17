# GEMINI.md - 프로젝트 가이드라인

이 파일은 `seungwoonlee.github.io` 프로젝트를 다루는 모든 AI 에이전트가 준수해야 할 최상위 지침입니다.

## 1. 언어 및 소통 원칙
- 모든 답변, 설명, 코드 내 주석은 **한국어**로 작성합니다.
- 전문 용어는 가급적 원문을 병기하되, 친절하고 직관적인 톤을 유지합니다.

## 2. 기술 스택 및 스타일
- **Framework**: Astro (최신 정적 사이트 생성기)
- **Styling**: Tailwind CSS (Utility-first CSS)
- **Vibe Coding**: 불필요한 추상화보다는 직관적이고 생산성이 높은 코딩 스타일을 지향합니다.
- **Visuals**: 배너나 그래픽 요소는 가급적 SVG를 활용하여 성능과 선명도를 동시에 확보합니다.

## 3. 작업 워크플로우
- **Verification**: 모든 코드 수정이나 설정 변경 후에는 반드시 `npm run build`를 실행하여 빌드 오류가 없는지 검증합니다.
- **Git Strategy**:
  - `master` 브랜치를 기본 배포 브랜치로 사용합니다.
  - 커밋 메시지는 `feat:`, `fix:`, `design:`, `docs:`, `chore:` 등 Conventional Commits 형식을 따릅니다.

## 4. 에셋 및 콘텐츠 관리
- **Images**: 블로그 포스트(`src/content/blog/`) 내에서 이미지를 참조할 때는 반드시 `src/assets/` 폴더 내의 파일을 상대 경로(예: `../../assets/...`)로 연결합니다.
- **Deployment**: GitHub Actions를 통한 자동 배포 시스템을 존중하며, 배포 실패 시 `nojekyll` 설정 등을 우선 점검합니다.

## 5. 특이 사항
- 데이터 시각화가 필요한 경우 `seaborn` 스타일을 사용하지 않으며, `matplotlib` 사용 시 `koreanize-matplotlib`를 사용하여 한글 폰트 설정을 보장합니다.
- 가상환경이 필요할 경우 `uv`를 사용하며 `.venv` 폴더명을 유지합니다.
