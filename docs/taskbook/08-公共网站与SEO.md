# 第 08 章：公共网站、静态导出与 SEO

本章创建 Next.js 公共站点。后台尚未创建。所有公共内容在构建时从 API 获取并导出为 HTML；生产不运行 Node 服务器。

## 08-01 创建分支和目录

```powershell
PS> Set-Location 'D:\CODEkingdom\rodolfoiolo'
PS> git switch main
PS> git pull --ff-only
PS> git status --short --branch
PS> git switch -c feat/public-web
PS> New-Item -ItemType Directory -Force -Path 'web\app\(public)\articles\[slug]','web\app\(public)\projects\[slug]','web\app\(public)\archive','web\app\(public)\about','web\components\cards','web\components\layout','web\components\markdown','web\components\ui','web\lib','web\public','web\scripts' | Out-Null
```

PowerShell 对 `[slug]` 有通配符语义；`New-Item -Path` 创建失败时改用 `-LiteralPath` 逐个创建。确认目录名真的包含方括号。

## 08-02 创建 package.json

创建 `web/package.json`：

```json
{
  "name": "personal-site-web",
  "version": "0.1.0",
  "private": true,
  "packageManager": "pnpm@11.19.0",
  "engines": {
    "node": "24.14.0",
    "pnpm": "11.19.0"
  },
  "scripts": {
    "dev": "next dev",
    "build": "node scripts/sync-public-metadata.mjs && next build",
    "lint": "eslint . --max-warnings=0",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  },
  "dependencies": {
    "@tailwindcss/postcss": "4.3.3",
    "lenis": "1.3.11",
    "lucide-react": "0.468.0",
    "motion": "12.23.12",
    "next": "16.3.2",
    "react": "19.1.1",
    "react-dom": "19.1.1",
    "react-markdown": "10.1.0",
    "rehype-sanitize": "6.0.0",
    "remark-gfm": "4.0.1",
    "tailwindcss": "4.3.3"
  },
  "devDependencies": {
    "@eslint/eslintrc": "3.3.1",
    "@testing-library/jest-dom": "6.8.0",
    "@testing-library/react": "16.3.0",
    "@types/node": "24.3.0",
    "@types/react": "19.1.10",
    "@types/react-dom": "19.1.7",
    "eslint": "9.34.0",
    "eslint-config-next": "16.3.2",
    "jsdom": "26.1.0",
    "typescript": "5.9.2",
    "vitest": "3.2.4"
  }
}
```

如果执行时 Next.js 16.3.2 要求与这里不同的 React 精确补丁版本，以 `pnpm` 的 peer dependency 错误为证据，在独立 `chore(deps)` 提交中同步修改两项，不能关闭 strict peers。

安装并更新 workspace 锁文件：

```powershell
PS> pnpm install
PS> pnpm --dir web exec next --version
PS> pnpm --dir web exec tsc --version
```

## 08-03 创建框架配置

创建 `web/next.config.mjs`：

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export',
  trailingSlash: true,
  images: { unoptimized: true },
  poweredByHeader: false,
  reactStrictMode: true,
};

export default nextConfig;
```

创建 `web/tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules", "out"]
}
```

创建 `web/next-env.d.ts`：

```typescript
/// <reference types="next" />
/// <reference types="next/image-types/global" />

// This file is maintained by Next.js tooling.
```

创建 `web/postcss.config.mjs`：

```javascript
export default {
  plugins: {
    '@tailwindcss/postcss': {},
  },
};
```

创建 `web/eslint.config.mjs`：

```javascript
import { FlatCompat } from '@eslint/eslintrc';

const compat = new FlatCompat({ baseDirectory: import.meta.dirname });

export default [
  ...compat.extends('next/core-web-vitals', 'next/typescript'),
  {
    ignores: ['.next/**', 'out/**', 'coverage/**', 'next-env.d.ts', 'types/openapi.d.ts'],
    rules: {
      '@typescript-eslint/consistent-type-imports': 'error',
      '@typescript-eslint/no-explicit-any': 'error',
    },
  },
];
```

## 08-04 创建环境类型和站点配置

创建 `web/lib/site.ts`：

```typescript
export const site = {
  name: 'Rodolfo Iolo',
  description: 'Computer science, software projects, and engineering notes.',
  url: process.env.NEXT_PUBLIC_SITE_URL ?? 'http://localhost:3000',
  apiBaseURL: process.env.CONTENT_API_BASE_URL ?? 'http://localhost:8080/api/v1',
} as const;
```

创建 `web/lib/types.ts`：

```typescript
import type { components } from '@/types/openapi';

