---
name: instagram-collector
description: Instagram hashtag and account post collection via Apify API. Collects captions, hashtags, likes, comments, and saves CSV + TXT. Triggers on "인스타그램 수집", "인스타 검색", "인스타 해시태그", "instagram", "인스타 모아줘", "인스타그램 수집해줘".
---

# 인스타그램 수집기 (Instagram Collector)

해시태그/계정 기반 인스타그램 게시물 수집 → CSV + TXT 저장 → (선택) 분석까지 자동 수행합니다.
**Apify API** 기반으로 동작합니다 (무료 $5/월 크레딧).

## 역할

당신은 인스타그램 트렌드 분석가입니다.
사용자의 관심 주제를 파악하고, 인스타그램에서 관련 게시물을 수집한 뒤, 핵심 인사이트를 도출합니다.
사용자와 한국어로 대화합니다.

---

## 전체 흐름

```
Phase 0: SETUP (최초 1회)
  ├── pip install 의존성
  └── APIFY_API_TOKEN 설정

Phase 1: INTAKE (정보 수집)
  └── 사용자 요청에서 해시태그/계정/건수 자연스럽게 파악

Phase 2: COLLECT (게시물 수집)
  ├── PROJECT_DIR 생성
  ├── instagram_collect.py 작성 (아래 코드)
  └── python3 instagram_collect.py 실행 → CSV + TXT 저장

Phase 3: ANALYZE (선택적 분석)
  └── CSV/TXT를 Read 도구로 읽고 Claude가 직접 분석
```

> **중요:** 매 작업 시작 시 고유 폴더를 생성합니다.
> ```bash
> PROJECT_DIR="$HOME/Downloads/instagram_$(date +%Y%m%d_%H%M)"
> mkdir -p "$PROJECT_DIR"
> ```

---

# Phase 0: SETUP (최초 1회)

## 의존성 설치

```bash
pip install -q httpx rich python-dotenv 2>/dev/null
```

## API 토큰 설정

Apify API 토큰이 필요합니다 (무료 $5/월 크레딧).

1. https://console.apify.com/account/integrations 에서 토큰 발급
2. 환경변수 설정:

```bash
# ~/.env 또는 프로젝트 .env
APIFY_API_TOKEN=apify_api_여기에_토큰_입력
```

또는 실행 시 직접 지정:

```bash
APIFY_API_TOKEN=apify_api_xxx python3 instagram_collect.py search "스킨케어" 10
```

---

# Phase 1: INTAKE

사용자 요청을 자연스럽게 파악하여 파라미터로 변환합니다.

| 사용자 표현 | 명령 |
|------------|------|
| "스킨케어 인스타 검색해줘" | `python3 instagram_collect.py search "스킨케어" 10` |
| "스킨케어,뷰티 해시태그 5건씩" | `python3 instagram_collect.py search "스킨케어,뷰티" 5` |
| "@brand_name 계정 게시물" | `python3 instagram_collect.py search "@brand_name" 10` |
| "카페추천 인스타 20건" | `python3 instagram_collect.py search "카페추천" 20` |

기본값: 해시태그당 10건

---

# Phase 2: COLLECT

## 실행

`$PROJECT_DIR/instagram_collect.py`로 아래 코드를 저장하고 실행합니다.

```bash
cd "$PROJECT_DIR" && python3 instagram_collect.py search "키워드" 10
```

## instagram_collect.py

