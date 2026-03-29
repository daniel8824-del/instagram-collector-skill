# Instagram Collector Skill

인스타그램 해시태그/계정 게시물 수집 Claude Code 스킬입니다. **Apify API** 기반으로 동작합니다.

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue)](https://www.python.org/)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://claude.com/claude-code)
[![Apify](https://img.shields.io/badge/Apify-API-00C896)](https://apify.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## Features

- 해시태그만 말하면 **인스타그램 게시물 자동 수집** (멀티 해시태그 지원)
- **계정 게시물 수집** (@username으로 특정 계정 크롤링)
- 게시물 상세 정보: 캡션, 해시태그, 좋아요, 댓글수, 게시일, 게시물 유형
- 최종 결과물: **CSV + TXT**
- Apify 무료 크레딧 $5/월로 충분히 사용 가능

## 사용 방법

### 1. Claude Code 스킬

```bash
git clone https://github.com/daniel8824-del/instagram-collector-skill.git ~/.claude/skills/instagram-collector
```

Claude Code에서:
```
"스킨케어 인스타 검색해줘"
"스킨케어,뷰티 해시태그 5건씩"
"@brand_name 계정 게시물 수집"
```

### 2. Google Antigravity

```bash
git clone https://github.com/daniel8824-del/instagram-collector-skill.git
cd instagram-collector-skill
```

`.agent/` 폴더가 자동 인식됩니다.

## 사전 설정

### Apify API 토큰 (필수)

1. https://console.apify.com/account/integrations 에서 토큰 발급
2. `~/.env` 또는 프로젝트 `.env`에 설정:

```bash
APIFY_API_TOKEN=apify_api_여기에_토큰_입력
```

## 기술 스택

- `httpx` -- Apify API 호출
- `python-dotenv` -- 환경변수 로딩
- `rich` -- 터미널 출력 포맷팅
- Apify Actors: Instagram Hashtag Scraper, Instagram Profile Scraper

## License

MIT