export type Article = components['schemas']['Article'];
export type ArticleDetail = components['schemas']['ArticleDetail'];
export type ArticleListResponse = components['schemas']['ArticleListResponse'];
export type Project = components['schemas']['Project'];
export type ProjectDetail = components['schemas']['ProjectDetail'];
export type ProjectListResponse = components['schemas']['ProjectListResponse'];
export type ArchiveGroup = components['schemas']['ArchiveGroup'];
export type SiteSettings = components['schemas']['SiteSettings'];
export type PublicStats = components['schemas']['PublicStats'];

export type Envelope<T> = { code: number; message: string; data: T };
```

## 08-05 创建构建期 API 客户端

创建 `web/lib/api.ts`：

```typescript
import { site } from '@/lib/site';
import type {
  ArchiveGroup,
  ArticleDetail,
  ArticleListResponse,
  Envelope,
  ProjectDetail,
  ProjectListResponse,
  PublicStats,
  SiteSettings,
} from '@/lib/types';

async function get<T>(path: string): Promise<T> {
  const response = await fetch(`${site.apiBaseURL}${path}`, {
    headers: { Accept: 'application/json' },
    cache: 'no-store',
  });
  if (!response.ok) {
    throw new Error(`Content API ${path} failed with ${response.status}`);
  }
  const envelope = (await response.json()) as Envelope<T>;
  if (envelope.code !== 0) {
    throw new Error(`Content API ${path} returned code ${envelope.code}`);
  }
  return envelope.data;
}

export const api = {
  articles: (pageSize = 20) => get<ArticleListResponse>(`/articles?page=1&page_size=${pageSize}`),
  article: (slug: string) => get<ArticleDetail>(`/articles/${encodeURIComponent(slug)}`),
  archive: () => get<ArchiveGroup[]>('/articles/archive'),
  projects: (pageSize = 20) => get<ProjectListResponse>(`/projects?page=1&page_size=${pageSize}`),
  project: (slug: string) => get<ProjectDetail>(`/projects/${encodeURIComponent(slug)}`),
  settings: () => get<SiteSettings>('/site/settings'),
  stats: () => get<PublicStats>('/site/stats'),
};
```

这里不 catch 并返回空数组。构建期内容 API 失败必须让 build 失败，保留线上旧 release。

## 08-06 创建 RSS/Sitemap 同步脚本

创建 `web/scripts/sync-public-metadata.mjs`：

```javascript
import { mkdir, writeFile } from 'node:fs/promises';

const apiBaseURL = process.env.CONTENT_API_BASE_URL ?? 'http://localhost:8080/api/v1';
const siteURL = process.env.NEXT_PUBLIC_SITE_URL ?? 'http://localhost:3000';

await mkdir(new URL('../public/', import.meta.url), { recursive: true });

for (const filename of ['rss.xml', 'sitemap.xml']) {
  const response = await fetch(`${apiBaseURL}/${filename}`);
  if (!response.ok) throw new Error(`${filename} fetch failed with ${response.status}`);
  await writeFile(new URL(`../public/${filename}`, import.meta.url), await response.text(), 'utf8');
}

const robots = `User-agent: *\nAllow: /\nDisallow: /admin/\nSitemap: ${siteURL}/sitemap.xml\n`;
await writeFile(new URL('../public/robots.txt', import.meta.url), robots, 'utf8');
```

`rss.xml`、`sitemap.xml`、`robots.txt` 是构建生成物，加入 `.gitignore`。在根 `.gitignore` 的 Node 段追加：

```gitignore
web/public/rss.xml
web/public/sitemap.xml
web/public/robots.txt
```

## 08-07 创建全局 Design Tokens

创建 `web/app/globals.css`：

```css
@import 'tailwindcss';

@theme {
  --color-ink: #18201d;
  --color-paper: #f5f7f2;
  --color-surface: #ffffff;
  --color-muted: #5f6b65;
  --color-line: #d6ddd7;
  --color-moss: #2f6b4f;
  --color-sky: #397b9c;
  --color-coral: #c75b4a;
  --font-sans: 'Noto Sans SC', 'Segoe UI', system-ui, sans-serif;
  --font-display: 'Space Grotesk', 'Segoe UI', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', Consolas, monospace;
}

:root {
  color-scheme: light;
  --page-bg: #f5f7f2;
  --panel-bg: rgba(255, 255, 255, 0.82);
  --text: #18201d;
  --text-muted: #5f6b65;
  --border: rgba(24, 32, 29, 0.14);
  --focus: #397b9c;
  --shadow: 0 12px 30px rgba(22, 42, 32, 0.12);
}

@media (prefers-color-scheme: dark) {
  :root {
    color-scheme: dark;
    --page-bg: #111714;
    --panel-bg: rgba(21, 31, 26, 0.86);
    --text: #eff4ef;
    --text-muted: #abb8b0;
    --border: rgba(239, 244, 239, 0.16);
    --focus: #7fc4de;
    --shadow: 0 14px 36px rgba(0, 0, 0, 0.3);
  }
}