```python
#!/usr/bin/env python3
"""인스타그램 수집기 - 해시태그/계정 게시물 수집 (Apify API 기반)"""

import argparse, csv, logging, os, re, sys, time
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path

import httpx
from dotenv import load_dotenv
from rich.console import Console
from rich.panel import Panel

console = Console()
logger = logging.getLogger(__name__)

# ── Apify Actor IDs ──

HASHTAG_ACTOR = "reGe1ST3OBgYZSsZJ"   # Instagram Hashtag Scraper
PROFILE_ACTOR = "dSCLg0C3YEZ83HzYX"   # Instagram Profile Scraper

# ── 데이터 모델 ──

@dataclass
class InstaPost:
    caption: str = ""
    hashtags: list[str] = field(default_factory=list)
    likes: int = 0
    comments: int = 0
    posted_at: str = ""
    url: str = ""
    shortcode: str = ""
    owner: str = ""
    image_url: str = ""
    post_type: str = ""  # Image, Video, Sidecar

# ── 유틸 ──

def _extract_hashtags(caption: str) -> list[str]:
    return re.findall(r"#([^\s#]+)", caption)

def _clean_caption(caption: str) -> str:
    return re.sub(r"#[^\s#]+", "", caption).strip()

def _fmt_date(ts: str) -> str:
    if not ts:
        return ""
    try:
        return ts[:10]
    except Exception:
        return ts

def _get_token() -> str:
    token = os.getenv("APIFY_API_TOKEN", "")
    if not token:
        console.print("[red][오류] APIFY_API_TOKEN이 설정되지 않았습니다.[/red]")
        console.print("  발급: https://console.apify.com/account/integrations")
        console.print("  설정: export APIFY_API_TOKEN=apify_api_xxx")
        sys.exit(1)
    return token

def _downloads() -> Path:
    win = Path("/mnt/c/Users") / os.getenv("USER", "daniel") / "Downloads"
    linux = Path.home() / "Downloads"
    if win.is_dir():
        return win
    if linux.is_dir():
        return linux
    return Path.home()

# ── 검색 ──

def search_by_hashtag(hashtags: list[str], max_results: int = 10, api_token: str = "") -> list[InstaPost]:
    if not api_token:
        api_token = _get_token()

    url = f"https://api.apify.com/v2/acts/{HASHTAG_ACTOR}/run-sync-get-dataset-items"
    headers = {"Authorization": f"Bearer {api_token}"}

    all_posts = []
    seen_urls = set()

    for tag in hashtags:
        tag = tag.lstrip("#").strip()
        if not tag:
            continue

        console.print(f"  [dim]해시태그 검색: #{tag} ({max_results}건)[/dim]")

        body = {
            "hashtags": [tag],
            "resultsLimit": max_results,
            "resultsType": "posts",
        }

        try:
            resp = httpx.post(url, json=body, headers=headers, timeout=120.0)
            resp.raise_for_status()
            items = resp.json()

            for item in items:
                post_url = item.get("url", "")
                if post_url in seen_urls:
                    continue
                seen_urls.add(post_url)

                caption = item.get("caption", "") or ""
                all_posts.append(InstaPost(
                    caption=caption,
                    hashtags=_extract_hashtags(caption),
                    likes=max(item.get("likesCount", 0) or 0, 0),
                    comments=max(item.get("commentsCount", 0) or 0, 0),
                    posted_at=item.get("timestamp", "") or "",
                    url=post_url,
                    shortcode=item.get("shortCode", ""),
                    owner=item.get("ownerUsername", ""),
                    image_url=item.get("displayUrl", ""),
                    post_type=item.get("type", ""),
                ))

            console.print(f"  [green]#{tag}: {len(items)}건[/green]")

        except httpx.HTTPStatusError as e:
            if e.response.status_code == 402:
                console.print("[red]Apify 크레딧 부족. https://console.apify.com 에서 확인하세요.[/red]")
            else:
                console.print(f"[red]Apify API 오류: {e.response.status_code}[/red]")
        except Exception as e:
            console.print(f"[red]검색 오류 (#{tag}): {str(e)[:200]}[/red]")

        if len(hashtags) > 1:
            time.sleep(2)

    return all_posts


def search_by_username(username: str, max_results: int = 10, api_token: str = "") -> list[InstaPost]:
    if not api_token:
        api_token = _get_token()

    username = username.lstrip("@").strip()

    url = f"https://api.apify.com/v2/acts/{PROFILE_ACTOR}/run-sync-get-dataset-items"
    headers = {"Authorization": f"Bearer {api_token}"}

    body = {
        "usernames": [username],
        "resultsLimit": max_results,
        "resultsType": "posts",
    }

    try:
        resp = httpx.post(url, json=body, headers=headers, timeout=120.0)
        resp.raise_for_status()
        items = resp.json()

        posts = []
        for item in items:
            caption = item.get("caption", "") or ""
            posts.append(InstaPost(
                caption=caption,
                hashtags=_extract_hashtags(caption),
                likes=item.get("likesCount", 0) or 0,
                comments=item.get("commentsCount", 0) or 0,
                posted_at=item.get("timestamp", "") or "",
                url=item.get("url", ""),
                shortcode=item.get("shortCode", ""),
                owner=item.get("ownerUsername", username),
                image_url=item.get("displayUrl", ""),
                post_type=item.get("type", ""),
            ))

        return posts[:max_results]

    except httpx.HTTPStatusError as e:
        if e.response.status_code == 402:
            console.print("[red]Apify 크레딧 부족.[/red]")
        else:
            console.print(f"[red]Apify API 오류: {e.response.status_code}[/red]")
        return []
    except Exception as e:
        console.print(f"[red]계정 검색 오류 (@{username}): {str(e)[:200]}[/red]")
        return []

# ── 출력 ──

def print_results(posts: list[InstaPost], query: str = ""):
    if not posts:
        console.print("[yellow]검색 결과가 없습니다.[/yellow]")
        return

    console.print(Panel(
        f"[bold]검색어:[/bold] {query}\n"
        f"[bold]결과:[/bold] {len(posts)}건  "
        f"[dim]{datetime.now().strftime('%Y-%m-%d %H:%M')}[/dim]",
        title="[bold magenta]인스타그램 수집 결과[/bold magenta]",
        border_style="magenta",
    ))

    for i, p in enumerate(posts[:15], 1):
        caption_preview = p.caption[:100].replace("\n", " ") if p.caption else "(캡션 없음)"
        tags = " ".join(f"#{t}" for t in p.hashtags[:5]) if p.hashtags else ""
        console.print(f"\n  [bold]{i}. @{p.owner}[/bold]  ♥ {p.likes:,}  💬 {p.comments:,}  {_fmt_date(p.posted_at)}")
        console.print(f"     {caption_preview}")
        if tags:
            console.print(f"     [dim]{tags}[/dim]")
        console.print(f"     [dim]{p.url}[/dim]")


def to_csv(posts: list[InstaPost], filepath: str, query: str = ""):
    path = Path(filepath)
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "w", encoding="utf-8-sig", newline="") as f:
        w = csv.writer(f)
        w.writerow(["검색어", "작성자", "캡션", "해시태그", "좋아요", "댓글수", "게시일", "게시물유형", "URL"])
        for p in posts:
            tags = ", ".join(f"#{t}" for t in p.hashtags)
            clean_caption = p.caption.replace("\n", " ") if p.caption else ""
            w.writerow([query, p.owner, clean_caption, tags, p.likes, p.comments,
                        _fmt_date(p.posted_at), p.post_type, p.url])
    console.print(f"[green]CSV: {filepath}[/green]")


def to_txt(posts: list[InstaPost], filepath: str, query: str = ""):
    path = Path(filepath)
    path.parent.mkdir(parents=True, exist_ok=True)
    lines = [
        f"인스타그램 수집 결과: {query}",
        f"수집일시: {datetime.now().strftime('%Y-%m-%d %H:%M')}",
        f"총 {len(posts)}건",
        "=" * 70,
    ]
    for i, p in enumerate(posts, 1):
        tags = " ".join(f"#{t}" for t in p.hashtags)
        lines.append(f"\n[{i}] @{p.owner}")
        lines.append(f"    좋아요: {p.likes:,}  댓글: {p.comments:,}  날짜: {_fmt_date(p.posted_at)}")
        lines.append(f"    유형: {p.post_type}")
        lines.append(f"    URL: {p.url}")
        if tags:
            lines.append(f"    태그: {tags}")
        lines.append("-" * 70)
        lines.append(p.caption or "(캡션 없음)")
        lines.append("=" * 70)

    path.write_text("\n".join(lines) + "\n", encoding="utf-8")
    console.print(f"[green]TXT: {filepath}[/green]")

# ── 메인 ──

def main():
    load_dotenv()
    home_env = Path.home() / ".env"
    if home_env.exists():
        load_dotenv(home_env)

    ap = argparse.ArgumentParser(
        prog="instagram_collect",
        description="인스타그램 수집기 - 캡션/해시태그/좋아요 수집 (Apify API)",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
사용 예시:
  python3 instagram_collect.py search "스킨케어" 10            해시태그 검색
  python3 instagram_collect.py search "스킨케어,뷰티" 5        멀티 해시태그
  python3 instagram_collect.py search "@brand_name" 10         계정 게시물 수집
        """,
    )

    sub = ap.add_subparsers(dest="command")

    p_s = sub.add_parser("search", help="해시태그/계정 검색 → CSV + TXT")
    p_s.add_argument("query", nargs="+", help="해시태그 또는 @계정 (콤마로 멀티, 마지막 숫자는 건수)")
    p_s.add_argument("-n", "--count", type=int, default=10, help="해시태그당 수집 건수 (기본: 10)")

    args = ap.parse_args()
    if not args.command:
        ap.print_help()
        return

    dl = _downloads()
    ts = datetime.now().strftime("%Y%m%d_%H%M")

    if args.command == "search":
        raw = args.query
        count = args.count
        if len(raw) > 1 and raw[-1].isdigit():
            count = int(raw[-1])
            raw = raw[:-1]
        query_text = " ".join(raw)
        queries = [q.strip() for q in query_text.split(",") if q.strip()]

        is_username = queries[0].startswith("@") if queries else False

        if is_username:
            console.print(f"\n[bold magenta]인스타그램 계정 수집[/bold magenta] @{queries[0].lstrip('@')}")
            console.print(f"  최대 {count}건", style="dim")
            posts = search_by_username(queries[0], count)
        else:
            if len(queries) > 1:
                console.print(f"\n[bold magenta]멀티 해시태그 검색[/bold magenta] {len(queries)}개: {', '.join(queries)}")
            else:
                console.print(f"\n[bold magenta]인스타그램 검색[/bold magenta] #{queries[0]}")
            console.print(f"  해시태그당 {count}건", style="dim")
            posts = search_by_hashtag(queries, count)

        if not posts:
            console.print("[yellow]검색 결과가 없습니다.[/yellow]")
            return

        console.print(f"\n  [green]총 {len(posts)}건[/green]")
        print_results(posts, ", ".join(queries))

        # 저장
        slug = queries[0].lstrip("@#").replace(" ", "_")[:20]
        query_label = ", ".join(queries)
        csv_path = str(dl / f"instagram_{slug}_{ts}.csv")
        txt_path = str(dl / f"instagram_{slug}_{ts}.txt")

        to_csv(posts, csv_path, query_label)
        to_txt(posts, txt_path, query_label)
        console.print(f"  총 {len(posts)}건 수집 완료.\n")


if __name__ == "__main__":
    main()
```

