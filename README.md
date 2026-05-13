# xodnjs9850.github.io

비전공자 대상 **OpenClaw로 만드는 나만의 메일 비서** 강의 자료 (4일 × 4시간).

🌐 사이트: https://xodnjs9850.github.io

## 구조

- `_config.yml` — Jekyll 설정 (just-the-docs 원격 테마)
- `index.md` — 강의 홈
- `00-pre-assignment.md` — 기본 설정
- `01-day1-setup.md` — Day 1 환경 셋업
- `02-day2-mail.md` — Day 2 네이버 메일 연동
- `03-day3-cron.md` — Day 3 스팸 분류 + 자동화 (cron)
- `04-day4-discord.md` — Day 4 Discord 연결 + 대화형 비서

## 로컬 미리보기 (선택)

Ruby 3+ + Bundler 설치된 환경:

```bash
bundle install
bundle exec jekyll serve
```

브라우저에서 `http://localhost:4000` 으로 확인.

## 발행

`main` 브랜치에 push하면 GitHub Pages가 자동 빌드합니다. 저장소 Settings → Pages에서 "Deploy from a branch" → `main` / root 선택.