* {
  box-sizing: border-box;
}
html {
  scroll-behavior: smooth;
}
body {
  margin: 0;
  min-height: 100vh;
  background: var(--page-bg);
  color: var(--text);
  font-family: var(--font-sans);
  line-height: 1.7;
  letter-spacing: 0;
}
a {
  color: inherit;
  text-decoration-thickness: 1px;
  text-underline-offset: 4px;
}
img {
  display: block;
  max-width: 100%;
}
button,
input,
textarea,
select {
  font: inherit;
}
:focus-visible {
  outline: 3px solid var(--focus);
  outline-offset: 3px;
}
.glass {
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--panel-bg);
  box-shadow: var(--shadow);
  backdrop-filter: blur(14px) saturate(120%);
}
.prose {
  max-width: 72ch;
}
.prose h2,
.prose h3 {
  font-family: var(--font-display);
  line-height: 1.25;
}
.prose pre {
  overflow-x: auto;
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 1rem;
  background: #101714;
  color: #eef6ef;
}
.prose code {
  font-family: var(--font-mono);
}
.prose :not(pre) > code {
  border-radius: 4px;
  padding: 0.12rem 0.34rem;
  background: color-mix(in srgb, var(--text) 9%, transparent);
}
@media (prefers-reduced-motion: reduce) {
  html {
    scroll-behavior: auto;
  }
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

颜色不由单一紫/蓝色统治；绿色、天空蓝和珊瑚色只作功能强调。固定面板圆角不超过 8px。

## 08-08 创建根布局

创建 `web/app/layout.tsx`：

```tsx
import type { Metadata } from 'next';
import type { ReactNode } from 'react';
import '@/app/globals.css';
import { site } from '@/lib/site';

export const metadata: Metadata = {
  metadataBase: new URL(site.url),
  title: { default: site.name, template: `%s | ${site.name}` },
  description: site.description,
  openGraph: {
    type: 'website',
    siteName: site.name,
    title: site.name,
    description: site.description,
    url: site.url,
  },
  twitter: { card: 'summary_large_image', title: site.name, description: site.description },
  alternates: { types: { 'application/rss+xml': `${site.url}/rss.xml` } },
};

export default function RootLayout({ children }: Readonly<{ children: ReactNode }>) {
  return (
    <html lang="zh-CN">
      <body>{children}</body>
    </html>
  );
}
```

## 08-09 创建基础 UI

创建 `web/components/ui/container.tsx`：

```tsx
import type { ReactNode } from 'react';
export function Container({
  children,
  className = '',
}: {
  children: ReactNode;
  className?: string;
}) {
  return <div className={`mx-auto w-full max-w-6xl px-5 sm:px-8 ${className}`}>{children}</div>;
}
```

创建 `web/components/ui/button-link.tsx`：

```tsx
import Link from 'next/link';
import type { ReactNode } from 'react';
export function ButtonLink({
  href,
  children,
  secondary = false,
}: {
  href: string;
  children: ReactNode;
  secondary?: boolean;
}) {
  const color = secondary
    ? 'border-[var(--border)] bg-[var(--panel-bg)]'
    : 'border-moss bg-moss text-white';
  return (
    <Link
      className={`inline-flex min-h-11 items-center justify-center border px-4 py-2 font-medium no-underline transition hover:-translate-y-0.5 ${color}`}
      href={href}
    >
      {children}
    </Link>
  );
}
```

创建 `web/components/ui/tag.tsx`：

```tsx
import type { ReactNode } from 'react';
export function Tag({ children }: { children: ReactNode }) {
  return (
    <span className="inline-flex items-center border border-[var(--border)] bg-[var(--panel-bg)] px-2 py-1 text-xs text-[var(--text-muted)]">
      {children}
    </span>
  );
}
```

## 08-10 创建 Header 和 Footer

创建 `web/components/layout/header.tsx`：

```tsx
import Link from 'next/link';
import { Container } from '@/components/ui/container';
const links = [
  ['文章', '/articles/'],
  ['项目', '/projects/'],
  ['归档', '/archive/'],
  ['关于', '/about/'],
] as const;
export function Header() {
  return (
    <header className="sticky top-0 z-40 border-b border-[var(--border)] bg-[color-mix(in_srgb,var(--page-bg)_86%,transparent)] backdrop-blur-xl">
      <Container className="flex min-h-16 items-center justify-between gap-6">
        <Link className="font-display text-lg font-semibold no-underline" href="/">
          Rodolfo Iolo
        </Link>
        <nav aria-label="主导航">
          <ul className="flex flex-wrap items-center justify-end gap-x-5 gap-y-2 text-sm">
            {links.map(([label, href]) => (
              <li key={href}>
                <Link href={href}>{label}</Link>
              </li>
            ))}
          </ul>
        </nav>
      </Container>
    </header>
  );
}
```

创建 `web/components/layout/footer.tsx`：

```tsx
import { Container } from '@/components/ui/container';
import type { SiteSettings } from '@/lib/types';
export function Footer({ settings }: { settings: SiteSettings }) {
  return (
    <footer className="border-t border-[var(--border)] py-10 text-sm text-[var(--text-muted)]">
      <Container className="flex flex-wrap items-center justify-between gap-4">
        <p>{settings.footer_text}</p>
        <p>
          © {new Date().getUTCFullYear()} {settings.site_title}
        </p>
      </Container>
    </footer>
  );
}
```

创建 `web/app/(public)/layout.tsx`：

```tsx
import type { ReactNode } from 'react';
import { Footer } from '@/components/layout/footer';
import { Header } from '@/components/layout/header';
import { api } from '@/lib/api';
export default async function PublicLayout({ children }: { children: ReactNode }) {
  const settings = await api.settings();
  return (
    <>
      <Header />
      {children}
      <Footer settings={settings} />
    </>
  );
}
```

## 08-11 创建内容卡片和 Markdown

创建 `web/components/cards/article-card.tsx`：

```tsx
import Link from 'next/link';
import { Tag } from '@/components/ui/tag';
import type { Article } from '@/lib/types';
export function ArticleCard({ article }: { article: Article }) {
  return (
    <article className="glass grid min-h-52 content-between gap-5 p-5 transition hover:-translate-y-1">
      <div>
        <p className="text-sm text-[var(--text-muted)]">
          {article.published_at
            ? new Intl.DateTimeFormat('zh-CN', { dateStyle: 'medium' }).format(
                new Date(article.published_at),
              )
            : '未发布'}
        </p>
        <h2 className="mt-2 font-display text-2xl font-semibold">
          <Link href={`/articles/${article.slug}/`}>{article.title}</Link>
        </h2>
        <p className="mt-3 text-[var(--text-muted)]">{article.summary}</p>
      </div>
      <ul className="flex flex-wrap gap-2" aria-label="文章标签">
        {article.tags.map((tag) => (
          <li key={tag.id}>
            <Tag>{tag.name}</Tag>
          </li>
        ))}
      </ul>
    </article>
  );
}
```

创建 `web/components/cards/project-card.tsx`：

```tsx
import Link from 'next/link';
import { Tag } from '@/components/ui/tag';
import type { Project } from '@/lib/types';
export function ProjectCard({ project }: { project: Project }) {
  return (
    <article className="glass grid min-h-64 content-between gap-5 p-5 transition hover:-translate-y-1">
      <div>
        <p className="text-sm font-medium text-coral">{project.featured ? '精选项目' : '项目'}</p>
        <h2 className="mt-2 font-display text-2xl font-semibold">
          <Link href={`/projects/${project.slug}/`}>{project.name}</Link>
        </h2>
        <p className="mt-3 text-[var(--text-muted)]">{project.summary}</p>
      </div>
      <ul className="flex flex-wrap gap-2" aria-label="技术栈">
        {project.technologies.map((technology) => (
          <li key={technology}>
            <Tag>{technology}</Tag>
          </li>
        ))}
      </ul>
    </article>
  );
}
```

创建 `web/components/markdown/markdown-content.tsx`：

```tsx
import ReactMarkdown from 'react-markdown';
import rehypeSanitize from 'rehype-sanitize';
import remarkGfm from 'remark-gfm';
export function MarkdownContent({ source }: { source: string }) {
  return (
    <div className="prose">
      <ReactMarkdown remarkPlugins={[remarkGfm]} rehypePlugins={[rehypeSanitize]}>
        {source}
      </ReactMarkdown>
    </div>
  );
}
```

Markdown 默认不允许原始 HTML；`rehype-sanitize` 是第二道防线。

## 08-12 创建首页

创建 `web/app/(public)/page.tsx`：

```tsx
import { ArticleCard } from '@/components/cards/article-card';
import { ProjectCard } from '@/components/cards/project-card';
import { ButtonLink } from '@/components/ui/button-link';
import { Container } from '@/components/ui/container';
import { api } from '@/lib/api';

export default async function HomePage() {
  const [articles, projects, stats] = await Promise.all([
    api.articles(3),
    api.projects(3),
    api.stats(),
  ]);
  return (
    <main>
      <section className="flex min-h-[72vh] items-center border-b border-[var(--border)] py-20">
        <Container>
          <p className="font-medium text-moss">计算机专业学生 · 独立开发者</p>
          <h1 className="mt-4 max-w-4xl font-display text-5xl font-semibold leading-tight sm:text-6xl">
            Rodolfo Iolo
          </h1>
          <p className="mt-6 max-w-2xl text-lg text-[var(--text-muted)]">
            记录软件工程实践、项目复盘，以及把想法从空仓库推进到生产环境的全过程。
          </p>
          <div className="mt-8 flex flex-wrap gap-3">
            <ButtonLink href="/projects/">查看项目</ButtonLink>
            <ButtonLink href="/articles/" secondary>
              阅读文章
            </ButtonLink>
          </div>
          <dl className="mt-12 grid max-w-2xl grid-cols-2 gap-4 sm:grid-cols-4">
            {[
              ['文章', stats.total_articles],
              ['项目', stats.total_projects],
              ['分类', stats.total_categories],
              ['标签', stats.total_tags],
            ].map(([label, value]) => (
              <div key={label} className="border-l-2 border-moss pl-3">
                <dt className="text-sm text-[var(--text-muted)]">{label}</dt>
                <dd className="m-0 font-display text-2xl font-semibold">{value}</dd>
              </div>
            ))}
          </dl>
        </Container>
      </section>
      <section className="py-20">
        <Container>
          <div className="flex items-end justify-between gap-4">
            <div>
              <p className="text-sm text-coral">Writing</p>
              <h2 className="font-display text-3xl font-semibold">最新文章</h2>
            </div>
            <a href="/articles/">全部文章</a>
          </div>
          <div className="mt-8 grid gap-5 lg:grid-cols-3">
            {articles.items.length ? (
              articles.items.map((article) => <ArticleCard key={article.id} article={article} />)
            ) : (
              <p>文章正在准备中。</p>
            )}
          </div>
        </Container>
      </section>
      <section className="border-t border-[var(--border)] py-20">
        <Container>
          <div>
            <p className="text-sm text-sky">Selected work</p>
            <h2 className="font-display text-3xl font-semibold">项目实践</h2>
          </div>
          <div className="mt-8 grid gap-5 md:grid-cols-2 lg:grid-cols-3">
            {projects.items.length ? (
              projects.items.map((project) => <ProjectCard key={project.id} project={project} />)
            ) : (
              <p>项目正在整理中。</p>
            )}
          </div>
        </Container>
      </section>
    </main>
  );
}
```

首屏直接出现姓名和身份，且高度为 72vh，桌面和移动端都能看到下一节提示，不做整屏封死的 Hero。

## 08-13 创建文章列表和详情

创建 `web/app/(public)/articles/page.tsx`：

```tsx
import type { Metadata } from 'next';
import { ArticleCard } from '@/components/cards/article-card';
import { Container } from '@/components/ui/container';
import { api } from '@/lib/api';

export const metadata: Metadata = {
  title: '文章',
  description: '软件工程、计算机科学与项目复盘文章。',
};
export default async function ArticlesPage() {
  const page = await api.articles(100);
  return (
    <main className="py-16">
      <Container>
        <h1 className="font-display text-4xl font-semibold">文章</h1>
        <p className="mt-3 text-[var(--text-muted)]">按时间记录问题、取舍和验证过程。</p>
        <div className="mt-10 grid gap-5 md:grid-cols-2">
          {page.items.length ? (
            page.items.map((article) => <ArticleCard key={article.id} article={article} />)
          ) : (
            <p>还没有已发布文章。</p>
          )}
        </div>
      </Container>
    </main>
  );
}
```

创建 `web/app/(public)/articles/[slug]/page.tsx`：

```tsx
import type { Metadata } from 'next';
import { notFound } from 'next/navigation';
import { MarkdownContent } from '@/components/markdown/markdown-content';
import { Container } from '@/components/ui/container';
import { Tag } from '@/components/ui/tag';
import { api } from '@/lib/api';
import { site } from '@/lib/site';

type Props = { params: Promise<{ slug: string }> };
export async function generateStaticParams() {
  const page = await api.articles(100);
  return page.items.map(({ slug }) => ({ slug }));
}
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  try {
    const article = await api.article(slug);
    return {
      title: article.title,
      description: article.summary,
      alternates: { canonical: `/articles/${article.slug}/` },
      openGraph: {
        type: 'article',
        title: article.title,
        description: article.summary,
        publishedTime: article.published_at ?? undefined,
        modifiedTime: article.updated_at,
      },
    };
  } catch {
    return { title: '文章不存在' };
  }
}
export default async function ArticlePage({ params }: Props) {
  const { slug } = await params;
  let article;
  try {
    article = await api.article(slug);
  } catch {
    notFound();
  }
  const structured = {
    '@context': 'https://schema.org',
    '@type': 'Article',
    headline: article.title,
    description: article.summary,
    datePublished: article.published_at,
    dateModified: article.updated_at,
    author: { '@type': 'Person', name: article.author.nickname },
    mainEntityOfPage: `${site.url}/articles/${article.slug}/`,
  };
  return (
    <main className="py-16">
      <Container className="max-w-4xl">
        <article>
          <header className="border-b border-[var(--border)] pb-8">
            <p className="text-sm text-[var(--text-muted)]">
              {article.published_at
                ? new Intl.DateTimeFormat('zh-CN', { dateStyle: 'long' }).format(
                    new Date(article.published_at),
                  )
                : ''}
            </p>
            <h1 className="mt-3 font-display text-4xl font-semibold leading-tight sm:text-5xl">
              {article.title}
            </h1>
            <p className="mt-5 text-lg text-[var(--text-muted)]">{article.summary}</p>
            <ul className="mt-5 flex flex-wrap gap-2">
              {article.tags.map((tag) => (
                <li key={tag.id}>
                  <Tag>{tag.name}</Tag>
                </li>
              ))}
            </ul>
          </header>
          <div className="mt-10">
            <MarkdownContent source={article.content} />
          </div>
        </article>
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{
            __html: JSON.stringify(structured).replaceAll('<', '\\u003c'),
          }}
        />
      </Container>
    </main>
  );
}
```

## 08-14 创建项目列表和详情

创建 `web/app/(public)/projects/page.tsx`：

```tsx
import type { Metadata } from 'next';
import { ProjectCard } from '@/components/cards/project-card';
import { Container } from '@/components/ui/container';
import { api } from '@/lib/api';
export const metadata: Metadata = {
  title: '项目',
  description: '从需求、架构、实现到部署的项目实践。',
};
export default async function ProjectsPage() {
  const page = await api.projects(100);
  return (
    <main className="py-16">
      <Container>
        <h1 className="font-display text-4xl font-semibold">项目</h1>
        <p className="mt-3 text-[var(--text-muted)]">不仅展示结果，也记录工程过程与关键取舍。</p>
        <div className="mt-10 grid gap-5 md:grid-cols-2">
          {page.items.length ? (
            page.items.map((project) => <ProjectCard key={project.id} project={project} />)
          ) : (
            <p>还没有公开项目。</p>
          )}
        </div>
      </Container>
    </main>
  );
}
```

创建 `web/app/(public)/projects/[slug]/page.tsx`：

```tsx
import type { Metadata } from 'next';
import { notFound } from 'next/navigation';
import { ExternalLink, Github } from 'lucide-react';
import { MarkdownContent } from '@/components/markdown/markdown-content';
import { Container } from '@/components/ui/container';
import { Tag } from '@/components/ui/tag';
import { api } from '@/lib/api';
import { site } from '@/lib/site';

type Props = { params: Promise<{ slug: string }> };
export async function generateStaticParams() {
  const page = await api.projects(100);
  return page.items.map(({ slug }) => ({ slug }));
}
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  try {
    const project = await api.project(slug);
    return {
      title: project.name,
      description: project.summary,
      alternates: { canonical: `/projects/${project.slug}/` },
    };
  } catch {
    return { title: '项目不存在' };
  }
}
export default async function ProjectPage({ params }: Props) {
  const { slug } = await params;
  let project;
  try {
    project = await api.project(slug);
  } catch {
    notFound();
  }
  const structured = {
    '@context': 'https://schema.org',
    '@type': 'SoftwareSourceCode',
    name: project.name,
    description: project.summary,
    codeRepository: project.repository_url,
    url: `${site.url}/projects/${project.slug}/`,
    programmingLanguage: project.technologies,
  };
  return (
    <main className="py-16">
      <Container className="max-w-4xl">
        <article>
          <header className="border-b border-[var(--border)] pb-8">
            <p className="text-sm text-coral">{project.featured ? '精选项目' : '项目'}</p>
            <h1 className="mt-3 font-display text-4xl font-semibold sm:text-5xl">{project.name}</h1>
            <p className="mt-5 text-lg text-[var(--text-muted)]">{project.summary}</p>
            <ul className="mt-5 flex flex-wrap gap-2">
              {project.technologies.map((technology) => (
                <li key={technology}>
                  <Tag>{technology}</Tag>
                </li>
              ))}
            </ul>
            <div className="mt-6 flex gap-4">
              {project.repository_url ? (
                <a
                  className="inline-flex items-center gap-2"
                  href={project.repository_url}
                  rel="noreferrer"
                  target="_blank"
                >
                  <Github size={18} aria-hidden />
                  源码
                </a>
              ) : null}
              {project.demo_url ? (
                <a
                  className="inline-flex items-center gap-2"
                  href={project.demo_url}
                  rel="noreferrer"
                  target="_blank"
                >
                  <ExternalLink size={18} aria-hidden />
                  演示
                </a>
              ) : null}
            </div>
          </header>
          <div className="mt-10">
            <MarkdownContent source={project.content} />
          </div>
        </article>
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{
            __html: JSON.stringify(structured).replaceAll('<', '\\u003c'),
          }}
        />
      </Container>
    </main>
  );
}
```

## 08-15 创建归档、关于和 404

创建 `web/app/(public)/archive/page.tsx`：

```tsx
import type { Metadata } from 'next';
import Link from 'next/link';
import { Container } from '@/components/ui/container';
import { api } from '@/lib/api';
export const metadata: Metadata = { title: '归档', description: '按年份和月份浏览所有文章。' };
export default async function ArchivePage() {
  const groups = await api.archive();
  return (
    <main className="py-16">
      <Container className="max-w-4xl">
        <h1 className="font-display text-4xl font-semibold">归档</h1>
        <div className="mt-10 space-y-12">
          {groups.length ? (
            groups.map((year) => (
              <section key={year.year}>
                <h2 className="font-display text-3xl font-semibold">{year.year}</h2>
                {year.months.map((month) => (
                  <div className="mt-6 grid gap-3 sm:grid-cols-[5rem_1fr]" key={month.month}>
                    <h3 className="text-[var(--text-muted)]">{month.month} 月</h3>
                    <ul className="space-y-3">
                      {month.articles.map((article) => (
                        <li
                          className="flex flex-wrap justify-between gap-3 border-b border-[var(--border)] pb-3"
                          key={article.slug}
                        >
                          <Link href={`/articles/${article.slug}/`}>{article.title}</Link>
                          <time className="text-sm text-[var(--text-muted)]">
                            {new Intl.DateTimeFormat('zh-CN', {
                              month: '2-digit',
                              day: '2-digit',
                            }).format(new Date(article.published_at))}
                          </time>
                        </li>
                      ))}
                    </ul>
                  </div>
                ))}
              </section>
            ))
          ) : (
            <p>还没有归档文章。</p>
          )}
        </div>
      </Container>
    </main>
  );
}
```

创建 `web/app/(public)/about/page.tsx`：

```tsx
import type { Metadata } from 'next';
import { MarkdownContent } from '@/components/markdown/markdown-content';
import { Container } from '@/components/ui/container';
import { api } from '@/lib/api';
export const metadata: Metadata = { title: '关于', description: '关于我、学习方向和联系方式。' };
export default async function AboutPage() {
  const settings = await api.settings();
  return (
    <main className="py-16">
      <Container className="max-w-4xl">
        <h1 className="font-display text-4xl font-semibold">关于</h1>
        <div className="mt-8">
          <MarkdownContent source={settings.about_content} />
        </div>
        <ul className="mt-10 flex flex-wrap gap-5">
          {settings.social_links.map((link) => (
            <li key={link.url}>
              <a href={link.url} rel="me noreferrer" target="_blank">
                {link.name}
              </a>
            </li>
          ))}
        </ul>
      </Container>
    </main>
  );
}
```

创建 `web/app/not-found.tsx`：

```tsx
import Link from 'next/link';
export default function NotFound() {
  return (
    <main className="grid min-h-screen place-items-center px-5 text-center">
      <div>
        <p className="font-display text-6xl font-semibold">404</p>
        <h1 className="mt-4 text-2xl font-semibold">没有找到这个页面</h1>
        <p className="mt-3 text-[var(--text-muted)]">地址可能已变化，或者内容尚未发布。</p>
        <Link className="mt-6 inline-block" href="/">
          返回首页
        </Link>
      </div>
    </main>
  );
}
```

## 08-16 创建 Web App Manifest

创建 `web/public/manifest.webmanifest`：

```json
{
  "name": "Rodolfo Iolo",
  "short_name": "Rodolfo",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#f5f7f2",
  "theme_color": "#2f6b4f",
  "icons": []
}
```

第 11 章生成 favicon 后再补 icons；当前不引用不存在的二进制文件。

## 08-17 创建本地 Web 启动脚本

创建 `scripts/run-web.ps1`：

```powershell
[CmdletBinding()]
param()

$ErrorActionPreference = 'Stop'
$repositoryRoot = Split-Path -Parent $PSScriptRoot
$env:CONTENT_API_BASE_URL = 'http://localhost:8080/api/v1'
$env:NEXT_PUBLIC_API_BASE_URL = 'http://localhost:8080/api/v1'
$env:NEXT_PUBLIC_SITE_URL = 'http://localhost:3000'

Push-Location (Join-Path $repositoryRoot 'web')
try {
    pnpm dev
    if ($LASTEXITCODE -ne 0) { throw 'Next.js development server failed.' }
}
finally {
    Pop-Location
}
```

## 08-18 运行静态分析

```powershell
PS> pnpm install --frozen-lockfile
PS> pnpm --dir web lint
PS> pnpm --dir web typecheck
PS> pnpm --dir web format
PS> pnpm --dir web format:check
PS> git diff --check
```

`next-env.d.ts` 若被当前 Next 版本机械更新，以生成结果为准并提交；不要在里面写业务类型。

## 08-19 启动开发环境并人工检查

终端 1：

```powershell
PS> docker compose -f compose.dev.yml up -d --wait postgres
PS> .\scripts\run-api.ps1
```

终端 2：

```powershell
PS> .\scripts\run-web.ps1
```

浏览器访问 `http://localhost:3000`，逐页检查：

1. 首页姓名、简介、文章和项目；
2. `/articles/` 与 `/articles/welcome/`；
3. `/projects/`；如果无项目，空状态正确；
4. `/archive/` 和 `/about/`；
5. Tab 键可到达所有链接，焦点环清楚；
6. 375px 窄窗口无横向滚动；
7. 深色系统主题文字仍可读。

浏览器开发者工具 Console 必须无 hydration error、key warning 和资源 404。

## 08-20 构建静态站点

保持 API 运行，另开终端：

```powershell
PS> $env:CONTENT_API_BASE_URL='http://localhost:8080/api/v1'
PS> $env:NEXT_PUBLIC_API_BASE_URL='http://localhost:8080/api/v1'
PS> $env:NEXT_PUBLIC_SITE_URL='http://localhost:3000'
PS> pnpm --dir web build
PS> Remove-Item Env:CONTENT_API_BASE_URL,Env:NEXT_PUBLIC_API_BASE_URL,Env:NEXT_PUBLIC_SITE_URL
PS> Get-ChildItem -LiteralPath 'web\out' | Select-Object Name
PS> Select-String -LiteralPath 'web\out\index.html' -Pattern 'Rodolfo Iolo'
PS> Select-String -LiteralPath 'web\out\articles\welcome\index.html' -Pattern 'The first published article'
PS> Get-Item -LiteralPath 'web\out\rss.xml','web\out\sitemap.xml','web\out\robots.txt' | Select-Object Name,Length
```

预期：build 退出 0；首页和文章正文存在于 HTML，不依赖浏览器执行后再出现；三个 SEO 文件非空。

停止 API 后再次 build 应失败。验证失败后重新启动 API，不需要提交这次失败产生的 `.next`。

## 08-21 添加 Frontend CI

在 `.github/workflows/ci.yml` 的 `jobs:` 下追加：

```yaml
frontend:
  name: Frontend checks
  runs-on: ubuntu-latest
  timeout-minutes: 15
  steps:
    - name: Check out repository
      uses: actions/checkout@v5

    - name: Set up Node.js
      uses: actions/setup-node@v5
      with:
        node-version-file: .node-version

    - name: Install pnpm
      run: corepack install --global pnpm@11.19.0

    - name: Install dependencies
      run: pnpm install --frozen-lockfile

    - name: Lint
      run: pnpm --dir web lint

    - name: Type check
      run: pnpm --dir web typecheck

    - name: Check formatting
      run: pnpm --dir web format:check
```

这里暂不运行 `next build`，因为它需要迁移后的 PostgreSQL 和真实 API。第 13 章添加 production-shaped build Job，不会使用前端 mock。

## 08-22 提交与 PR

```powershell
PS> git status --short
PS> git diff --check
PS> git add -- web/package.json web/next.config.mjs web/tsconfig.json web/next-env.d.ts web/postcss.config.mjs web/eslint.config.mjs web/lib web/scripts web/app/layout.tsx web/app/globals.css web/app/not-found.tsx package.json pnpm-lock.yaml .gitignore
PS> git commit -m 'feat(web): establish static public site foundation'
PS> git add -- 'web/app/(public)' web/components web/public/manifest.webmanifest scripts/run-web.ps1 .github/workflows/ci.yml
PS> git commit -m 'feat(web): add content pages and SEO metadata'
PS> git push --set-upstream origin feat/public-web
```

PowerShell 的括号路径必须放在单引号中。PR 验证栏记录 lint、typecheck、build、HTML 正文检查、键盘和 375px 检查。CI 全绿后合并，把 Frontend checks 加入 main 门禁并清理分支。

## 08-23 本章停止点

- [ ] 所有公共页从真实 API 构建；
- [ ] API 停止时 build 失败；
- [ ] 文章正文存在于导出 HTML；
- [ ] 动态文章/项目参数全部预生成；
- [ ] RSS、Sitemap、robots 进入 out；
- [ ] metadata、canonical 和结构化数据存在；
- [ ] lint/typecheck/format/build 全通过；
- [ ] 375px、键盘、深色和空状态可用；
- [ ] Frontend checks 已成为 main 门禁。