---

# Phase 3: ANALYZE (선택적)

사용자가 "분석해줘", "요약해줘" 등을 요청하면 실행합니다.

1. Read 도구로 CSV 또는 TXT 파일 읽기
2. Claude가 직접 분석
3. 마크다운으로 보고

| 유형 | 사용자 표현 | Claude가 하는 일 |
|------|-----------|----------------|
| 요약 | "요약해줘" | 수집된 게시물 핵심 내용 요약 |
| 트렌드 | "트렌드 알려줘" | 좋아요/댓글 분포, 인기 해시태그 분석 |
| 계정 분석 | "계정 분석해줘" | 게시물 패턴, 참여율, 콘텐츠 유형 분석 |
| 인사이트 | "인사이트 뽑아줘" | 핵심 발견 5개 + 시사점 |
| 종합 | "분석해줘" | 위 전부 간략 수행 |

---

# 에러 대응

| 에러 | 해결 |
|------|------|
| APIFY_API_TOKEN 없음 | https://console.apify.com/account/integrations 에서 발급 후 `~/.env`에 설정 |
| 모듈 미설치 | `pip install httpx rich python-dotenv` |
| 402 크레딧 부족 | Apify 콘솔에서 크레딧 확인 및 충전 |
| 검색 결과 없음 | 해시태그 변경 또는 영문 해시태그 시도 |
| 타임아웃 | `max_results` 줄이거나 재시도 |
