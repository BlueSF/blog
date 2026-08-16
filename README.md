# Blog

백엔드 개발 기록. [AstroPaper](https://github.com/satnaing/astro-paper) 테마 기반.

## 개발

```bash
pnpm install
pnpm dev      # http://localhost:4321
pnpm build    # astro check + build + pagefind 색인
pnpm preview
```

## 글 쓰기

`src/content/posts/` 에 `.md` 또는 `.mdx` 파일을 추가한다. **파일명이 그대로 URL 슬러그**가 된다
(`kotlin-entity-cannot-be-removed.md` → `/posts/kotlin-entity-cannot-be-removed/`).
`_` 로 시작하는 파일·디렉터리는 빌드에서 제외된다.

프론트매터 스키마는 `src/content.config.ts` 참고.

| 필드                                                                 | 필수 | 비고                                             |
| -------------------------------------------------------------------- | ---- | ------------------------------------------------ |
| `title`                                                              | O    |                                                  |
| `pubDatetime`                                                        | O    | `2026-08-14T22:00:00+09:00`                      |
| `description`                                                        | O    | SEO 메타 · 목록 요약                             |
| `author`                                                             |      | 기본값: `astro-paper.config.ts` 의 `site.author` |
| `tags`                                                               |      | 기본값 `["others"]`                              |
| `modDatetime` `featured` `draft` `ogImage` `canonicalURL` `timezone` |      |                                                  |

## 설정

사이트 제목·설명·저자·URL·소셜 링크는 전부 `astro-paper.config.ts` 에 있다.

## 배포

Cloudflare Workers 정적 자산 (`wrangler.jsonc`, 배포 디렉터리 `./dist`).

## 라이선스

MIT. 테마 원저작권은 [Sat Naing](https://github.com/satnaing) 에게 있다 (`LICENSE` 참고).
