# 第 07 章：内容领域与完整 API

本章实现 OpenAPI 中除认证外的所有后端 operation，并在结尾首次注册 Strict Server。不能跳过资源，也不能保留 501。互动前端在第 12 章完成，但其后端契约本章就绪。

## 07-01 创建分支、目录和依赖

```powershell
PS> Set-Location 'D:\CODEkingdom\rodolfoiolo'
PS> git switch main
PS> git pull --ff-only
PS> git status --short --branch
PS> git switch -c feat/content-api
PS> New-Item -ItemType Directory -Force -Path 'server\internal\storage','server\tests' | Out-Null
PS> Set-Location 'server'
PS> go get github.com/gosimple/slug@v1.15.0
PS> go get github.com/microcosm-cc/bluemonday@v1.0.27
PS> go mod tidy
PS> Set-Location '..'
```

## 07-02 创建领域模型

创建 `server/internal/domain/content.go`：

```go
package domain

import "time"

type Category struct {
	ID           int64
	Name         string
	Slug         string
	Description  *string
	Icon         *string
	ParentID     *int64
	SortOrder    int
	ArticleCount int
	CreatedAt    time.Time
}

type Tag struct {
	ID           int64
	Name         string
	Slug         string
	Color        *string
	ArticleCount int
	CreatedAt    time.Time
}

type Article struct {
	ID           int64
	Slug         string
	Title        string
	Summary      string
	Content      string
	CoverImage   *string
	Category     *Category
	Tags         []Tag
	Author       UserSummary
	Status       string
	IsTop        bool
	ViewCount    int
	LikeCount    int
	CommentCount int
	PublishedAt  *time.Time
	CreatedAt    time.Time
	UpdatedAt    time.Time
}

type UserSummary struct {
	ID        int64
	Username  string
	Nickname  string
	AvatarURL *string
	Bio       *string
	Role      string
	CreatedAt time.Time
}

type Project struct {
	ID            int64
	Slug          string
	Name          string
	Summary       string
	Content       string
	CoverImage    *string
	Technologies  []string
	RepositoryURL *string
	DemoURL       *string
	Featured      bool
	SortOrder     int
	Status        string
	StartedAt     *time.Time
	CompletedAt   *time.Time
	CreatedAt     time.Time
	UpdatedAt     time.Time
}

type Page[T any] struct {
	Items      []T
	Page       int
	PageSize   int
	Total      int
	TotalPages int
}

type Comment struct {
	ID        int64
	ArticleID int64
	ParentID  *int64
	Nickname  string
	Content   string
	Status    string
	CreatedAt time.Time
}

type SiteSettings struct {
	SiteTitle       string       `json:"site_title"`
	SiteDescription string       `json:"site_description"`
	SiteKeywords    string       `json:"site_keywords"`
	AboutContent    string       `json:"about_content"`
	FooterText      string       `json:"footer_text"`
	ICPNumber       *string      `json:"icp_number"`
	SocialLinks     []SocialLink `json:"social_links"`
}

type SocialLink struct {
	Name string `json:"name"`
	URL  string `json:"url"`
	Icon string `json:"icon"`
}

type Dashboard struct {
	TotalArticles  int
	DraftArticles  int
	TotalProjects  int
	PendingComments int
	TotalViews     int
	TotalLikes     int
}

type MediaAsset struct {
	ID           int64
	ObjectKey    string
	OriginalName string
	MIMEType     string
	ByteSize     int64
	Width        int
	Height       int
	SHA256       string
	CreatedAt    time.Time
}
```

## 07-03 创建输入和查询模型

创建 `server/internal/domain/input.go`：

```go
package domain

import "time"

type ArticleFilter struct {
	Page, PageSize int
	Category       string
	Tag            string
	Keyword        string
	Status         string
	Sort           string
	IncludePrivate bool
}

type ArticleInput struct {
	Slug       string
	Title      string
	Summary    string
	Content    string
	CoverImage *string
	CategoryID *int64
	TagIDs     []int64
	Status     string
	IsTop      bool
}

type ProjectFilter struct {
	Page, PageSize int
	Technology     string
	Status         string
	Featured       *bool
	IncludePrivate bool
}

type ProjectInput struct {
	Slug          string
	Name          string
	Summary       string
	Content       string
	CoverImage    *string
	Technologies  []string
	RepositoryURL *string
	DemoURL       *string
	Featured      bool
	SortOrder     int
	Status        string
	StartedAt     *time.Time
	CompletedAt   *time.Time
}

type CategoryInput struct {
	Name        string
	Slug        string
	Description *string
	Icon        *string
	ParentID    *int64
	SortOrder   int
}

type TagInput struct {
	Name  string
	Slug  string
	Color *string
}

type CommentInput struct {
	ArticleSlug string
	ParentID    *int64
	Nickname    string
	EmailHash   *string
	Content     string
	VisitorID   string
	IPHash      string
	UserAgent   string
}
```

## 07-04 创建统一输入校验工具

创建 `server/internal/service/validation.go`：

```go
package service

import (
	"errors"
	"net/url"
	"regexp"
	"strings"
	"unicode/utf8"

	slugger "github.com/gosimple/slug"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
)

var colorPattern = regexp.MustCompile(`^#[0-9A-Fa-f]{6}$`)

func normalizedSlug(explicit, source string) (string, error) {
	value := strings.TrimSpace(explicit)
	if value == "" {
		value = slugger.Make(source)
	} else {
		value = slugger.Make(value)
	}
	if value == "" || len(value) > 200 {
		return "", domain.ErrBadRequest
	}
	return value, nil
}

func requiredText(value string, minimum, maximum int) (string, error) {
	value = strings.TrimSpace(value)
	length := utf8.RuneCountInString(value)
	if length < minimum || length > maximum {
		return "", domain.ErrBadRequest
	}
	return value, nil
}

func optionalHTTPURL(value *string) (*string, error) {
	if value == nil || strings.TrimSpace(*value) == "" {
		return nil, nil
	}
	trimmed := strings.TrimSpace(*value)
	parsed, err := url.ParseRequestURI(trimmed)
	if err != nil || (parsed.Scheme != "http" && parsed.Scheme != "https") || parsed.Host == "" {
		return nil, domain.ErrBadRequest
	}
	return &trimmed, nil
}

func normalizedTechnologies(values []string) ([]string, error) {
	if len(values) > 20 {
		return nil, domain.ErrBadRequest
	}
	seen := make(map[string]struct{}, len(values))
	result := make([]string, 0, len(values))
	for _, value := range values {
		value = strings.TrimSpace(value)
		if value == "" || utf8.RuneCountInString(value) > 50 {
			return nil, domain.ErrBadRequest
		}
		key := strings.ToLower(value)
		if _, ok := seen[key]; ok {
			continue
		}
		seen[key] = struct{}{}
		result = append(result, value)
	}
	return result, nil
}

func normalizedPage(page, pageSize *int) (int, int, error) {
	resultPage, resultSize := 1, 20
	if page != nil && *page != 0 { resultPage = *page }
	if pageSize != nil && *pageSize != 0 { resultSize = *pageSize }
	if resultPage < 1 || resultSize < 1 || resultSize > 100 {
		return 0, 0, domain.ErrBadRequest
	}
	return resultPage, resultSize, nil
}

func validColor(color *string) error {
	if color != nil && !colorPattern.MatchString(*color) {
		return domain.ErrBadRequest
	}
	return nil
}

func statusAllowed(value string, allowed ...string) error {
	for _, candidate := range allowed {
		if value == candidate { return nil }
	}
	return errors.Join(domain.ErrBadRequest, errors.New("invalid status"))
}
```

这些函数在 Service 边界清理输入；数据库 CHECK 仍是最后防线，两层不能互相替代。

## 07-05 创建 Markdown 评论清洗器

创建 `server/internal/security/sanitize.go`：

```go
package security

import (
	"html"
	"strings"

	"github.com/microcosm-cc/bluemonday"
)

type CommentSanitizer struct {
	policy *bluemonday.Policy
}

func NewCommentSanitizer() *CommentSanitizer {
	policy := bluemonday.StrictPolicy()
	return &CommentSanitizer{policy: policy}
}

func (s *CommentSanitizer) Clean(value string) string {
	plain := s.policy.Sanitize(value)
	plain = html.UnescapeString(plain)
	return strings.TrimSpace(plain)
}
```

评论以纯文本保存和输出；文章 Markdown 由管理员写入，前端渲染时仍经过 HTML 白名单。

## 07-06 本卷文件台账

后续必须按这个顺序创建，任何一项不能以“同理”省略：

| 顺序 | 文件                        | 职责                             |
| ---: | --------------------------- | -------------------------------- |
|    1 | `repository/category.go`    | 分类 SQL 与父级约束              |
|    2 | `repository/tag.go`         | 标签 SQL 与公开计数              |
|    3 | `repository/article.go`     | 文章查询、标签事务和状态机持久化 |
|    4 | `repository/project.go`     | 项目查询与 JSONB 技术栈          |
|    5 | `repository/site.go`        | 设置、统计、RSS/Sitemap 数据     |
|    6 | `repository/interaction.go` | 评论审核、点赞和计数事务         |
|    7 | `repository/media.go`       | 媒体元数据                       |
|    8 | `service/content.go`        | 所有业务校验、权限和状态机       |
|    9 | `handler/content.go`        | OpenAPI DTO 映射                 |
|   10 | `handler/interaction.go`    | 互动和媒体 DTO 映射              |
|   11 | `handler/health_strict.go`  | 契约内健康响应                   |
|   12 | `server/server.go`          | 中间件顺序和 Strict 注册         |
|   13 | `cmd/api/main.go`           | 最终依赖装配                     |

本章后续代码量较大，执行时每创建一个文件立即运行 `gofmt` 和对应包测试；不要先粘贴完所有文件再一起排错。

## 07-07 创建分类 Repository

创建 `server/internal/repository/category.go`：

```go
package repository

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
)

type CategoryRepository struct{ pool *pgxpool.Pool }

func NewCategoryRepository(pool *pgxpool.Pool) *CategoryRepository { return &CategoryRepository{pool: pool} }

func (r *CategoryRepository) List(ctx context.Context) ([]domain.Category, error) {
	rows, err := r.pool.Query(ctx, `
		SELECT c.id, c.name, c.slug, c.description, c.icon, c.parent_id, c.sort_order, c.created_at,
		       count(a.id) FILTER (WHERE a.status = 'published')::int
		FROM categories c
		LEFT JOIN articles a ON a.category_id = c.id
		GROUP BY c.id
		ORDER BY c.sort_order, lower(c.name)`)
	if err != nil { return nil, fmt.Errorf("list categories: %w", err) }
	defer rows.Close()
	items := make([]domain.Category, 0)
	for rows.Next() {
		item, err := scanCategory(rows)
		if err != nil { return nil, err }
		items = append(items, item)
	}
	return items, rows.Err()
}

func (r *CategoryRepository) GetBySlug(ctx context.Context, slug string) (domain.Category, error) {
	return scanCategory(r.pool.QueryRow(ctx, `
		SELECT c.id, c.name, c.slug, c.description, c.icon, c.parent_id, c.sort_order, c.created_at,
		       count(a.id) FILTER (WHERE a.status = 'published')::int
		FROM categories c
		LEFT JOIN articles a ON a.category_id = c.id
		WHERE c.slug = $1 GROUP BY c.id`, slug))
}

func (r *CategoryRepository) Create(ctx context.Context, input domain.CategoryInput) (domain.Category, error) {
	item, err := scanCategory(r.pool.QueryRow(ctx, `
		INSERT INTO categories (name, slug, description, icon, parent_id, sort_order)
		VALUES ($1, $2, $3, $4, $5, $6)
		RETURNING id, name, slug, description, icon, parent_id, sort_order, created_at, 0`,
		input.Name, input.Slug, input.Description, input.Icon, input.ParentID, input.SortOrder))
	if isUniqueViolation(err) { return domain.Category{}, domain.ErrConflict }
	return item, err
}

func (r *CategoryRepository) Update(ctx context.Context, currentSlug string, input domain.CategoryInput) (domain.Category, error) {
	item, err := scanCategory(r.pool.QueryRow(ctx, `
		UPDATE categories SET name=$2, slug=$3, description=$4, icon=$5, parent_id=$6, sort_order=$7, updated_at=now()
		WHERE slug=$1
		RETURNING id, name, slug, description, icon, parent_id, sort_order, created_at,
		  (SELECT count(*)::int FROM articles WHERE category_id=categories.id AND status='published')`,
		currentSlug, input.Name, input.Slug, input.Description, input.Icon, input.ParentID, input.SortOrder))
	if isUniqueViolation(err) { return domain.Category{}, domain.ErrConflict }
	return item, err
}

func (r *CategoryRepository) Delete(ctx context.Context, slug string) error {
	command, err := r.pool.Exec(ctx, `DELETE FROM categories WHERE slug=$1`, slug)
	if err != nil { return fmt.Errorf("delete category: %w", err) }
	if command.RowsAffected() == 0 { return domain.ErrNotFound }
	return nil
}

func (r *CategoryRepository) ParentDepth(ctx context.Context, parentID int64) (int, error) {
	var parentParent *int64
	err := r.pool.QueryRow(ctx, `SELECT parent_id FROM categories WHERE id=$1`, parentID).Scan(&parentParent)
	if errors.Is(err, pgx.ErrNoRows) { return 0, domain.ErrBadRequest }
	if err != nil { return 0, fmt.Errorf("read category parent: %w", err) }
	if parentParent == nil { return 1, nil }
	return 2, nil
}

func scanCategory(row rowScanner) (domain.Category, error) {
	var item domain.Category
	err := row.Scan(&item.ID, &item.Name, &item.Slug, &item.Description, &item.Icon, &item.ParentID, &item.SortOrder, &item.CreatedAt, &item.ArticleCount)
	if errors.Is(err, pgx.ErrNoRows) { return domain.Category{}, domain.ErrNotFound }
	if err != nil { return domain.Category{}, fmt.Errorf("scan category: %w", err) }
	return item, nil
}
```

## 07-08 创建标签 Repository

创建 `server/internal/repository/tag.go`：

```go
package repository

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
)

type TagRepository struct{ pool *pgxpool.Pool }

func NewTagRepository(pool *pgxpool.Pool) *TagRepository { return &TagRepository{pool: pool} }

func (r *TagRepository) List(ctx context.Context) ([]domain.Tag, error) {
	rows, err := r.pool.Query(ctx, `
		SELECT t.id, t.name, t.slug, t.color, t.created_at,
		       count(a.id) FILTER (WHERE a.status='published')::int
		FROM tags t
		LEFT JOIN article_tags at ON at.tag_id=t.id
		LEFT JOIN articles a ON a.id=at.article_id
		GROUP BY t.id ORDER BY lower(t.name)`)
	if err != nil { return nil, fmt.Errorf("list tags: %w", err) }
	defer rows.Close()
	items := make([]domain.Tag, 0)
	for rows.Next() {
		item, err := scanTag(rows)
		if err != nil { return nil, err }
		items = append(items, item)
	}
	return items, rows.Err()
}

func (r *TagRepository) GetBySlug(ctx context.Context, slug string) (domain.Tag, error) {
	return scanTag(r.pool.QueryRow(ctx, `
		SELECT t.id, t.name, t.slug, t.color, t.created_at,
		       count(a.id) FILTER (WHERE a.status='published')::int
		FROM tags t
		LEFT JOIN article_tags at ON at.tag_id=t.id
		LEFT JOIN articles a ON a.id=at.article_id
		WHERE t.slug=$1 GROUP BY t.id`, slug))
}

func (r *TagRepository) Create(ctx context.Context, input domain.TagInput) (domain.Tag, error) {
	item, err := scanTag(r.pool.QueryRow(ctx, `
		INSERT INTO tags (name, slug, color) VALUES ($1,$2,$3)
		RETURNING id,name,slug,color,created_at,0`, input.Name, input.Slug, input.Color))
	if isUniqueViolation(err) { return domain.Tag{}, domain.ErrConflict }
	return item, err
}

func (r *TagRepository) Update(ctx context.Context, currentSlug string, input domain.TagInput) (domain.Tag, error) {
	item, err := scanTag(r.pool.QueryRow(ctx, `
		UPDATE tags SET name=$2,slug=$3,color=$4,updated_at=now() WHERE slug=$1
		RETURNING id,name,slug,color,created_at,
		  (SELECT count(*)::int FROM article_tags at JOIN articles a ON a.id=at.article_id WHERE at.tag_id=tags.id AND a.status='published')`,
		currentSlug, input.Name, input.Slug, input.Color))
	if isUniqueViolation(err) { return domain.Tag{}, domain.ErrConflict }
	return item, err
}

func (r *TagRepository) Delete(ctx context.Context, slug string) error {
	command, err := r.pool.Exec(ctx, `DELETE FROM tags WHERE slug=$1`, slug)
	if err != nil { return fmt.Errorf("delete tag: %w", err) }
	if command.RowsAffected() == 0 { return domain.ErrNotFound }
	return nil
}

func scanTag(row rowScanner) (domain.Tag, error) {
	var item domain.Tag
	err := row.Scan(&item.ID, &item.Name, &item.Slug, &item.Color, &item.CreatedAt, &item.ArticleCount)
	if errors.Is(err, pgx.ErrNoRows) { return domain.Tag{}, domain.ErrNotFound }
	if err != nil { return domain.Tag{}, fmt.Errorf("scan tag: %w", err) }
	return item, nil
}
```

执行：

```powershell
PS> Set-Location 'server'
PS> gofmt -w internal/domain internal/repository/category.go internal/repository/tag.go internal/service/validation.go internal/security/sanitize.go
PS> go test ./internal/domain ./internal/repository ./internal/service ./internal/security
PS> Set-Location '..'
```

## 07-09 创建文章 Repository

创建 `server/internal/repository/article.go`：

```go
package repository

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"strings"
	"time"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
)

const articleSelect = `
	SELECT a.id,a.slug,a.title,a.summary,a.content,a.cover_image,a.status,a.is_top,
	       a.view_count::int,a.like_count::int,a.comment_count::int,a.published_at,a.created_at,a.updated_at,
	       u.id,u.username,u.nickname,u.avatar_url,u.bio,u.role,u.created_at,
	       c.id,c.name,c.slug,c.description,c.icon,c.parent_id,c.sort_order,c.created_at,
	       COALESCE((
	         SELECT jsonb_agg(jsonb_build_object(
	           'id',t.id,'name',t.name,'slug',t.slug,'color',t.color,'created_at',t.created_at
	         ) ORDER BY lower(t.name))
	         FROM article_tags at JOIN tags t ON t.id=at.tag_id WHERE at.article_id=a.id
	       ), '[]'::jsonb)
	FROM articles a
	JOIN users u ON u.id=a.author_id
	LEFT JOIN categories c ON c.id=a.category_id`

type ArticleRepository struct{ pool *pgxpool.Pool }

func NewArticleRepository(pool *pgxpool.Pool) *ArticleRepository { return &ArticleRepository{pool: pool} }

func (r *ArticleRepository) List(ctx context.Context, filter domain.ArticleFilter) (domain.Page[domain.Article], error) {
	conditions, arguments := articleConditions(filter)
	where := " WHERE " + strings.Join(conditions, " AND ")

	var total int
	if err := r.pool.QueryRow(ctx, "SELECT count(*)::int FROM articles a"+where, arguments...).Scan(&total); err != nil {
		return domain.Page[domain.Article]{}, fmt.Errorf("count articles: %w", err)
	}
	order := "a.is_top DESC, a.published_at DESC NULLS LAST, a.created_at DESC"
	if filter.Sort == "hottest" { order = "a.is_top DESC, a.like_count DESC, a.published_at DESC NULLS LAST" }
	if filter.Sort == "views" { order = "a.is_top DESC, a.view_count DESC, a.published_at DESC NULLS LAST" }
	arguments = append(arguments, filter.PageSize, (filter.Page-1)*filter.PageSize)
	query := articleSelect + where + fmt.Sprintf(" ORDER BY %s LIMIT $%d OFFSET $%d", order, len(arguments)-1, len(arguments))
	rows, err := r.pool.Query(ctx, query, arguments...)
	if err != nil { return domain.Page[domain.Article]{}, fmt.Errorf("list articles: %w", err) }
	defer rows.Close()
	items := make([]domain.Article, 0, filter.PageSize)
	for rows.Next() {
		item, err := scanArticle(rows)
		if err != nil { return domain.Page[domain.Article]{}, err }
		items = append(items, item)
	}
	if err := rows.Err(); err != nil { return domain.Page[domain.Article]{}, fmt.Errorf("iterate articles: %w", err) }
	return domain.Page[domain.Article]{Items: items, Page: filter.Page, PageSize: filter.PageSize, Total: total, TotalPages: pageCount(total, filter.PageSize)}, nil
}

func (r *ArticleRepository) Get(ctx context.Context, slug string, includePrivate bool) (domain.Article, error) {
	condition := "a.slug=$1 AND a.status='published'"
	if includePrivate { condition = "a.slug=$1" }
	return scanArticle(r.pool.QueryRow(ctx, articleSelect+" WHERE "+condition, slug))
}

func (r *ArticleRepository) Create(ctx context.Context, authorID int64, input domain.ArticleInput) (domain.Article, error) {
	tx, err := r.pool.Begin(ctx)
	if err != nil { return domain.Article{}, fmt.Errorf("begin article creation: %w", err) }
	defer func() { _ = tx.Rollback(ctx) }()
	var publishedAt *time.Time
	if input.Status == "published" { now := time.Now().UTC(); publishedAt = &now }
	var id int64
	err = tx.QueryRow(ctx, `
		INSERT INTO articles (slug,title,summary,content,cover_image,category_id,author_id,status,is_top,published_at)
		VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10) RETURNING id`,
		input.Slug,input.Title,input.Summary,input.Content,input.CoverImage,input.CategoryID,authorID,input.Status,input.IsTop,publishedAt).Scan(&id)
	if isUniqueViolation(err) { return domain.Article{}, domain.ErrConflict }
	if err != nil { return domain.Article{}, fmt.Errorf("insert article: %w", err) }
	if err := replaceArticleTags(ctx, tx, id, input.TagIDs); err != nil { return domain.Article{}, err }
	if err := tx.Commit(ctx); err != nil { return domain.Article{}, fmt.Errorf("commit article creation: %w", err) }
	return r.Get(ctx, input.Slug, true)
}

func (r *ArticleRepository) Update(ctx context.Context, currentSlug string, input domain.ArticleInput) (domain.Article, error) {
	tx, err := r.pool.Begin(ctx)
	if err != nil { return domain.Article{}, fmt.Errorf("begin article update: %w", err) }
	defer func() { _ = tx.Rollback(ctx) }()
	var id int64
	err = tx.QueryRow(ctx, `
		UPDATE articles SET slug=$2,title=$3,summary=$4,content=$5,cover_image=$6,category_id=$7,is_top=$8,updated_at=now()
		WHERE slug=$1 RETURNING id`, currentSlug,input.Slug,input.Title,input.Summary,input.Content,input.CoverImage,input.CategoryID,input.IsTop).Scan(&id)
	if errors.Is(err, pgx.ErrNoRows) { return domain.Article{}, domain.ErrNotFound }
	if isUniqueViolation(err) { return domain.Article{}, domain.ErrConflict }
	if err != nil { return domain.Article{}, fmt.Errorf("update article: %w", err) }
	if err := replaceArticleTags(ctx, tx, id, input.TagIDs); err != nil { return domain.Article{}, err }
	if err := tx.Commit(ctx); err != nil { return domain.Article{}, fmt.Errorf("commit article update: %w", err) }
	return r.Get(ctx, input.Slug, true)
}

func (r *ArticleRepository) SetStatus(ctx context.Context, slug, status string) (domain.Article, error) {
	command, err := r.pool.Exec(ctx, `
		UPDATE articles SET status=$2,
		  published_at=CASE WHEN $2='published' THEN COALESCE(published_at,now()) ELSE NULL END,
		  updated_at=now() WHERE slug=$1`, slug, status)
	if err != nil { return domain.Article{}, fmt.Errorf("set article status: %w", err) }
	if command.RowsAffected()==0 { return domain.Article{}, domain.ErrNotFound }
	return r.Get(ctx, slug, true)
}

func (r *ArticleRepository) SetTop(ctx context.Context, slug string, top bool) (domain.Article, error) {
	command, err := r.pool.Exec(ctx, `UPDATE articles SET is_top=$2,updated_at=now() WHERE slug=$1`,slug,top)
	if err != nil { return domain.Article{}, fmt.Errorf("set article top: %w", err) }
	if command.RowsAffected()==0 { return domain.Article{}, domain.ErrNotFound }
	return r.Get(ctx, slug, true)
}

func (r *ArticleRepository) Delete(ctx context.Context, slug string) error {
	command, err := r.pool.Exec(ctx, `DELETE FROM articles WHERE slug=$1`,slug)
	if err != nil { return fmt.Errorf("delete article: %w", err) }
	if command.RowsAffected()==0 { return domain.ErrNotFound }
	return nil
}

func (r *ArticleRepository) IncrementView(ctx context.Context, slug string) error {
	_, err := r.pool.Exec(ctx, `UPDATE articles SET view_count=view_count+1 WHERE slug=$1 AND status='published'`,slug)
	return err
}

func articleConditions(filter domain.ArticleFilter) ([]string, []any) {
	conditions := make([]string,0,5)
	arguments := make([]any,0,5)
	add := func(format string, value any) { arguments=append(arguments,value); conditions=append(conditions,fmt.Sprintf(format,len(arguments))) }
	if filter.IncludePrivate {
		if filter.Status!="" { add("a.status=$%d",filter.Status) } else { conditions=append(conditions,"TRUE") }
	} else { conditions=append(conditions,"a.status='published'") }
	if filter.Category!="" { add("EXISTS (SELECT 1 FROM categories c WHERE c.id=a.category_id AND c.slug=$%d)",filter.Category) }
	if filter.Tag!="" { add("EXISTS (SELECT 1 FROM article_tags at JOIN tags t ON t.id=at.tag_id WHERE at.article_id=a.id AND t.slug=$%d)",filter.Tag) }
	if filter.Keyword!="" { arguments=append(arguments,"%"+escapeLike(filter.Keyword)+"%"); conditions=append(conditions,fmt.Sprintf("(a.title ILIKE $%d ESCAPE E'\\\\' OR a.summary ILIKE $%d ESCAPE E'\\\\')",len(arguments),len(arguments))) }
	return conditions,arguments
}

func replaceArticleTags(ctx context.Context, tx pgx.Tx, articleID int64, tagIDs []int64) error {
	if _, err := tx.Exec(ctx, `DELETE FROM article_tags WHERE article_id=$1`,articleID); err != nil { return fmt.Errorf("clear article tags: %w",err) }
	if len(tagIDs)==0 { return nil }
	command, err := tx.Exec(ctx, `
		INSERT INTO article_tags(article_id,tag_id)
		SELECT $1,id FROM tags WHERE id=ANY($2::bigint[])`,articleID,tagIDs)
	if err != nil { return fmt.Errorf("insert article tags: %w",err) }
	if command.RowsAffected()!=int64(len(tagIDs)) { return domain.ErrBadRequest }
	return nil
}

func scanArticle(row rowScanner) (domain.Article,error) {
	var item domain.Article
	var categoryID *int64
	var categoryName,categorySlug *string
	var categoryDescription,categoryIcon *string
	var categoryParent *int64
	var categorySort *int
	var categoryCreated *time.Time
	var tagsJSON []byte
	err:=row.Scan(&item.ID,&item.Slug,&item.Title,&item.Summary,&item.Content,&item.CoverImage,&item.Status,&item.IsTop,
		&item.ViewCount,&item.LikeCount,&item.CommentCount,&item.PublishedAt,&item.CreatedAt,&item.UpdatedAt,
		&item.Author.ID,&item.Author.Username,&item.Author.Nickname,&item.Author.AvatarURL,&item.Author.Bio,&item.Author.Role,&item.Author.CreatedAt,
		&categoryID,&categoryName,&categorySlug,&categoryDescription,&categoryIcon,&categoryParent,&categorySort,&categoryCreated,&tagsJSON)
	if errors.Is(err,pgx.ErrNoRows) { return domain.Article{},domain.ErrNotFound }
	if err!=nil { return domain.Article{},fmt.Errorf("scan article: %w",err) }
	if categoryID!=nil { item.Category=&domain.Category{ID:*categoryID,Name:*categoryName,Slug:*categorySlug,Description:categoryDescription,Icon:categoryIcon,ParentID:categoryParent,SortOrder:*categorySort,CreatedAt:*categoryCreated} }
	var tags []struct { ID int64 `json:"id"`; Name string `json:"name"`; Slug string `json:"slug"`; Color *string `json:"color"`; CreatedAt time.Time `json:"created_at"` }
	if err:=json.Unmarshal(tagsJSON,&tags);err!=nil { return domain.Article{},fmt.Errorf("decode article tags: %w",err) }
	item.Tags=make([]domain.Tag,0,len(tags))
	for _,tag:=range tags { item.Tags=append(item.Tags,domain.Tag{ID:tag.ID,Name:tag.Name,Slug:tag.Slug,Color:tag.Color,CreatedAt:tag.CreatedAt}) }
	return item,nil
}

func pageCount(total,size int) int { if total==0{return 0}; return (total+size-1)/size }
func escapeLike(value string) string { value=strings.ReplaceAll(value,`\`,`\\`); value=strings.ReplaceAll(value,"%",`\%`); return strings.ReplaceAll(value,"_",`\_`) }
```

说明：SQL 字符串只拼接程序内的固定片段；筛选值始终通过 `$n` 参数传入。`ORDER BY` 只接受代码白名单，不接受用户原文。

执行：

```powershell
PS> Set-Location 'server'
PS> gofmt -w internal/repository/article.go
PS> go test ./internal/repository
PS> Set-Location '..'
```

## 07-10 创建项目 Repository

创建 `server/internal/repository/project.go`：

```go
package repository

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"strings"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
)

const projectSelect = `SELECT id,slug,name,summary,content,cover_image,technologies,
	repository_url,demo_url,featured,sort_order,status,started_at,completed_at,created_at,updated_at FROM projects`

type ProjectRepository struct{ pool *pgxpool.Pool }

func NewProjectRepository(pool *pgxpool.Pool) *ProjectRepository { return &ProjectRepository{pool: pool} }

func (r *ProjectRepository) List(ctx context.Context, filter domain.ProjectFilter) (domain.Page[domain.Project],error) {
	conditions:=make([]string,0,4)
	arguments:=make([]any,0,4)
	add:=func(format string,value any){arguments=append(arguments,value);conditions=append(conditions,fmt.Sprintf(format,len(arguments)))}
	if filter.IncludePrivate { if filter.Status!="" { add("status=$%d",filter.Status) } else { conditions=append(conditions,"TRUE") } } else { conditions=append(conditions,"status='active'") }
	if filter.Featured!=nil { add("featured=$%d",*filter.Featured) }
	if filter.Technology!="" { add("technologies @> to_jsonb(ARRAY[$%d]::text[])",filter.Technology) }
	where:=" WHERE "+strings.Join(conditions," AND ")
	var total int
	if err:=r.pool.QueryRow(ctx,"SELECT count(*)::int FROM projects"+where,arguments...).Scan(&total);err!=nil{return domain.Page[domain.Project]{},fmt.Errorf("count projects: %w",err)}
	arguments=append(arguments,filter.PageSize,(filter.Page-1)*filter.PageSize)
	rows,err:=r.pool.Query(ctx,projectSelect+where+fmt.Sprintf(" ORDER BY featured DESC,sort_order,updated_at DESC LIMIT $%d OFFSET $%d",len(arguments)-1,len(arguments)),arguments...)
	if err!=nil{return domain.Page[domain.Project]{},fmt.Errorf("list projects: %w",err)}
	defer rows.Close()
	items:=make([]domain.Project,0,filter.PageSize)
	for rows.Next(){item,err:=scanProject(rows);if err!=nil{return domain.Page[domain.Project]{},err};items=append(items,item)}
	if err:=rows.Err();err!=nil{return domain.Page[domain.Project]{},fmt.Errorf("iterate projects: %w",err)}
	return domain.Page[domain.Project]{Items:items,Page:filter.Page,PageSize:filter.PageSize,Total:total,TotalPages:pageCount(total,filter.PageSize)},nil
}

func (r *ProjectRepository) Get(ctx context.Context,slug string,includePrivate bool)(domain.Project,error){
	condition:="slug=$1 AND status='active'";if includePrivate{condition="slug=$1"}
	return scanProject(r.pool.QueryRow(ctx,projectSelect+" WHERE "+condition,slug))
}

func (r *ProjectRepository) Create(ctx context.Context,input domain.ProjectInput)(domain.Project,error){
	technologies,err:=json.Marshal(input.Technologies);if err!=nil{return domain.Project{},err}
	_,err=r.pool.Exec(ctx,`INSERT INTO projects(slug,name,summary,content,cover_image,technologies,repository_url,demo_url,featured,sort_order,status,started_at,completed_at)
		VALUES($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13)`,input.Slug,input.Name,input.Summary,input.Content,input.CoverImage,technologies,input.RepositoryURL,input.DemoURL,input.Featured,input.SortOrder,input.Status,input.StartedAt,input.CompletedAt)
	if isUniqueViolation(err){return domain.Project{},domain.ErrConflict};if err!=nil{return domain.Project{},fmt.Errorf("insert project: %w",err)}
	return r.Get(ctx,input.Slug,true)
}

func (r *ProjectRepository) Update(ctx context.Context,currentSlug string,input domain.ProjectInput)(domain.Project,error){
	technologies,err:=json.Marshal(input.Technologies);if err!=nil{return domain.Project{},err}
	command,err:=r.pool.Exec(ctx,`UPDATE projects SET slug=$2,name=$3,summary=$4,content=$5,cover_image=$6,technologies=$7,
		repository_url=$8,demo_url=$9,featured=$10,sort_order=$11,status=$12,started_at=$13,completed_at=$14,updated_at=now() WHERE slug=$1`,
		currentSlug,input.Slug,input.Name,input.Summary,input.Content,input.CoverImage,technologies,input.RepositoryURL,input.DemoURL,input.Featured,input.SortOrder,input.Status,input.StartedAt,input.CompletedAt)
	if isUniqueViolation(err){return domain.Project{},domain.ErrConflict};if err!=nil{return domain.Project{},fmt.Errorf("update project: %w",err)}
	if command.RowsAffected()==0{return domain.Project{},domain.ErrNotFound}
	return r.Get(ctx,input.Slug,true)
}

func (r *ProjectRepository) Delete(ctx context.Context,slug string)error{
	command,err:=r.pool.Exec(ctx,`DELETE FROM projects WHERE slug=$1`,slug);if err!=nil{return fmt.Errorf("delete project: %w",err)};if command.RowsAffected()==0{return domain.ErrNotFound};return nil
}

func scanProject(row rowScanner)(domain.Project,error){
	var item domain.Project;var technologies []byte
	err:=row.Scan(&item.ID,&item.Slug,&item.Name,&item.Summary,&item.Content,&item.CoverImage,&technologies,&item.RepositoryURL,&item.DemoURL,&item.Featured,&item.SortOrder,&item.Status,&item.StartedAt,&item.CompletedAt,&item.CreatedAt,&item.UpdatedAt)
	if errors.Is(err,pgx.ErrNoRows){return domain.Project{},domain.ErrNotFound};if err!=nil{return domain.Project{},fmt.Errorf("scan project: %w",err)}
	if err:=json.Unmarshal(technologies,&item.Technologies);err!=nil{return domain.Project{},fmt.Errorf("decode technologies: %w",err)}
	return item,nil
}
```

## 07-11 创建站点设置与统计 Repository

创建 `server/internal/repository/site.go`：

```go
package repository

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
)

type SiteRepository struct{ pool *pgxpool.Pool }
func NewSiteRepository(pool *pgxpool.Pool)*SiteRepository{return &SiteRepository{pool:pool}}

func(r *SiteRepository)GetSettings(ctx context.Context)(domain.SiteSettings,error){
	var raw []byte
	err:=r.pool.QueryRow(ctx,`SELECT value FROM site_settings WHERE key='site' AND is_public=true`).Scan(&raw)
	if err!=nil{return domain.SiteSettings{},fmt.Errorf("read site settings: %w",err)}
	var settings domain.SiteSettings
	if err:=json.Unmarshal(raw,&settings);err!=nil{return domain.SiteSettings{},fmt.Errorf("decode site settings: %w",err)}
	return settings,nil
}

func(r *SiteRepository)UpdateSettings(ctx context.Context,settings domain.SiteSettings,userID int64)error{
	raw,err:=json.Marshal(settings);if err!=nil{return err}
	_,err=r.pool.Exec(ctx,`INSERT INTO site_settings(key,value,is_public,updated_by,updated_at) VALUES('site',$1,true,$2,now())
		ON CONFLICT(key) DO UPDATE SET value=EXCLUDED.value,is_public=true,updated_by=EXCLUDED.updated_by,updated_at=now()`,raw,userID)
	if err!=nil{return fmt.Errorf("update site settings: %w",err)};return nil
}

func(r *SiteRepository)PublicStats(ctx context.Context)(articles,projects,categories,tags int,err error){
	err=r.pool.QueryRow(ctx,`SELECT
		(SELECT count(*)::int FROM articles WHERE status='published'),
		(SELECT count(*)::int FROM projects WHERE status='active'),
		(SELECT count(*)::int FROM categories),
		(SELECT count(*)::int FROM tags)`).Scan(&articles,&projects,&categories,&tags)
	return
}

func(r *SiteRepository)Dashboard(ctx context.Context)(domain.Dashboard,error){
	var result domain.Dashboard
	err:=r.pool.QueryRow(ctx,`SELECT
		(SELECT count(*)::int FROM articles),
		(SELECT count(*)::int FROM articles WHERE status='draft'),
		(SELECT count(*)::int FROM projects),
		(SELECT count(*)::int FROM comments WHERE status='pending'),
		(SELECT COALESCE(sum(view_count),0)::int FROM articles),
		(SELECT COALESCE(sum(like_count),0)::int FROM articles)`).Scan(&result.TotalArticles,&result.DraftArticles,&result.TotalProjects,&result.PendingComments,&result.TotalViews,&result.TotalLikes)
	if err!=nil{return domain.Dashboard{},fmt.Errorf("read dashboard: %w",err)};return result,nil
}
```

迁移中的初始 `site` JSON 在本章执行前需要与模型一致。创建新迁移 `server/migrations/000005_normalize_site_settings.up.sql`：

```sql
UPDATE site_settings
SET value = '{"site_title":"Rodolfo Iolo","site_description":"Articles, projects, and engineering notes.","site_keywords":"computer science,go,web development","about_content":"# About\n\nComputer science student learning in public.","footer_text":"Built with care and documented end to end.","icp_number":null,"social_links":[]}'::jsonb,
    updated_at = now()
WHERE key = 'site';
```

创建 `server/migrations/000005_normalize_site_settings.down.sql`：

```sql
UPDATE site_settings
SET value = '{"title":"Rodolfo Iolo","description":"Articles, projects, and engineering notes."}'::jsonb,
    updated_at = now()
WHERE key = 'site';
```

执行迁移：

```powershell
PS> docker compose -f compose.dev.yml run --rm migrate
PS> docker compose -f compose.dev.yml exec postgres psql -U personal_site -d personal_site -c 'SELECT version, dirty FROM schema_migrations;'
```

预期版本 5、dirty=false。

## 07-12 创建互动 Repository

创建 `server/internal/repository/interaction.go`：

```go
package repository

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
)

type InteractionRepository struct{pool *pgxpool.Pool}
func NewInteractionRepository(pool *pgxpool.Pool)*InteractionRepository{return &InteractionRepository{pool:pool}}

func(r *InteractionRepository)ListComments(ctx context.Context,articleSlug string,page,pageSize int,includePending bool,status string)(domain.Page[domain.Comment],error){
	conditions:="a.slug=$1 AND c.status='approved'";arguments:=[]any{articleSlug}
	if includePending{conditions="a.slug=$1";if status!=""{arguments=append(arguments,status);conditions+=" AND c.status=$2"}}
	var total int
	if err:=r.pool.QueryRow(ctx,`SELECT count(*)::int FROM comments c JOIN articles a ON a.id=c.article_id WHERE `+conditions,arguments...).Scan(&total);err!=nil{return domain.Page[domain.Comment]{},fmt.Errorf("count comments: %w",err)}
	arguments=append(arguments,pageSize,(page-1)*pageSize)
	rows,err:=r.pool.Query(ctx,`SELECT c.id,c.article_id,c.parent_id,c.author_name,c.content,c.status,c.created_at
		FROM comments c JOIN articles a ON a.id=c.article_id WHERE `+conditions+fmt.Sprintf(" ORDER BY c.created_at LIMIT $%d OFFSET $%d",len(arguments)-1,len(arguments)),arguments...)
	if err!=nil{return domain.Page[domain.Comment]{},fmt.Errorf("list comments: %w",err)};defer rows.Close()
	items:=make([]domain.Comment,0,pageSize);for rows.Next(){var item domain.Comment;if err:=rows.Scan(&item.ID,&item.ArticleID,&item.ParentID,&item.Nickname,&item.Content,&item.Status,&item.CreatedAt);err!=nil{return domain.Page[domain.Comment]{},err};items=append(items,item)}
	return domain.Page[domain.Comment]{Items:items,Page:page,PageSize:pageSize,Total:total,TotalPages:pageCount(total,pageSize)},rows.Err()
}

func(r *InteractionRepository)ListAdminComments(ctx context.Context,page,pageSize int,status string)(domain.Page[domain.Comment],error){
	condition:="TRUE";arguments:=[]any{};if status!=""{condition="status=$1";arguments=append(arguments,status)}
	var total int;if err:=r.pool.QueryRow(ctx,"SELECT count(*)::int FROM comments WHERE "+condition,arguments...).Scan(&total);err!=nil{return domain.Page[domain.Comment]{},err}
	arguments=append(arguments,pageSize,(page-1)*pageSize)
	rows,err:=r.pool.Query(ctx,`SELECT id,article_id,parent_id,author_name,content,status,created_at FROM comments WHERE `+condition+fmt.Sprintf(" ORDER BY created_at DESC LIMIT $%d OFFSET $%d",len(arguments)-1,len(arguments)),arguments...)
	if err!=nil{return domain.Page[domain.Comment]{},err};defer rows.Close();items:=make([]domain.Comment,0,pageSize)
	for rows.Next(){var item domain.Comment;if err:=rows.Scan(&item.ID,&item.ArticleID,&item.ParentID,&item.Nickname,&item.Content,&item.Status,&item.CreatedAt);err!=nil{return domain.Page[domain.Comment]{},err};items=append(items,item)}
	return domain.Page[domain.Comment]{Items:items,Page:page,PageSize:pageSize,Total:total,TotalPages:pageCount(total,pageSize)},rows.Err()
}

func(r *InteractionRepository)CreateComment(ctx context.Context,input domain.CommentInput)(domain.Comment,error){
	tx,err:=r.pool.Begin(ctx);if err!=nil{return domain.Comment{},err};defer func(){_ = tx.Rollback(ctx)}()
	var articleID int64;if err:=tx.QueryRow(ctx,`SELECT id FROM articles WHERE slug=$1 AND status='published' FOR SHARE`,input.ArticleSlug).Scan(&articleID);errors.Is(err,pgx.ErrNoRows){return domain.Comment{},domain.ErrNotFound}else if err!=nil{return domain.Comment{},err}
	if input.ParentID!=nil{var parentArticle int64;var grandParent *int64;err:=tx.QueryRow(ctx,`SELECT article_id,parent_id FROM comments WHERE id=$1 AND status='approved' FOR SHARE`,*input.ParentID).Scan(&parentArticle,&grandParent);if errors.Is(err,pgx.ErrNoRows)||parentArticle!=articleID||grandParent!=nil{return domain.Comment{},domain.ErrBadRequest};if err!=nil{return domain.Comment{},err}}
	var item domain.Comment
	err=tx.QueryRow(ctx,`INSERT INTO comments(article_id,parent_id,author_name,author_email_hash,content,visitor_id,ip_hash,user_agent)
		VALUES($1,$2,$3,$4,$5,$6,$7,$8) RETURNING id,article_id,parent_id,author_name,content,status,created_at`,articleID,input.ParentID,input.Nickname,input.EmailHash,input.Content,input.VisitorID,input.IPHash,input.UserAgent).Scan(&item.ID,&item.ArticleID,&item.ParentID,&item.Nickname,&item.Content,&item.Status,&item.CreatedAt)
	if err!=nil{return domain.Comment{},fmt.Errorf("insert comment: %w",err)};if err:=tx.Commit(ctx);err!=nil{return domain.Comment{},err};return item,nil
}

func(r *InteractionRepository)ReviewComment(ctx context.Context,id,userID int64,status string)(domain.Comment,error){
	tx,err:=r.pool.Begin(ctx);if err!=nil{return domain.Comment{},err};defer func(){_ = tx.Rollback(ctx)}()
	var item domain.Comment
	if err:=tx.QueryRow(ctx,`SELECT id,article_id,parent_id,author_name,content,status,created_at FROM comments WHERE id=$1 FOR UPDATE`,id).Scan(&item.ID,&item.ArticleID,&item.ParentID,&item.Nickname,&item.Content,&item.Status,&item.CreatedAt);errors.Is(err,pgx.ErrNoRows){return domain.Comment{},domain.ErrNotFound}else if err!=nil{return domain.Comment{},err}
	previous:=item.Status
	if _,err:=tx.Exec(ctx,`UPDATE comments SET status=$2,reviewed_by=$3,reviewed_at=now() WHERE id=$1`,id,status,userID);err!=nil{return domain.Comment{},err};item.Status=status
	if previous!="approved"&&status=="approved"{_,err=tx.Exec(ctx,`UPDATE articles SET comment_count=comment_count+1 WHERE id=$1`,item.ArticleID)}else if previous=="approved"&&status!="approved"{_,err=tx.Exec(ctx,`UPDATE articles SET comment_count=GREATEST(comment_count-1,0) WHERE id=$1`,item.ArticleID)}
	if err!=nil{return domain.Comment{},err};if err:=tx.Commit(ctx);err!=nil{return domain.Comment{},err};return item,nil
}

func(r *InteractionRepository)DeleteComment(ctx context.Context,id int64)error{
	tx,err:=r.pool.Begin(ctx);if err!=nil{return err};defer func(){_ = tx.Rollback(ctx)}();var articleID int64;var status string
	err=tx.QueryRow(ctx,`DELETE FROM comments WHERE id=$1 RETURNING article_id,status`,id).Scan(&articleID,&status);if errors.Is(err,pgx.ErrNoRows){return domain.ErrNotFound};if err!=nil{return err};if status=="approved"{if _,err=tx.Exec(ctx,`UPDATE articles SET comment_count=GREATEST(comment_count-1,0) WHERE id=$1`,articleID);err!=nil{return err}};return tx.Commit(ctx)
}

func(r *InteractionRepository)SetLike(ctx context.Context,articleSlug,visitorID,ipHash string,liked bool)(bool,int,error){
	tx,err:=r.pool.Begin(ctx);if err!=nil{return false,0,err};defer func(){_ = tx.Rollback(ctx)}();var articleID int64
	if err:=tx.QueryRow(ctx,`SELECT id FROM articles WHERE slug=$1 AND status='published' FOR UPDATE`,articleSlug).Scan(&articleID);errors.Is(err,pgx.ErrNoRows){return false,0,domain.ErrNotFound}else if err!=nil{return false,0,err}
	if liked{command,err:=tx.Exec(ctx,`INSERT INTO likes(article_id,visitor_id,ip_hash) VALUES($1,$2,$3) ON CONFLICT(article_id,visitor_id) DO NOTHING`,articleID,visitorID,ipHash);if err!=nil{return false,0,err};if command.RowsAffected()>0{_,err=tx.Exec(ctx,`UPDATE articles SET like_count=like_count+1 WHERE id=$1`,articleID);if err!=nil{return false,0,err}}}else{command,err:=tx.Exec(ctx,`DELETE FROM likes WHERE article_id=$1 AND visitor_id=$2`,articleID,visitorID);if err!=nil{return false,0,err};if command.RowsAffected()>0{_,err=tx.Exec(ctx,`UPDATE articles SET like_count=GREATEST(like_count-1,0) WHERE id=$1`,articleID);if err!=nil{return false,0,err}}}
	var finalLiked bool;var count int
	if err:=tx.QueryRow(ctx,`SELECT EXISTS(SELECT 1 FROM likes WHERE article_id=$1 AND visitor_id=$2),like_count::int FROM articles WHERE id=$1`,articleID,visitorID).Scan(&finalLiked,&count);err!=nil{return false,0,err};if err:=tx.Commit(ctx);err!=nil{return false,0,err};return finalLiked,count,nil
}

func(r *InteractionRepository)LikeStatus(ctx context.Context,articleSlug,visitorID string)(bool,int,error){
	var liked bool;var count int;err:=r.pool.QueryRow(ctx,`SELECT EXISTS(SELECT 1 FROM likes l WHERE l.article_id=a.id AND l.visitor_id=$2),a.like_count::int FROM articles a WHERE a.slug=$1 AND a.status='published'`,articleSlug,visitorID).Scan(&liked,&count);if errors.Is(err,pgx.ErrNoRows){return false,0,domain.ErrNotFound};return liked,count,err
}
```

## 07-13 创建签名访客身份

创建 `server/internal/security/visitor.go`：

```go
package security

import (
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/base64"
	"errors"
	"strings"
)

type VisitorSigner struct{ secret []byte }

func NewVisitorSigner(secret []byte) (VisitorSigner, error) {
	if len(secret) < 32 { return VisitorSigner{}, errors.New("visitor secret must contain at least 32 bytes") }
	return VisitorSigner{secret: append([]byte(nil), secret...)}, nil
}
func (s VisitorSigner) New() (string, string, error) {
	buffer := make([]byte, 16)
	if _, err := rand.Read(buffer); err != nil { return "", "", err }
	id := base64.RawURLEncoding.EncodeToString(buffer)
	return id, id + "." + s.signature(id), nil
}
func (s VisitorSigner) Verify(value string) (string, bool) {
	parts := strings.Split(value, ".")
	if len(parts) != 2 || parts[0] == "" { return "", false }
	expected := s.signature(parts[0])
	if !hmac.Equal([]byte(expected), []byte(parts[1])) { return "", false }
	return parts[0], true
}
func (s VisitorSigner) signature(id string) string {
	digest := hmac.New(sha256.New, s.secret)
	_, _ = digest.Write([]byte(id))
	return base64.RawURLEncoding.EncodeToString(digest.Sum(nil))
}
```

创建 `server/internal/middleware/visitor.go`：

```go
package middleware

import (
	"net/http"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/requestctx"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/security"
)

func VisitorIdentity(name string, secure bool, signer security.VisitorSigner) echo.MiddlewareFunc {
	return func(next echo.HandlerFunc) echo.HandlerFunc {
		return func(c echo.Context) error {
			var id string
			if cookie, err := c.Cookie(name); err == nil { id, _ = signer.Verify(cookie.Value) }
			if id == "" {
				var signed string
				var err error
				id, signed, err = signer.New()
				if err != nil { return err }
				c.SetCookie(&http.Cookie{Name: name, Value: signed, Path: "/api/v1", MaxAge: 31536000, Expires: time.Now().UTC().Add(365*24*time.Hour), HttpOnly: true, Secure: secure, SameSite: http.SameSiteLaxMode})
			}
			ctx := c.Request().Context()
			metadata := requestctx.MetadataFrom(ctx)
			metadata.VisitorID = id
			c.SetRequest(c.Request().WithContext(requestctx.WithMetadata(ctx, metadata)))
			return next(c)
		}
	}
}
```

创建 `server/internal/security/visitor_test.go`：

```go
package security_test

import (
	"testing"

	"github.com/stretchr/testify/require"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/security"
)

func TestVisitorSignatureRejectsTampering(t *testing.T) {
	signer, err := security.NewVisitorSigner([]byte("0123456789abcdef0123456789abcdef"))
	require.NoError(t, err)
	id, signed, err := signer.New()
	require.NoError(t, err)
	verified, ok := signer.Verify(signed)
	require.True(t, ok)
	require.Equal(t, id, verified)
	_, ok = signer.Verify(signed + "x")
	require.False(t, ok)
}
```

## 07-14 创建媒体 Repository 和对象存储接口

创建 `server/internal/repository/media.go`：

```go
package repository

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
)

type MediaRepository struct{ pool *pgxpool.Pool }
func NewMediaRepository(pool *pgxpool.Pool) *MediaRepository { return &MediaRepository{pool: pool} }
func (r *MediaRepository) Create(ctx context.Context, item domain.MediaAsset, userID int64) (domain.MediaAsset, error) {
	err := r.pool.QueryRow(ctx, `INSERT INTO media_assets(object_key,original_name,mime_type,byte_size,width,height,sha256,uploaded_by)
		VALUES($1,$2,$3,$4,$5,$6,$7,$8) RETURNING id,created_at`, item.ObjectKey, item.OriginalName, item.MIMEType, item.ByteSize, item.Width, item.Height, item.SHA256, userID).Scan(&item.ID, &item.CreatedAt)
	if err != nil { return domain.MediaAsset{}, fmt.Errorf("insert media metadata: %w", err) }
	return item, nil
}
```

创建 `server/internal/storage/store.go`：

```go
package storage

import "context"

type ObjectStore interface {
	Put(context.Context, string, string, []byte) error
	Delete(context.Context, string) error
	PublicURL(string) string
}
```

创建 `server/internal/storage/local.go`：

```go
package storage

import (
	"context"
	"errors"
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

type Local struct{ root, publicBase string }
func NewLocal(root, publicBase string) (*Local, error) {
	absolute, err := filepath.Abs(root)
	if err != nil { return nil, err }
	if err := os.MkdirAll(absolute, 0o750); err != nil { return nil, err }
	return &Local{root: absolute, publicBase: strings.TrimRight(publicBase, "/")}, nil
}
func (s *Local) Put(_ context.Context, key, _ string, data []byte) error {
	path, err := s.safePath(key); if err != nil { return err }
	if err := os.MkdirAll(filepath.Dir(path), 0o750); err != nil { return err }
	if err := os.WriteFile(path, data, 0o640); err != nil { return fmt.Errorf("write local object: %w", err) }
	return nil
}
func (s *Local) Delete(_ context.Context, key string) error {
	path, err := s.safePath(key); if err != nil { return err }
	if err := os.Remove(path); err != nil && !errors.Is(err, os.ErrNotExist) { return err }
	return nil
}
func (s *Local) PublicURL(key string) string { return s.publicBase + "/" + strings.TrimLeft(key, "/") }
func (s *Local) safePath(key string) (string, error) {
	clean := filepath.Clean(filepath.FromSlash(key))
	if clean == "." || filepath.IsAbs(clean) || strings.HasPrefix(clean, "..") { return "", errors.New("invalid object key") }
	path := filepath.Join(s.root, clean)
	if !strings.HasPrefix(path, s.root+string(filepath.Separator)) { return "", errors.New("object path escapes root") }
	return path, nil
}
```

本地实现阻止绝对路径和 `..` 穿越。`uploads/` 已被忽略；生产卷会提供 OSS 实现并保持同一接口。

## 07-15 生成 Strict 适配器，消除机械重复

创建目录：

```powershell
PS> New-Item -ItemType Directory -Force -Path 'server\internal\handler\cmd\adaptergen' | Out-Null
```

创建 `server/internal/handler/response.go`：

```go
package handler

import (
	"encoding/json"
	"net/http"
)

type OperationResponse struct {
	Status      int
	ContentType string
	Body        any
	Headers     map[string]string
}

func JSON(status int, data any) *OperationResponse {
	return &OperationResponse{Status: status, ContentType: "application/json", Body: data}
}

func Success(status int, data any) *OperationResponse {
	return JSON(status, map[string]any{"code": 0, "message": "success", "data": data})
}

func Failure(status, code int, message string) *OperationResponse {
	return JSON(status, map[string]any{"code": code, "message": message, "data": nil})
}

func (r *OperationResponse) visit(writer http.ResponseWriter) error {
	contentType := r.ContentType
	if contentType == "" { contentType = "application/json" }
	writer.Header().Set("Content-Type", contentType)
	for name, value := range r.Headers { writer.Header().Set(name, value) }
	status := r.Status
	if status == 0 { status = http.StatusOK }
	writer.WriteHeader(status)
	if r.Body == nil { return nil }
	if bytes, ok := r.Body.([]byte); ok { _, err := writer.Write(bytes); return err }
	return json.NewEncoder(writer).Encode(r.Body)
}
```

创建 `server/internal/handler/generate.go`：

```go
package handler

//go:generate go run ./cmd/adaptergen -input ../../generated/oapi/api.gen.go -output adapter.gen.go
```

创建 `server/internal/handler/cmd/adaptergen/main.go`：

```go
package main

import (
	"bytes"
	"flag"
	"fmt"
	"go/ast"
	"go/format"
	"go/parser"
	"go/token"
	"os"
	"sort"
	"strings"
)

type method struct{ name, request, response, visitor string }

func main() {
	input := flag.String("input", "", "generated oapi Go file")
	output := flag.String("output", "", "generated handler adapter")
	flag.Parse()
	if *input == "" || *output == "" { fail("input and output are required") }
	fileSet := token.NewFileSet()
	file, err := parser.ParseFile(fileSet, *input, nil, 0)
	if err != nil { fail(err.Error()) }

	visitors := map[string]string{}
	var strict *ast.InterfaceType
	for _, declaration := range file.Decls {
		general, ok := declaration.(*ast.GenDecl); if !ok { continue }
		for _, specification := range general.Specs {
			typeSpec, ok := specification.(*ast.TypeSpec); if !ok { continue }
			if typeSpec.Name.Name == "StrictServerInterface" { strict, _ = typeSpec.Type.(*ast.InterfaceType) }
			if strings.HasSuffix(typeSpec.Name.Name, "ResponseObject") {
				if iface, ok := typeSpec.Type.(*ast.InterfaceType); ok && len(iface.Methods.List) == 1 && len(iface.Methods.List[0].Names) == 1 {
					visitors[typeSpec.Name.Name] = iface.Methods.List[0].Names[0].Name
				}
			}
		}
	}
	if strict == nil { fail("StrictServerInterface not found") }
	skipped := map[string]bool{"Login":true,"Logout":true,"GetCurrentUser":true,"RefreshAccessToken":true}
	methods := make([]method, 0)
	for _, field := range strict.Methods.List {
		if len(field.Names) != 1 || skipped[field.Names[0].Name] { continue }
		function, ok := field.Type.(*ast.FuncType); if !ok || len(function.Params.List) != 2 || len(function.Results.List) != 2 { fail("unexpected strict method shape") }
		request := expression(fileSet, function.Params.List[1].Type)
		response := expression(fileSet, function.Results.List[0].Type)
		visitor, ok := visitors[response]; if !ok { fail("response visitor not found for " + response) }
		methods = append(methods, method{name:field.Names[0].Name,request:request,response:response,visitor:visitor})
	}
	sort.Slice(methods, func(i,j int)bool{return methods[i].name<methods[j].name})
	var source bytes.Buffer
	source.WriteString("// Code generated by adaptergen; DO NOT EDIT.\npackage handler\n\nimport (\n\t\"context\"\n\t\"net/http\"\n\n\t\"github.com/<GITHUB_USER>/<REPOSITORY>/server/generated/oapi\"\n)\n\n")
	for _, item := range methods {
		fmt.Fprintf(&source,"func (h *Handler) %s(ctx context.Context, request oapi.%s) (oapi.%s, error) { return h.dispatch(ctx, %q, request) }\n",item.name,item.request,item.response,item.name)
		fmt.Fprintf(&source,"func (r *OperationResponse) %s(w http.ResponseWriter) error { return r.visit(w) }\n",item.visitor)
	}
	formatted, err := format.Source(source.Bytes()); if err != nil { fail(err.Error()+"\n"+source.String()) }
	if err := os.WriteFile(*output, formatted, 0o644); err != nil { fail(err.Error()) }
}

func expression(fileSet *token.FileSet, expression ast.Expr) string {
	var output bytes.Buffer
	if err := format.Node(&output, fileSet, expression); err != nil { fail(err.Error()) }
	return output.String()
}
func fail(message string) { fmt.Fprintln(os.Stderr, message); os.Exit(1) }
```

把生成器模板中的 module 路径替换为真实值，然后执行：

```powershell
PS> Set-Location 'server\internal\handler'
PS> go generate
PS> Set-Location '..\..\..'
PS> Get-Content -LiteralPath 'server\internal\handler\adapter.gen.go' -TotalCount 20
PS> git diff --check
```

`adapter.gen.go` 是命令生成文件，禁止手工编辑。生成器跳过四个已有认证方法，其余 Strict 方法全部转发到 `dispatch`；若 OpenAPI 新增 operation，重新生成就会出现新的编译缺口，迫使你实现它。

## 07-16 创建内容 Service

创建 `server/internal/service/content.go`：

```go
package service

import (
	"context"
	"strings"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/repository"
)

type ContentService struct {
	categories *repository.CategoryRepository
	tags       *repository.TagRepository
	articles   *repository.ArticleRepository
	projects   *repository.ProjectRepository
	site       *repository.SiteRepository
}

func NewContentService(categories *repository.CategoryRepository, tags *repository.TagRepository, articles *repository.ArticleRepository, projects *repository.ProjectRepository, site *repository.SiteRepository) *ContentService {
	return &ContentService{categories: categories, tags: tags, articles: articles, projects: projects, site: site}
}

func (s *ContentService) ListCategories(ctx context.Context) ([]domain.Category,error){return s.categories.List(ctx)}
func (s *ContentService) GetCategory(ctx context.Context,slug string)(domain.Category,error){return s.categories.GetBySlug(ctx,slug)}
func (s *ContentService) SaveCategory(ctx context.Context,currentSlug string,input domain.CategoryInput)(domain.Category,error){
	name,err:=requiredText(input.Name,1,100);if err!=nil{return domain.Category{},err};input.Name=name
	input.Slug,err=normalizedSlug(input.Slug,input.Name);if err!=nil{return domain.Category{},err}
	if input.ParentID!=nil{depth,err:=s.categories.ParentDepth(ctx,*input.ParentID);if err!=nil||depth>1{return domain.Category{},domain.ErrBadRequest}}
	if currentSlug==""{return s.categories.Create(ctx,input)};existing,err:=s.categories.GetBySlug(ctx,currentSlug);if err!=nil{return domain.Category{},err};if input.ParentID!=nil&&*input.ParentID==existing.ID{return domain.Category{},domain.ErrBadRequest};return s.categories.Update(ctx,currentSlug,input)
}
func(s *ContentService)DeleteCategory(ctx context.Context,slug string)error{return s.categories.Delete(ctx,slug)}

func(s *ContentService)ListTags(ctx context.Context)([]domain.Tag,error){return s.tags.List(ctx)}
func(s *ContentService)GetTag(ctx context.Context,slug string)(domain.Tag,error){return s.tags.GetBySlug(ctx,slug)}
func(s *ContentService)SaveTag(ctx context.Context,currentSlug string,input domain.TagInput)(domain.Tag,error){
	name,err:=requiredText(input.Name,1,100);if err!=nil{return domain.Tag{},err};input.Name=name;input.Slug,err=normalizedSlug(input.Slug,input.Name);if err!=nil{return domain.Tag{},err};if err:=validColor(input.Color);err!=nil{return domain.Tag{},err};if currentSlug==""{return s.tags.Create(ctx,input)};return s.tags.Update(ctx,currentSlug,input)
}
func(s *ContentService)DeleteTag(ctx context.Context,slug string)error{return s.tags.Delete(ctx,slug)}

func(s *ContentService)ListArticles(ctx context.Context,filter domain.ArticleFilter)(domain.Page[domain.Article],error){
	page,size,err:=normalizedPage(&filter.Page,&filter.PageSize);if err!=nil{return domain.Page[domain.Article]{},err};filter.Page,filter.PageSize=page,size;filter.Keyword=strings.TrimSpace(filter.Keyword);return s.articles.List(ctx,filter)
}
func(s *ContentService)GetArticle(ctx context.Context,slug string,private bool)(domain.Article,error){return s.articles.Get(ctx,slug,private)}
func(s *ContentService)CreateArticle(ctx context.Context,userID int64,input domain.ArticleInput)(domain.Article,error){
	var err error;input.Title,err=requiredText(input.Title,1,300);if err!=nil{return domain.Article{},err};input.Content,err=requiredText(input.Content,1,200000);if err!=nil{return domain.Article{},err};input.Slug,err=normalizedSlug(input.Slug,input.Title);if err!=nil{return domain.Article{},err};if strings.TrimSpace(input.Summary)==""{input.Summary=excerpt(input.Content,200)};input.Summary,err=requiredText(input.Summary,1,500);if err!=nil{return domain.Article{},err};if input.Status==""{input.Status="draft"};if err:=statusAllowed(input.Status,"draft","published");err!=nil{return domain.Article{},err};return s.articles.Create(ctx,userID,input)
}
func(s *ContentService)UpdateArticle(ctx context.Context,slug string,patch domain.ArticleInput)(domain.Article,error){
	current,err:=s.articles.Get(ctx,slug,true);if err!=nil{return domain.Article{},err}
	if patch.Title==""{patch.Title=current.Title};if patch.Content==""{patch.Content=current.Content};if patch.Summary==""{patch.Summary=current.Summary};if patch.Slug==""{patch.Slug=current.Slug};if patch.CoverImage==nil{patch.CoverImage=current.CoverImage};if patch.CategoryID==nil&&current.Category!=nil{id:=current.Category.ID;patch.CategoryID=&id};if patch.TagIDs==nil{patch.TagIDs=make([]int64,0,len(current.Tags));for _,tag:=range current.Tags{patch.TagIDs=append(patch.TagIDs,tag.ID)}};patch.Status=current.Status
	patch.Title,err=requiredText(patch.Title,1,300);if err!=nil{return domain.Article{},err};patch.Content,err=requiredText(patch.Content,1,200000);if err!=nil{return domain.Article{},err};patch.Summary,err=requiredText(patch.Summary,1,500);if err!=nil{return domain.Article{},err};patch.Slug,err=normalizedSlug(patch.Slug,patch.Title);if err!=nil{return domain.Article{},err};return s.articles.Update(ctx,slug,patch)
}
func(s *ContentService)PublishArticle(ctx context.Context,slug string)(domain.Article,error){current,err:=s.articles.Get(ctx,slug,true);if err!=nil{return domain.Article{},err};if current.Status=="archived"{return domain.Article{},domain.ErrConflict};return s.articles.SetStatus(ctx,slug,"published")}
func(s *ContentService)UnpublishArticle(ctx context.Context,slug string)(domain.Article,error){current,err:=s.articles.Get(ctx,slug,true);if err!=nil{return domain.Article{},err};if current.Status!="published"{return domain.Article{},domain.ErrConflict};return s.articles.SetStatus(ctx,slug,"draft")}
func(s *ContentService)SetArticleTop(ctx context.Context,slug string,top bool)(domain.Article,error){return s.articles.SetTop(ctx,slug,top)}
func(s *ContentService)DeleteArticle(ctx context.Context,slug string)error{return s.articles.Delete(ctx,slug)}

func(s *ContentService)ListProjects(ctx context.Context,filter domain.ProjectFilter)(domain.Page[domain.Project],error){page,size,err:=normalizedPage(&filter.Page,&filter.PageSize);if err!=nil{return domain.Page[domain.Project]{},err};filter.Page,filter.PageSize=page,size;return s.projects.List(ctx,filter)}
func(s *ContentService)GetProject(ctx context.Context,slug string,private bool)(domain.Project,error){return s.projects.Get(ctx,slug,private)}
func(s *ContentService)SaveProject(ctx context.Context,currentSlug string,input domain.ProjectInput)(domain.Project,error){
	if currentSlug!=""{current,err:=s.projects.Get(ctx,currentSlug,true);if err!=nil{return domain.Project{},err};if input.Name==""{input.Name=current.Name};if input.Slug==""{input.Slug=current.Slug};if input.Summary==""{input.Summary=current.Summary};if input.Content==""{input.Content=current.Content};if input.CoverImage==nil{input.CoverImage=current.CoverImage};if input.Technologies==nil{input.Technologies=current.Technologies};if input.RepositoryURL==nil{input.RepositoryURL=current.RepositoryURL};if input.DemoURL==nil{input.DemoURL=current.DemoURL};if input.Status==""{input.Status=current.Status};if input.StartedAt==nil{input.StartedAt=current.StartedAt};if input.CompletedAt==nil{input.CompletedAt=current.CompletedAt}}
	var err error;input.Name,err=requiredText(input.Name,1,200);if err!=nil{return domain.Project{},err};input.Slug,err=normalizedSlug(input.Slug,input.Name);if err!=nil{return domain.Project{},err};input.Summary,err=requiredText(input.Summary,1,500);if err!=nil{return domain.Project{},err};input.Content,err=requiredText(input.Content,1,200000);if err!=nil{return domain.Project{},err};input.Technologies,err=normalizedTechnologies(input.Technologies);if err!=nil{return domain.Project{},err};input.RepositoryURL,err=optionalHTTPURL(input.RepositoryURL);if err!=nil{return domain.Project{},err};input.DemoURL,err=optionalHTTPURL(input.DemoURL);if err!=nil{return domain.Project{},err};if input.Status==""{input.Status="active"};if err:=statusAllowed(input.Status,"active","archived");err!=nil{return domain.Project{},err};if input.StartedAt!=nil&&input.CompletedAt!=nil&&input.CompletedAt.Before(*input.StartedAt){return domain.Project{},domain.ErrBadRequest};if currentSlug==""{return s.projects.Create(ctx,input)};return s.projects.Update(ctx,currentSlug,input)
}
func(s *ContentService)DeleteProject(ctx context.Context,slug string)error{return s.projects.Delete(ctx,slug)}

func(s *ContentService)GetSettings(ctx context.Context)(domain.SiteSettings,error){return s.site.GetSettings(ctx)}
func(s *ContentService)UpdateSettings(ctx context.Context,userID int64,settings domain.SiteSettings)(domain.SiteSettings,error){
	var err error;settings.SiteTitle,err=requiredText(settings.SiteTitle,1,100);if err!=nil{return domain.SiteSettings{},err};settings.SiteDescription,err=requiredText(settings.SiteDescription,1,300);if err!=nil{return domain.SiteSettings{},err};settings.AboutContent,err=requiredText(settings.AboutContent,1,100000);if err!=nil{return domain.SiteSettings{},err};for _,link:=range settings.SocialLinks{if _,err:=requiredText(link.Name,1,50);err!=nil{return domain.SiteSettings{},err};value:=link.URL;if _,err:=optionalHTTPURL(&value);err!=nil{return domain.SiteSettings{},err}};if err:=s.site.UpdateSettings(ctx,settings,userID);err!=nil{return domain.SiteSettings{},err};return settings,nil
}
func(s *ContentService)PublicStats(ctx context.Context)(int,int,int,int,error){return s.site.PublicStats(ctx)}
func(s *ContentService)Dashboard(ctx context.Context)(domain.Dashboard,error){return s.site.Dashboard(ctx)}

func excerpt(value string,maximum int)string{value=strings.TrimSpace(value);runes:=[]rune(value);if len(runes)<=maximum{return value};return string(runes[:maximum])+"..."}
```

说明：更新文章时 `TagIDs` 的空切片表示清空标签；前端必须始终发送当前完整标签 ID。对 optional 字段需要表达“未改”和“清空”两种语义时，后续契约升级应采用明确 patch DTO，而不是在 Handler 猜测。

## 07-17 创建互动 Service

创建 `server/internal/service/interaction.go`：

```go
package service

import (
	"context"
	"strings"
	"unicode/utf8"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/repository"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/security"
)

type InteractionService struct {
	repository *repository.InteractionRepository
	sanitizer  *security.CommentSanitizer
	secret     []byte
}
func NewInteractionService(repository *repository.InteractionRepository,sanitizer *security.CommentSanitizer,secret []byte)*InteractionService{return &InteractionService{repository:repository,sanitizer:sanitizer,secret:append([]byte(nil),secret...)}}
func(s *InteractionService)List(ctx context.Context,slug string,page,pageSize int)(domain.Page[domain.Comment],error){normalizedPage,normalizedSize,err:=normalizedPage(&page,&pageSize);if err!=nil{return domain.Page[domain.Comment]{},err};return s.repository.ListComments(ctx,slug,normalizedPage,normalizedSize,false,"")}
func(s *InteractionService)ListAdmin(ctx context.Context,page,pageSize int,status string)(domain.Page[domain.Comment],error){normalizedPage,normalizedSize,err:=normalizedPage(&page,&pageSize);if err!=nil{return domain.Page[domain.Comment]{},err};if status!=""{if err:=statusAllowed(status,"pending","approved","rejected");err!=nil{return domain.Page[domain.Comment]{},err}};return s.repository.ListAdminComments(ctx,normalizedPage,normalizedSize,status)}
func(s *InteractionService)Create(ctx context.Context,input domain.CommentInput,email,honeypot string)(domain.Comment,error){
	if strings.TrimSpace(honeypot)!=""{return domain.Comment{},domain.ErrBadRequest};input.Nickname=strings.TrimSpace(input.Nickname);input.Content=s.sanitizer.Clean(input.Content);if utf8.RuneCountInString(input.Nickname)<1||utf8.RuneCountInString(input.Nickname)>100||utf8.RuneCountInString(input.Content)<2||utf8.RuneCountInString(input.Content)>5000||input.VisitorID==""{return domain.Comment{},domain.ErrBadRequest};if email!=""{hash:=security.HMACHash(s.secret,strings.ToLower(strings.TrimSpace(email)));input.EmailHash=&hash};return s.repository.CreateComment(ctx,input)
}
func(s *InteractionService)Review(ctx context.Context,id,userID int64,status string)(domain.Comment,error){if err:=statusAllowed(status,"approved","rejected");err!=nil{return domain.Comment{},err};return s.repository.ReviewComment(ctx,id,userID,status)}
func(s *InteractionService)Delete(ctx context.Context,id int64)error{return s.repository.DeleteComment(ctx,id)}
func(s *InteractionService)SetLike(ctx context.Context,slug,visitorID,ipHash string,liked bool)(bool,int,error){if visitorID==""{return false,0,domain.ErrBadRequest};return s.repository.SetLike(ctx,slug,visitorID,ipHash,liked)}
func(s *InteractionService)LikeStatus(ctx context.Context,slug,visitorID string)(bool,int,error){if visitorID==""{return false,0,domain.ErrBadRequest};return s.repository.LikeStatus(ctx,slug,visitorID)}
```

## 07-18 创建媒体 Service

安装 WebP 解码器：

```powershell
PS> Set-Location 'server'
PS> go get golang.org/x/image@v0.27.0
PS> go mod tidy
PS> Set-Location '..'
```

创建 `server/internal/service/media.go`：

```go
package service

import (
	"bytes"
	"context"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"image"
	_ "image/jpeg"
	_ "image/png"
	"net/http"
	"path/filepath"
	"strings"

	_ "golang.org/x/image/webp"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/repository"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/storage"
)

const maximumImageBytes = 6 << 20

type MediaResult struct{ Asset domain.MediaAsset; URL string }
type MediaService struct{ repository *repository.MediaRepository; store storage.ObjectStore }
func NewMediaService(repository *repository.MediaRepository,store storage.ObjectStore)*MediaService{return &MediaService{repository:repository,store:store}}
func(s *MediaService)Upload(ctx context.Context,userID int64,filename string,data []byte)(MediaResult,error){
	if len(data)==0||len(data)>maximumImageBytes{return MediaResult{},domain.ErrBadRequest}
	mime:=http.DetectContentType(data);extensions:=map[string]string{"image/jpeg":".jpg","image/png":".png","image/webp":".webp"};extension,ok:=extensions[mime];if !ok{return MediaResult{},domain.ErrBadRequest}
	configuration,_,err:=image.DecodeConfig(bytes.NewReader(data));if err!=nil||configuration.Width<1||configuration.Height<1||configuration.Width>8000||configuration.Height>8000{return MediaResult{},domain.ErrBadRequest}
	random:=make([]byte,16);if _,err:=rand.Read(random);err!=nil{return MediaResult{},err};key:="media/"+hex.EncodeToString(random)+extension;sum:=sha256.Sum256(data)
	if err:=s.store.Put(ctx,key,mime,data);err!=nil{return MediaResult{},err}
	asset,err:=s.repository.Create(ctx,domain.MediaAsset{ObjectKey:key,OriginalName:filepath.Base(strings.ReplaceAll(filename,"\\","/")),MIMEType:mime,ByteSize:int64(len(data)),Width:configuration.Width,Height:configuration.Height,SHA256:hex.EncodeToString(sum[:])},userID)
	if err!=nil{deleteErr:=s.store.Delete(ctx,key);if deleteErr!=nil{return MediaResult{},errors.Join(err,deleteErr)};return MediaResult{},err}
	return MediaResult{Asset:asset,URL:s.store.PublicURL(key)},nil
}
```

上传顺序是校验 -> 对象存储 -> 元数据。数据库失败时尽力删除对象；对象存储失败时绝不写元数据。

## 07-19 创建 RSS、Sitemap 和归档 Service

创建 `server/internal/service/feed.go`：

```go
package service

import (
	"context"
	"encoding/xml"
	"fmt"
	"net/url"
	"time"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
)

type rss struct{XMLName xml.Name `xml:"rss"`;Version string `xml:"version,attr"`;Channel rssChannel `xml:"channel"`}
type rssChannel struct{Title string `xml:"title"`;Link string `xml:"link"`;Description string `xml:"description"`;Items []rssItem `xml:"item"`}
type rssItem struct{Title string `xml:"title"`;Link string `xml:"link"`;GUID string `xml:"guid"`;Description string `xml:"description"`;Published string `xml:"pubDate"`}
type sitemap struct{XMLName xml.Name `xml:"urlset"`;XMLNS string `xml:"xmlns,attr"`;URLs []sitemapURL `xml:"url"`}
type sitemapURL struct{Location string `xml:"loc"`;LastModified string `xml:"lastmod,omitempty"`}

func(s *ContentService)RSS(ctx context.Context,publicURL string)([]byte,error){settings,err:=s.site.GetSettings(ctx);if err!=nil{return nil,err};page,err:=s.articles.List(ctx,domain.ArticleFilter{Page:1,PageSize:20});if err!=nil{return nil,err};channel:=rssChannel{Title:settings.SiteTitle,Link:publicURL,Description:settings.SiteDescription,Items:make([]rssItem,0,len(page.Items))};for _,article:=range page.Items{link:=publicURL+"/articles/"+url.PathEscape(article.Slug)+"/";published:="";if article.PublishedAt!=nil{published=article.PublishedAt.Format(time.RFC1123Z)};channel.Items=append(channel.Items,rssItem{Title:article.Title,Link:link,GUID:link,Description:article.Summary,Published:published})};body,err:=xml.MarshalIndent(rss{Version:"2.0",Channel:channel},"","  ");return append([]byte(xml.Header),body...),err}
func(s *ContentService)Sitemap(ctx context.Context,publicURL string)([]byte,error){locations:=[]sitemapURL{{Location:publicURL+"/"},{Location:publicURL+"/articles/"},{Location:publicURL+"/projects/"},{Location:publicURL+"/archive/"},{Location:publicURL+"/about/"}};for pageNumber:=1;;pageNumber++{page,err:=s.articles.List(ctx,domain.ArticleFilter{Page:pageNumber,PageSize:100});if err!=nil{return nil,err};for _,article:=range page.Items{locations=append(locations,sitemapURL{Location:publicURL+"/articles/"+url.PathEscape(article.Slug)+"/",LastModified:article.UpdatedAt.Format("2006-01-02")})};if pageNumber>=page.TotalPages{break}}
	projects,err:=s.projects.List(ctx,domain.ProjectFilter{Page:1,PageSize:100});if err!=nil{return nil,err};for _,project:=range projects.Items{locations=append(locations,sitemapURL{Location:publicURL+"/projects/"+url.PathEscape(project.Slug)+"/",LastModified:project.UpdatedAt.Format("2006-01-02")})};body,err:=xml.MarshalIndent(sitemap{XMLNS:"http://www.sitemaps.org/schemas/sitemap/0.9",URLs:locations},"","  ");return append([]byte(xml.Header),body...),err}
func(s *ContentService)Archive(ctx context.Context)([]domain.Article,error){items:=make([]domain.Article,0);for pageNumber:=1;;pageNumber++{page,err:=s.articles.List(ctx,domain.ArticleFilter{Page:pageNumber,PageSize:100});if err!=nil{return nil,fmt.Errorf("archive page %d: %w",pageNumber,err)};items=append(items,page.Items...);if pageNumber>=page.TotalPages{break}};return items,nil}
```

当文章数为 0 时 `TotalPages=0`，循环在第一次空页后结束；不会无限循环。

## 07-20 扩展 Handler 依赖

打开第 06 章创建的 `server/internal/handler/auth.go`，把 `type Handler` 和 `NewAuth` 两个定义替换为以下代码；文件其余认证方法保持不变：

```go
type Handler struct {
	auth         AuthUseCases
	cookies      security.CookieManager
	content      *service.ContentService
	interactions *service.InteractionService
	media        *service.MediaService
	database     Pinger
	publicURL    string
	now          func() time.Time
}

type Dependencies struct {
	Auth         AuthUseCases
	Cookies      security.CookieManager
	Content      *service.ContentService
	Interactions *service.InteractionService
	Media        *service.MediaService
	Database     Pinger
	PublicURL    string
}

func New(dependencies Dependencies) *Handler {
	return &Handler{
		auth: dependencies.Auth, cookies: dependencies.Cookies,
		content: dependencies.Content, interactions: dependencies.Interactions,
		media: dependencies.Media, database: dependencies.Database,
		publicURL: strings.TrimRight(dependencies.PublicURL, "/"), now: time.Now,
	}
}

func NewAuth(auth AuthUseCases, cookies security.CookieManager) *Handler {
	return New(Dependencies{Auth: auth, Cookies: cookies})
}
```

同时在 `auth.go` import 中加入：

```go
"strings"
```

## 07-21 创建完整 operation dispatch

创建 `server/internal/handler/dispatch.go`：

```go
package handler

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log/slog"
	"mime/multipart"
	"net/http"
	"sort"
	"time"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/generated/oapi"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/domain"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/requestctx"
)

func (h *Handler) dispatch(ctx context.Context, operation string, rawRequest any) (*OperationResponse, error) {
	switch operation {
	case "ListCategories":
		items,err:=h.content.ListCategories(ctx);return result(itemsToAny(items,categoryData),err,http.StatusOK)
	case "GetCategory":
		request:=rawRequest.(oapi.GetCategoryRequestObject);item,err:=h.content.GetCategory(ctx,request.Slug);return result(categoryData(item),err,http.StatusOK)
	case "CreateCategory":
		if response:=requireAdmin(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.CreateCategoryRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};item,err:=h.content.SaveCategory(ctx,"",domain.CategoryInput{Name:request.Body.Name,Slug:stringValue(request.Body.Slug),Description:request.Body.Description,Icon:request.Body.Icon,ParentID:request.Body.ParentID,SortOrder:intValue(request.Body.SortOrder)});return result(categoryData(item),err,http.StatusCreated)
	case "UpdateCategory":
		if response:=requireAdmin(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.UpdateCategoryRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};item,err:=h.content.SaveCategory(ctx,request.Slug,domain.CategoryInput{Name:request.Body.Name,Slug:stringValue(request.Body.Slug),Description:request.Body.Description,Icon:request.Body.Icon,ParentID:request.Body.ParentID,SortOrder:intValue(request.Body.SortOrder)});return result(categoryData(item),err,http.StatusOK)
	case "DeleteCategory":
		if response:=requireAdmin(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.DeleteCategoryRequestObject);return emptyResult(h.content.DeleteCategory(ctx,request.Slug))
	case "ListTags":
		items,err:=h.content.ListTags(ctx);return result(itemsToAny(items,tagData),err,http.StatusOK)
	case "GetTag":
		request:=rawRequest.(oapi.GetTagRequestObject);item,err:=h.content.GetTag(ctx,request.Slug);return result(tagData(item),err,http.StatusOK)
	case "CreateTag":
		if response:=requireAdmin(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.CreateTagRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};item,err:=h.content.SaveTag(ctx,"",domain.TagInput{Name:request.Body.Name,Slug:stringValue(request.Body.Slug),Color:request.Body.Color});return result(tagData(item),err,http.StatusCreated)
	case "UpdateTag":
		if response:=requireAdmin(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.UpdateTagRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};item,err:=h.content.SaveTag(ctx,request.Slug,domain.TagInput{Name:request.Body.Name,Slug:stringValue(request.Body.Slug),Color:request.Body.Color});return result(tagData(item),err,http.StatusOK)
	case "DeleteTag":
		if response:=requireAdmin(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.DeleteTagRequestObject);return emptyResult(h.content.DeleteTag(ctx,request.Slug))
	case "ListArticles":
		request:=rawRequest.(oapi.ListArticlesRequestObject);page,err:=h.content.ListArticles(ctx,domain.ArticleFilter{Page:intPointerValue(request.Params.Page),PageSize:intPointerValue(request.Params.PageSize),Category:stringValue(request.Params.Category),Tag:stringValue(request.Params.Tag),Keyword:stringValue(request.Params.Keyword),Sort:enumString(request.Params.Sort)});return result(articlePageData(page,false),err,http.StatusOK)
	case "ListAdminArticles":
		if response:=requireEditor(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.ListAdminArticlesRequestObject);page,err:=h.content.ListArticles(ctx,domain.ArticleFilter{Page:intPointerValue(request.Params.Page),PageSize:intPointerValue(request.Params.PageSize),Status:enumString(request.Params.Status),Keyword:stringValue(request.Params.Keyword),IncludePrivate:true});return result(articlePageData(page,false),err,http.StatusOK)
	case "GetArticle":
		request:=rawRequest.(oapi.GetArticleRequestObject);item,err:=h.content.GetArticle(ctx,request.Slug,false);return result(articleData(item,true),err,http.StatusOK)
	case "CreateArticle":
		identity,response:=editorIdentity(ctx);if response!=nil{return response,nil};request:=rawRequest.(oapi.CreateArticleRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};input:=articleCreateInput(*request.Body);item,err:=h.content.CreateArticle(ctx,identity.UserID,input);return result(articleData(item,true),err,http.StatusCreated)
	case "UpdateArticle":
		if response:=requireEditor(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.UpdateArticleRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};item,err:=h.content.UpdateArticle(ctx,request.Slug,articleUpdateInput(*request.Body));return result(articleData(item,true),err,http.StatusOK)
	case "DeleteArticle":
		if response:=requireAdmin(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.DeleteArticleRequestObject);return emptyResult(h.content.DeleteArticle(ctx,request.Slug))
	case "PublishArticle":
		if response:=requireEditor(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.PublishArticleRequestObject);item,err:=h.content.PublishArticle(ctx,request.Slug);return result(articleData(item,true),err,http.StatusOK)
	case "UnpublishArticle":
		if response:=requireEditor(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.UnpublishArticleRequestObject);item,err:=h.content.UnpublishArticle(ctx,request.Slug);return result(articleData(item,true),err,http.StatusOK)
	case "SetArticleTop":
		if response:=requireEditor(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.SetArticleTopRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};item,err:=h.content.SetArticleTop(ctx,request.Slug,request.Body.IsTop);return result(articleData(item,true),err,http.StatusOK)
	case "ListArticleArchive":
		items,err:=h.content.Archive(ctx);return result(archiveData(items),err,http.StatusOK)
	case "ListProjects":
		request:=rawRequest.(oapi.ListProjectsRequestObject);page,err:=h.content.ListProjects(ctx,domain.ProjectFilter{Page:intPointerValue(request.Params.Page),PageSize:intPointerValue(request.Params.PageSize),Featured:request.Params.Featured,Technology:stringValue(request.Params.Technology)});return result(projectPageData(page,false),err,http.StatusOK)
	case "ListAdminProjects":
		if response:=requireEditor(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.ListAdminProjectsRequestObject);page,err:=h.content.ListProjects(ctx,domain.ProjectFilter{Page:intPointerValue(request.Params.Page),PageSize:intPointerValue(request.Params.PageSize),Status:enumString(request.Params.Status),IncludePrivate:true});return result(projectPageData(page,false),err,http.StatusOK)
	case "GetProject":
		request:=rawRequest.(oapi.GetProjectRequestObject);item,err:=h.content.GetProject(ctx,request.Slug,false);return result(projectData(item,true),err,http.StatusOK)
	case "CreateProject":
		if response:=requireEditor(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.CreateProjectRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};item,err:=h.content.SaveProject(ctx,"",projectCreateInput(*request.Body));return result(projectData(item,true),err,http.StatusCreated)
	case "UpdateProject":
		if response:=requireEditor(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.UpdateProjectRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};item,err:=h.content.SaveProject(ctx,request.Slug,projectUpdateInput(*request.Body));return result(projectData(item,true),err,http.StatusOK)
	case "DeleteProject":
		if response:=requireAdmin(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.DeleteProjectRequestObject);return emptyResult(h.content.DeleteProject(ctx,request.Slug))
	case "GetSiteSettings":
		settings,err:=h.content.GetSettings(ctx);return result(settings,err,http.StatusOK)
	case "UpdateSiteSettings":
		identity,response:=adminIdentity(ctx);if response!=nil{return response,nil};request:=rawRequest.(oapi.UpdateSiteSettingsRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};bytes,err:=json.Marshal(request.Body);if err!=nil{return nil,err};var settings domain.SiteSettings;if err:=json.Unmarshal(bytes,&settings);err!=nil{return nil,err};settings,err=h.content.UpdateSettings(ctx,identity.UserID,settings);return result(settings,err,http.StatusOK)
	case "GetSiteStats":
		articles,projects,categories,tags,err:=h.content.PublicStats(ctx);return result(map[string]int{"total_articles":articles,"total_projects":projects,"total_categories":categories,"total_tags":tags},err,http.StatusOK)
	case "GetAdminDashboard":
		if response:=requireEditor(ctx);response!=nil{return response,nil};dashboard,err:=h.content.Dashboard(ctx);return result(map[string]int{"total_articles":dashboard.TotalArticles,"draft_articles":dashboard.DraftArticles,"total_projects":dashboard.TotalProjects,"pending_comments":dashboard.PendingComments,"total_views":dashboard.TotalViews,"total_likes":dashboard.TotalLikes},err,http.StatusOK)
	case "ListArticleComments":
		request:=rawRequest.(oapi.ListArticleCommentsRequestObject);page,err:=h.interactions.List(ctx,request.Slug,intPointerValue(request.Params.Page),intPointerValue(request.Params.PageSize));return result(commentPageData(page),err,http.StatusOK)
	case "CreateComment":
		request:=rawRequest.(oapi.CreateCommentRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};metadata:=requestctx.MetadataFrom(ctx);email:="";if request.Body.Email!=nil{email=string(*request.Body.Email)};honeypot:=stringValue(request.Body.WebsiteURL);item,err:=h.interactions.Create(ctx,domain.CommentInput{ArticleSlug:request.Slug,ParentID:request.Body.ParentID,Nickname:request.Body.Nickname,Content:request.Body.Content,VisitorID:metadata.VisitorID,IPHash:metadata.IPHash,UserAgent:metadata.UserAgent},email,honeypot);return result(commentData(item),err,http.StatusCreated)
	case "ListAdminComments":
		if response:=requireEditor(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.ListAdminCommentsRequestObject);page,err:=h.interactions.ListAdmin(ctx,intPointerValue(request.Params.Page),intPointerValue(request.Params.PageSize),enumString(request.Params.Status));return result(commentPageData(page),err,http.StatusOK)
	case "ApproveComment", "RejectComment":
		identity,response:=editorIdentity(ctx);if response!=nil{return response,nil};id:=int64(0);status:="approved";if operation=="ApproveComment"{id=rawRequest.(oapi.ApproveCommentRequestObject).ID}else{id=rawRequest.(oapi.RejectCommentRequestObject).ID;status="rejected"};item,err:=h.interactions.Review(ctx,id,identity.UserID,status);return result(commentData(item),err,http.StatusOK)
	case "DeleteComment":
		if response:=requireAdmin(ctx);response!=nil{return response,nil};request:=rawRequest.(oapi.DeleteCommentRequestObject);return emptyResult(h.interactions.Delete(ctx,request.ID))
	case "SetArticleLike":
		request:=rawRequest.(oapi.SetArticleLikeRequestObject);if request.Body==nil{return Failure(400,40000,"request body is required"),nil};metadata:=requestctx.MetadataFrom(ctx);liked,count,err:=h.interactions.SetLike(ctx,request.Slug,metadata.VisitorID,metadata.IPHash,string(request.Body.Action)=="like");return result(map[string]any{"liked":liked,"like_count":count},err,http.StatusOK)
	case "GetArticleLikeStatus":
		request:=rawRequest.(oapi.GetArticleLikeStatusRequestObject);metadata:=requestctx.MetadataFrom(ctx);liked,count,err:=h.interactions.LikeStatus(ctx,request.Slug,metadata.VisitorID);return result(map[string]any{"liked":liked,"like_count":count},err,http.StatusOK)
	case "UploadImage":
		identity,response:=editorIdentity(ctx);if response!=nil{return response,nil};request:=rawRequest.(oapi.UploadImageRequestObject);filename,data,err:=readUpload(request.Body);if err!=nil{return operationError(err),nil};uploaded,err:=h.media.Upload(ctx,identity.UserID,filename,data);return result(map[string]any{"filename":uploaded.Asset.OriginalName,"url":uploaded.URL,"size":uploaded.Asset.ByteSize,"width":uploaded.Asset.Width,"height":uploaded.Asset.Height},err,http.StatusOK)
	case "GetRss":
		body,err:=h.content.RSS(ctx,h.publicURL);if err!=nil{return nil,err};return &OperationResponse{Status:200,ContentType:"application/rss+xml; charset=utf-8",Body:body},nil
	case "GetSitemap":
		body,err:=h.content.Sitemap(ctx,h.publicURL);if err!=nil{return nil,err};return &OperationResponse{Status:200,ContentType:"application/xml; charset=utf-8",Body:body},nil
	case "GetHealth":
		status:="ok";return Success(200,map[string]any{"status":status,"timestamp":h.now().UTC()}),nil
	case "GetReady":
		pingContext,cancel:=context.WithTimeout(ctx,2*time.Second);defer cancel();if err:=h.database.Ping(pingContext);err!=nil{return JSON(503,map[string]string{"status":"not_ready","database":"disconnected"}),nil};return JSON(200,map[string]string{"status":"ready","database":"connected"}),nil
	default:
		return nil,fmt.Errorf("operation %q is not implemented",operation)
	}
}

func result(data any,err error,status int)(*OperationResponse,error){if err!=nil{return operationError(err),nil};return Success(status,data),nil}
func emptyResult(err error)(*OperationResponse,error){if err!=nil{return operationError(err),nil};return Success(200,nil),nil}
func operationError(err error)*OperationResponse{switch{case errors.Is(err,domain.ErrBadRequest):return Failure(400,40000,"request is invalid");case errors.Is(err,domain.ErrUnauthorized):return Failure(401,40100,"authentication is required");case errors.Is(err,domain.ErrForbidden):return Failure(403,40300,"permission is denied");case errors.Is(err,domain.ErrNotFound):return Failure(404,40400,"resource was not found");case errors.Is(err,domain.ErrConflict):return Failure(409,40900,"resource state conflicts with the request");default:slog.Error("operation_failed","error",err);return Failure(500,50000,"internal server error")}}
func requireAdmin(ctx context.Context)*OperationResponse{_,response:=adminIdentity(ctx);return response}
func requireEditor(ctx context.Context)*OperationResponse{_,response:=editorIdentity(ctx);return response}
func adminIdentity(ctx context.Context)(requestctx.Identity,*OperationResponse){identity,ok:=requestctx.IdentityFrom(ctx);if !ok{return requestctx.Identity{},Failure(401,40100,"authentication is required")};if identity.Role!="admin"{return requestctx.Identity{},Failure(403,40300,"administrator permission is required")};return identity,nil}
func editorIdentity(ctx context.Context)(requestctx.Identity,*OperationResponse){identity,ok:=requestctx.IdentityFrom(ctx);if !ok{return requestctx.Identity{},Failure(401,40100,"authentication is required")};if identity.Role!="admin"&&identity.Role!="editor"{return requestctx.Identity{},Failure(403,40300,"editor permission is required")};return identity,nil}

func readUpload(reader *multipart.Reader)(string,[]byte,error){if reader==nil{return "",nil,domain.ErrBadRequest};for{part,err:=reader.NextPart();if errors.Is(err,io.EOF){break};if err!=nil{return "",nil,domain.ErrBadRequest};if part.FormName()!="file"{_ = part.Close();continue};data,err:=io.ReadAll(io.LimitReader(part,(6<<20)+1));_ = part.Close();if err!=nil||len(data)>(6<<20){return "",nil,domain.ErrBadRequest};return part.FileName(),data,nil};return "",nil,domain.ErrBadRequest}
func stringValue[T ~string](value *T)string{if value==nil{return ""};return string(*value)}
func enumString[T ~string](value *T)string{return stringValue(value)}
func intValue(value *int)int{if value==nil{return 0};return *value}
func intPointerValue[T ~int](value *T)int{if value==nil{return 0};return int(*value)}

func articleCreateInput(body oapi.ArticleCreateRequest)domain.ArticleInput{input:=domain.ArticleInput{Slug:stringValue(body.Slug),Title:body.Title,Content:body.Content,CoverImage:body.CoverImage,CategoryID:body.CategoryID,IsTop:body.IsTop!=nil&&*body.IsTop};if body.Summary!=nil{input.Summary=*body.Summary};if body.TagIds!=nil{input.TagIDs=*body.TagIds};if body.Status!=nil{input.Status=string(*body.Status)};return input}
func articleUpdateInput(body oapi.ArticleUpdateRequest)domain.ArticleInput{input:=domain.ArticleInput{Slug:stringValue(body.Slug),CoverImage:body.CoverImage,CategoryID:body.CategoryID};if body.Title!=nil{input.Title=*body.Title};if body.Summary!=nil{input.Summary=*body.Summary};if body.Content!=nil{input.Content=*body.Content};if body.TagIds!=nil{input.TagIDs=*body.TagIds};if body.IsTop!=nil{input.IsTop=*body.IsTop};return input}
func projectCreateInput(body oapi.ProjectCreateRequest)domain.ProjectInput{input:=domain.ProjectInput{Name:body.Name,Slug:stringValue(body.Slug),CoverImage:body.CoverImage,RepositoryURL:body.RepositoryURL,DemoURL:body.DemoURL,Featured:body.Featured!=nil&&*body.Featured,SortOrder:intValue(body.SortOrder)};if body.Summary!=nil{input.Summary=*body.Summary};if body.Content!=nil{input.Content=*body.Content};if body.Technologies!=nil{input.Technologies=*body.Technologies};if body.Status!=nil{input.Status=string(*body.Status)};if body.StartedAt!=nil{value:=body.StartedAt.Time;input.StartedAt=&value};if body.CompletedAt!=nil{value:=body.CompletedAt.Time;input.CompletedAt=&value};return input}
func projectUpdateInput(body oapi.ProjectUpdateRequest)domain.ProjectInput{input:=domain.ProjectInput{Slug:stringValue(body.Slug),CoverImage:body.CoverImage,RepositoryURL:body.RepositoryURL,DemoURL:body.DemoURL};if body.Name!=nil{input.Name=*body.Name};if body.Summary!=nil{input.Summary=*body.Summary};if body.Content!=nil{input.Content=*body.Content};if body.Technologies!=nil{input.Technologies=*body.Technologies};if body.Featured!=nil{input.Featured=*body.Featured};if body.SortOrder!=nil{input.SortOrder=*body.SortOrder};if body.Status!=nil{input.Status=string(*body.Status)};if body.StartedAt!=nil{value:=body.StartedAt.Time;input.StartedAt=&value};if body.CompletedAt!=nil{value:=body.CompletedAt.Time;input.CompletedAt=&value};return input}

func categoryData(item domain.Category)map[string]any{return map[string]any{"id":item.ID,"name":item.Name,"slug":item.Slug,"description":item.Description,"icon":item.Icon,"parent_id":item.ParentID,"sort_order":item.SortOrder,"article_count":item.ArticleCount,"created_at":item.CreatedAt}}
func tagData(item domain.Tag)map[string]any{return map[string]any{"id":item.ID,"name":item.Name,"slug":item.Slug,"color":item.Color,"article_count":item.ArticleCount,"created_at":item.CreatedAt}}
func articleData(item domain.Article,detail bool)map[string]any{tags:=itemsToAny(item.Tags,tagData);result:=map[string]any{"id":item.ID,"slug":item.Slug,"title":item.Title,"summary":item.Summary,"cover_image":item.CoverImage,"category":nil,"tags":tags,"author":map[string]any{"id":item.Author.ID,"username":item.Author.Username,"nickname":item.Author.Nickname,"avatar_url":item.Author.AvatarURL,"bio":item.Author.Bio,"role":item.Author.Role,"created_at":item.Author.CreatedAt},"status":item.Status,"is_top":item.IsTop,"view_count":item.ViewCount,"like_count":item.LikeCount,"comment_count":item.CommentCount,"published_at":item.PublishedAt,"created_at":item.CreatedAt,"updated_at":item.UpdatedAt};if item.Category!=nil{result["category"]=categoryData(*item.Category)};if detail{result["content"]=item.Content};return result}
func projectData(item domain.Project,detail bool)map[string]any{result:=map[string]any{"id":item.ID,"slug":item.Slug,"name":item.Name,"summary":item.Summary,"cover_image":item.CoverImage,"technologies":item.Technologies,"repository_url":item.RepositoryURL,"demo_url":item.DemoURL,"featured":item.Featured,"sort_order":item.SortOrder,"status":item.Status,"started_at":dateValue(item.StartedAt),"completed_at":dateValue(item.CompletedAt),"created_at":item.CreatedAt,"updated_at":item.UpdatedAt};if detail{result["content"]=item.Content};return result}
func commentData(item domain.Comment)map[string]any{return map[string]any{"id":item.ID,"article_id":item.ArticleID,"parent_id":item.ParentID,"nickname":item.Nickname,"content":item.Content,"status":item.Status,"created_at":item.CreatedAt,"website":nil}}
func articlePageData(page domain.Page[domain.Article],detail bool)map[string]any{items:=make([]any,0,len(page.Items));for _,item:=range page.Items{items=append(items,articleData(item,detail))};return pageData(items,page.Page,page.PageSize,page.Total,page.TotalPages)}
func projectPageData(page domain.Page[domain.Project],detail bool)map[string]any{items:=make([]any,0,len(page.Items));for _,item:=range page.Items{items=append(items,projectData(item,detail))};return pageData(items,page.Page,page.PageSize,page.Total,page.TotalPages)}
func commentPageData(page domain.Page[domain.Comment])map[string]any{items:=make([]any,0,len(page.Items));for _,item:=range page.Items{items=append(items,commentData(item))};return pageData(items,page.Page,page.PageSize,page.Total,page.TotalPages)}
func pageData(items []any,page,pageSize,total,totalPages int)map[string]any{return map[string]any{"items":items,"pagination":map[string]int{"page":page,"page_size":pageSize,"total":total,"total_pages":totalPages}}}
func itemsToAny[T any](items []T,convert func(T)map[string]any)[]any{result:=make([]any,0,len(items));for _,item:=range items{result=append(result,convert(item))};return result}
func dateValue(value *time.Time)any{if value==nil{return nil};return value.Format("2006-01-02")}
func archiveData(items []domain.Article)[]any{sort.SliceStable(items,func(i,j int)bool{left,right:=items[i].PublishedAt,items[j].PublishedAt;if left==nil{return false};if right==nil{return true};return left.After(*right)});years:=make([]any,0);yearIndex:=-1;monthIndex:=-1;currentYear,currentMonth:=-1,-1;var months []any;var articles []any;flushMonth:=func(){if monthIndex>=0{months=append(months,map[string]any{"month":currentMonth,"articles":articles})}};flushYear:=func(){if yearIndex>=0{flushMonth();years=append(years,map[string]any{"year":currentYear,"months":months})}};for _,item:=range items{if item.PublishedAt==nil{continue};year,month,_:=item.PublishedAt.Date();if year!=currentYear{flushYear();yearIndex++;currentYear=year;currentMonth=-1;monthIndex=-1;months=[]any{}};if int(month)!=currentMonth{flushMonth();monthIndex++;currentMonth=int(month);articles=[]any{}};articles=append(articles,map[string]any{"slug":item.Slug,"title":item.Title,"published_at":item.PublishedAt})};flushYear();return years}
```

`dispatch` 的 default 返回 error，不返回“成功空数据”。所有写 operation 显式调用 `requireAdmin` 或 `requireEditor`。

## 07-22 扩展配置全文

把 `server/internal/config/config.go` 替换为以下文件全文：

```go
package config

import (
	"errors"
	"fmt"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

type Config struct {
	Environment, HTTPAddress, DatabaseURL string
	DatabaseMaxConns, DatabaseMinConns int32
	DatabaseTimeout time.Duration
	PublicSiteURL string
	AllowedOrigins []string
	JWTPrivateKeyFile, JWTPublicKeyFile, JWTIssuer, JWTAudience string
	AccessTTL, RefreshTTL time.Duration
	RefreshCookieName, VisitorCookieName, VisitorSecretFile string
	CookieSecure bool
	UploadRoot, MediaPublicURL string
}

func Load() (Config,error) {
	maxConns,err:=int32Environment("DATABASE_MAX_CONNS",10);if err!=nil{return Config{},err}
	minConns,err:=int32Environment("DATABASE_MIN_CONNS",1);if err!=nil{return Config{},err}
	timeout,err:=durationEnvironment("DATABASE_TIMEOUT",5*time.Second);if err!=nil{return Config{},err}
	accessTTL,err:=durationEnvironment("ACCESS_TOKEN_TTL",15*time.Minute);if err!=nil{return Config{},err}
	refreshTTL,err:=durationEnvironment("REFRESH_TOKEN_TTL",30*24*time.Hour);if err!=nil{return Config{},err}
	secure,err:=boolEnvironment("COOKIE_SECURE",false);if err!=nil{return Config{},err}
	cfg:=Config{
		Environment:environment("APP_ENV","development"),HTTPAddress:environment("HTTP_ADDRESS",":8080"),
		DatabaseURL:environment("DATABASE_URL","postgres://personal_site:personal_site_dev@localhost:5432/personal_site?sslmode=disable"),DatabaseMaxConns:maxConns,DatabaseMinConns:minConns,DatabaseTimeout:timeout,
		PublicSiteURL:environment("PUBLIC_SITE_URL","http://localhost:3000"),AllowedOrigins:splitEnvironment("ALLOWED_ORIGINS",[]string{"http://localhost:3000"}),
		JWTPrivateKeyFile:environment("JWT_PRIVATE_KEY_FILE","../secrets/jwt_private_key.pem"),JWTPublicKeyFile:environment("JWT_PUBLIC_KEY_FILE","../secrets/jwt_public_key.pem"),JWTIssuer:environment("JWT_ISSUER","personal-site-api"),JWTAudience:environment("JWT_AUDIENCE","personal-site-admin"),AccessTTL:accessTTL,RefreshTTL:refreshTTL,
		RefreshCookieName:environment("REFRESH_COOKIE_NAME","personal_site_refresh"),VisitorCookieName:environment("VISITOR_COOKIE_NAME","personal_site_visitor"),VisitorSecretFile:environment("VISITOR_SECRET_FILE","../secrets/visitor_secret.txt"),CookieSecure:secure,
		UploadRoot:environment("UPLOAD_ROOT","../uploads"),MediaPublicURL:environment("MEDIA_PUBLIC_URL","http://localhost:8080/uploads"),
	}
	if err:=cfg.Validate();err!=nil{return Config{},err};return cfg,nil
}

func(c Config)Validate()error{
	if c.HTTPAddress==""{return errors.New("HTTP_ADDRESS is required")}
	if c.DatabaseMinConns<0||c.DatabaseMaxConns<1||c.DatabaseMinConns>c.DatabaseMaxConns{return errors.New("database connection limits are invalid")}
	if c.DatabaseTimeout<=0||c.AccessTTL<=0||c.RefreshTTL<=0{return errors.New("timeouts and token TTLs must be positive")}
	for name,rawURL:=range map[string]string{"DATABASE_URL":c.DatabaseURL,"PUBLIC_SITE_URL":c.PublicSiteURL,"MEDIA_PUBLIC_URL":c.MediaPublicURL}{parsed,err:=url.ParseRequestURI(rawURL);if err!=nil||parsed.Scheme==""||parsed.Host==""{return fmt.Errorf("%s must be an absolute URL",name)}}
	if len(c.AllowedOrigins)==0{return errors.New("ALLOWED_ORIGINS must contain at least one origin")}
	if c.JWTPrivateKeyFile==""||c.JWTPublicKeyFile==""||c.VisitorSecretFile==""||c.RefreshCookieName==""||c.VisitorCookieName==""{return errors.New("authentication file and cookie settings are required")}
	return nil
}
func environment(key,fallback string)string{if value,ok:=os.LookupEnv(key);ok{if trimmed:=strings.TrimSpace(value);trimmed!=""{return trimmed}};return fallback}
func splitEnvironment(key string,fallback []string)[]string{raw:=environment(key,"");if raw==""{return fallback};parts:=strings.Split(raw,",");result:=make([]string,0,len(parts));for _,part:=range parts{if value:=strings.TrimSpace(part);value!=""{result=append(result,value)}};return result}
func int32Environment(key string,fallback int32)(int32,error){raw:=environment(key,"");if raw==""{return fallback,nil};value,err:=strconv.ParseInt(raw,10,32);if err!=nil{return 0,fmt.Errorf("%s must be an integer: %w",key,err)};return int32(value),nil}
func durationEnvironment(key string,fallback time.Duration)(time.Duration,error){raw:=environment(key,"");if raw==""{return fallback,nil};value,err:=time.ParseDuration(raw);if err!=nil{return 0,fmt.Errorf("%s must be a duration: %w",key,err)};return value,nil}
func boolEnvironment(key string,fallback bool)(bool,error){raw:=environment(key,"");if raw==""{return fallback,nil};value,err:=strconv.ParseBool(raw);if err!=nil{return false,fmt.Errorf("%s must be a boolean: %w",key,err)};return value,nil}
```

运行第 05 章配置测试；原测试仍应通过。

## 07-23 替换 HTTP Server 全文

把 `server/internal/server/server.go` 替换为：

```go
package server

import (
	"log/slog"
	"net/http"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/labstack/echo/v4"
	echomiddleware "github.com/labstack/echo/v4/middleware"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/generated/oapi"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/handler"
	appmiddleware "github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/middleware"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/security"
)

type Dependencies struct {
	Database *pgxpool.Pool
	Logger *slog.Logger
	API oapi.StrictServerInterface
	AccessVerifier appmiddleware.AccessVerifier
	VisitorSigner security.VisitorSigner
	RefreshCookieName,VisitorCookieName string
	CookieSecure bool
	VisitorSecret []byte
	AllowedOrigins []string
	UploadRoot string
}

func New(dependencies Dependencies)*echo.Echo{
	e:=echo.New();e.HideBanner=true;e.HidePort=true;e.Server.ReadHeaderTimeout=5*time.Second;e.Server.ReadTimeout=15*time.Second;e.Server.WriteTimeout=30*time.Second;e.Server.IdleTimeout=60*time.Second
	e.Use(echomiddleware.RequestID());e.Use(echomiddleware.Recover());e.Use(echomiddleware.BodyLimit("7M"));e.Use(appmiddleware.SecurityHeaders)
	e.Use(echomiddleware.CORSWithConfig(echomiddleware.CORSConfig{AllowOrigins:dependencies.AllowedOrigins,AllowMethods:[]string{http.MethodGet,http.MethodPost,http.MethodPut,http.MethodDelete,http.MethodOptions},AllowHeaders:[]string{echo.HeaderAccept,echo.HeaderAuthorization,echo.HeaderContentType,echo.HeaderOrigin,"X-Requested-With"},AllowCredentials:true,MaxAge:600}))
	e.Use(appmiddleware.AuthContext(dependencies.AccessVerifier,dependencies.RefreshCookieName,dependencies.VisitorSecret))
	e.Use(appmiddleware.VisitorIdentity(dependencies.VisitorCookieName,dependencies.CookieSecure,dependencies.VisitorSigner))
	e.Use(appmiddleware.CSRF(dependencies.AllowedOrigins));e.Use(appmiddleware.NewLoginLimiter().Middleware);e.Use(appmiddleware.AccessLog(dependencies.Logger))

	strict:=oapi.NewStrictHandler(dependencies.API,nil)
	oapi.RegisterHandlersWithBaseURL(e,strict,"/api/v1")
	health:=handler.NewHealth(dependencies.Database);e.GET("/healthz",health.Liveness);e.GET("/readyz",health.Readiness)
	if dependencies.UploadRoot!=""{e.Static("/uploads",dependencies.UploadRoot)}
	e.GET("/",func(c echo.Context)error{return c.NoContent(http.StatusNotFound)})
	return e
}
```

中间件顺序从外到内：Request ID -> panic recovery -> body limit -> security/CORS -> auth metadata -> visitor -> CSRF -> login limit -> access log -> generated binding -> Handler。

## 07-24 替换 API 入口全文

把 `server/cmd/api/main.go` 替换为：

```go
package main

import (
	"context"
	"encoding/base64"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"strings"
	"syscall"
	"time"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/config"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/database"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/handler"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/repository"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/security"
	appserver "github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/server"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/service"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/storage"
)

func main(){
	logger:=slog.New(slog.NewJSONHandler(os.Stdout,&slog.HandlerOptions{Level:slog.LevelInfo}));slog.SetDefault(logger)
	cfg,err:=config.Load();if err!=nil{fatal(logger,"load_configuration",err)}
	rootContext,stop:=signal.NotifyContext(context.Background(),os.Interrupt,syscall.SIGTERM);defer stop()
	pool,err:=database.Open(rootContext,database.Options{URL:cfg.DatabaseURL,MaxConns:cfg.DatabaseMaxConns,MinConns:cfg.DatabaseMinConns,Timeout:cfg.DatabaseTimeout});if err!=nil{fatal(logger,"connect_database",err)};defer pool.Close()
	jwtManager,err:=security.LoadJWTManager(cfg.JWTPrivateKeyFile,cfg.JWTPublicKeyFile,cfg.JWTIssuer,cfg.JWTAudience,cfg.AccessTTL);if err!=nil{fatal(logger,"load_jwt",err)}
	secretEncoded,err:=os.ReadFile(cfg.VisitorSecretFile);if err!=nil{fatal(logger,"read_visitor_secret",err)};visitorSecret,err:=base64.RawURLEncoding.DecodeString(strings.TrimSpace(string(secretEncoded)));if err!=nil{fatal(logger,"decode_visitor_secret",err)};visitorSigner,err:=security.NewVisitorSigner(visitorSecret);if err!=nil{fatal(logger,"create_visitor_signer",err)}
	objectStore,err:=storage.NewLocal(cfg.UploadRoot,cfg.MediaPublicURL);if err!=nil{fatal(logger,"create_object_store",err)}
	authRepository:=repository.NewAuthRepository(pool);categoryRepository:=repository.NewCategoryRepository(pool);tagRepository:=repository.NewTagRepository(pool);articleRepository:=repository.NewArticleRepository(pool);projectRepository:=repository.NewProjectRepository(pool);siteRepository:=repository.NewSiteRepository(pool);interactionRepository:=repository.NewInteractionRepository(pool);mediaRepository:=repository.NewMediaRepository(pool)
	authService:=service.NewAuthService(authRepository,jwtManager,cfg.RefreshTTL);contentService:=service.NewContentService(categoryRepository,tagRepository,articleRepository,projectRepository,siteRepository);interactionService:=service.NewInteractionService(interactionRepository,security.NewCommentSanitizer(),visitorSecret);mediaService:=service.NewMediaService(mediaRepository,objectStore)
	api:=handler.New(handler.Dependencies{Auth:authService,Cookies:security.CookieManager{Name:cfg.RefreshCookieName,Secure:cfg.CookieSecure,TTL:cfg.RefreshTTL},Content:contentService,Interactions:interactionService,Media:mediaService,Database:pool,PublicURL:cfg.PublicSiteURL})
	application:=appserver.New(appserver.Dependencies{Database:pool,Logger:logger,API:api,AccessVerifier:jwtManager,VisitorSigner:visitorSigner,RefreshCookieName:cfg.RefreshCookieName,VisitorCookieName:cfg.VisitorCookieName,CookieSecure:cfg.CookieSecure,VisitorSecret:visitorSecret,AllowedOrigins:cfg.AllowedOrigins,UploadRoot:cfg.UploadRoot})
	serverErrors:=make(chan error,1);go func(){logger.Info("api_starting","address",cfg.HTTPAddress,"environment",cfg.Environment);serverErrors<-application.Start(cfg.HTTPAddress)}()
	select{case<-rootContext.Done():logger.Info("shutdown_requested");case err:=<-serverErrors:if err!=nil&&!errors.Is(err,http.ErrServerClosed){logger.Error("http_server_failed","error",err)}}
	shutdownContext,cancel:=context.WithTimeout(context.Background(),10*time.Second);defer cancel();if err:=application.Shutdown(shutdownContext);err!=nil{fatal(logger,"graceful_shutdown_failed",err)};logger.Info("api_stopped")
}
func fatal(logger *slog.Logger,message string,err error){logger.Error(message,"error",err);os.Exit(1)}
```

## 07-25 添加编译期完整性断言

在 `server/internal/handler/dispatch.go` import 后、`dispatch` 前加入：

```go
var _ oapi.StrictServerInterface = (*Handler)(nil)
```

如果任何 operation 没有由认证方法或生成适配器覆盖，编译会在这里失败。

## 07-26 更新生成漂移门禁

把 `scripts/check-generated.ps1` 中的 `git diff --exit-code` 命令替换为：

```powershell
git diff --exit-code -- 'server/generated/oapi/api.gen.go' 'server/internal/handler/adapter.gen.go' 'web/types/openapi.d.ts'
```

把 `.github/workflows/ci.yml` 的 `Fail on generated drift` run 命令替换为：

```yaml
run: git diff --exit-code -- server/generated/oapi/api.gen.go server/internal/handler/adapter.gen.go web/types/openapi.d.ts
```

执行两次：

```powershell
PS> .\scripts\generate.ps1
PS> .\scripts\check-generated.ps1
```

预期第二次无 diff。

## 07-27 替换 Server 测试全文

把 `server/internal/server/server_test.go` 替换为：

```go
package server_test

import (
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/require"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/handler"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/security"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/server"
)

func TestServerMiddlewareAndLivenessRoute(t *testing.T){
	signer,err:=security.NewVisitorSigner([]byte("0123456789abcdef0123456789abcdef"));require.NoError(t,err)
	application:=server.New(server.Dependencies{Logger:slog.New(slog.NewTextHandler(io.Discard,nil)),API:handler.New(handler.Dependencies{}),VisitorSigner:signer,RefreshCookieName:"refresh",VisitorCookieName:"visitor",VisitorSecret:[]byte("0123456789abcdef0123456789abcdef"),AllowedOrigins:[]string{"http://localhost:3000"}})
	request:=httptest.NewRequest(http.MethodGet,"/healthz",nil);recorder:=httptest.NewRecorder();application.ServeHTTP(recorder,request)
	require.Equal(t,http.StatusOK,recorder.Code);require.Equal(t,"nosniff",recorder.Header().Get("X-Content-Type-Options"));require.NotEmpty(t,recorder.Header().Get("X-Request-Id"))
}
```

## 07-28 创建路由合同测试

创建 `server/tests/routes_test.go`：

```go
package tests

import (
	"io"
	"log/slog"
	"strings"
	"testing"

	"github.com/getkin/kin-openapi/openapi3"
	"github.com/stretchr/testify/require"

	"github.com/<GITHUB_USER>/<REPOSITORY>/server/generated/oapi"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/handler"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/security"
	"github.com/<GITHUB_USER>/<REPOSITORY>/server/internal/server"
)

func TestEveryOpenAPIOperationIsRegistered(t *testing.T){
	signer,err:=security.NewVisitorSigner([]byte("0123456789abcdef0123456789abcdef"));require.NoError(t,err)
	application:=server.New(server.Dependencies{Logger:slog.New(slog.NewTextHandler(io.Discard,nil)),API:handler.New(handler.Dependencies{}),VisitorSigner:signer,RefreshCookieName:"refresh",VisitorCookieName:"visitor",VisitorSecret:[]byte("0123456789abcdef0123456789abcdef"),AllowedOrigins:[]string{"http://localhost:3000"}})
	registered:=make(map[string]struct{});for _,route:=range application.Routes(){registered[route.Method+" "+route.Path]=struct{}{}}
	specification,err:=oapi.GetSwagger();require.NoError(t,err)
	for path,item:=range specification.Paths.Map(){for method,operation:=range operations(item){if operation==nil{continue};echoPath:="/api/v1"+strings.ReplaceAll(strings.ReplaceAll(path,"{",":"),"}","");_,ok:=registered[strings.ToUpper(method)+" "+echoPath];require.Truef(t,ok,"operation %s is not registered at %s %s",operation.OperationID,method,echoPath)}}
}

func operations(item *openapi3.PathItem)map[string]*openapi3.Operation{return map[string]*openapi3.Operation{"get":item.Get,"post":item.Post,"put":item.Put,"delete":item.Delete,"patch":item.Patch,"head":item.Head,"options":item.Options,"trace":item.Trace}}
```

这个测试证明契约 operation 有路由，不等于证明业务正确；Repository 集成测试和 HTTP 冒烟仍必须执行。

## 07-29 格式、测试、迁移与构建

```powershell
PS> docker compose -f compose.dev.yml up -d --wait postgres
PS> docker compose -f compose.dev.yml run --rm migrate
PS> Set-Location 'server'
PS> go generate ./...
PS> gofmt -w cmd internal tests
PS> go mod tidy
PS> go mod verify
PS> go vet ./...
PS> $env:TEST_DATABASE_URL='postgres://personal_site:personal_site_dev@localhost:5432/personal_site?sslmode=disable'
PS> go test ./... -race -count=1 -cover
PS> Remove-Item Env:TEST_DATABASE_URL
PS> go build ./cmd/...
PS> Set-Location '..'
PS> .\scripts\check-generated.ps1
PS> pnpm format
PS> pnpm format:check
PS> git diff --check
```

预期没有 compile error、Skip、race、生成漂移或格式差异。遇到错误时只修当前文件，不删除测试或降低约束。

## 07-30 运行完整 API 冒烟

终端 1：

```powershell
PS> .\scripts\run-api.ps1
```

终端 2：

```powershell
PS> $originHeaders=@{'Origin'='http://localhost:3000';'X-Requested-With'='XMLHttpRequest'}
PS> $session=New-Object Microsoft.PowerShell.Commands.WebRequestSession
PS> $credential=Get-Credential -UserName '<管理员用户名>' -Message '输入管理员密码'
PS> $loginBody=@{username=$credential.UserName;password=$credential.GetNetworkCredential().Password}|ConvertTo-Json
PS> $login=Invoke-RestMethod -Uri 'http://localhost:8080/api/v1/auth/login' -Method Post -ContentType 'application/json' -Headers $originHeaders -WebSession $session -Body $loginBody
PS> $access=$login.data.access_token
PS> $credential=$null; $loginBody=$null; $login=$null
PS> $adminHeaders=@{'Origin'='http://localhost:3000';'X-Requested-With'='XMLHttpRequest';'Authorization'="Bearer $access"}
PS> Invoke-RestMethod -Uri 'http://localhost:8080/api/v1/auth/me' -Headers $adminHeaders
PS> $category=Invoke-RestMethod -Uri 'http://localhost:8080/api/v1/categories' -Method Post -ContentType 'application/json' -Headers $adminHeaders -Body (@{name='Engineering';slug='engineering';parent_id=$null}|ConvertTo-Json)
PS> $articleBody=@{title='Welcome';slug='welcome';summary='The first published article.';content='# Welcome';category_id=$category.data.id;tag_ids=@();status='published';is_top=$true;cover_image=$null}|ConvertTo-Json
PS> Invoke-RestMethod -Uri 'http://localhost:8080/api/v1/articles' -Method Post -ContentType 'application/json' -Headers $adminHeaders -Body $articleBody
PS> Invoke-RestMethod -Uri 'http://localhost:8080/api/v1/articles/welcome'
PS> Invoke-WebRequest -Uri 'http://localhost:8080/api/v1/rss.xml' | Select-Object StatusCode,Headers
PS> Invoke-WebRequest -Uri 'http://localhost:8080/api/v1/sitemap.xml' | Select-Object StatusCode,Headers
PS> $refreshed=Invoke-RestMethod -Uri 'http://localhost:8080/api/v1/auth/refresh' -Method Post -Headers $originHeaders -WebSession $session
PS> $access=$refreshed.data.access_token; $refreshed=$null
PS> Invoke-RestMethod -Uri 'http://localhost:8080/api/v1/auth/logout' -Method Post -Headers $originHeaders -WebSession $session
PS> $access=$null; $adminHeaders=$null; $articleBody=$null
```

预期：登录成功但 Token 未打印；分类创建 201；文章创建 201；公开详情 200；RSS/Sitemap 200 且内容类型为 XML；刷新后 Cookie 轮换；登出成功。

错误路径：不带 Authorization 创建分类应为 401；使用错误 Origin 登录应为 403；请求不存在文章应为 404。

## 07-31 提交与 PR

```powershell
PS> git status --short
PS> git diff --check
PS> git add -- server/migrations/000005_normalize_site_settings.up.sql server/migrations/000005_normalize_site_settings.down.sql server/internal/domain server/internal/repository/category.go server/internal/repository/tag.go server/internal/repository/article.go server/internal/repository/project.go server/internal/repository/site.go
PS> git commit -m 'feat(content): add PostgreSQL content repositories'
PS> git add -- server/internal/repository/interaction.go server/internal/repository/media.go server/internal/security/sanitize.go server/internal/security/visitor.go server/internal/security/visitor_test.go server/internal/storage server/internal/service
PS> git commit -m 'feat(content): add validated content and interaction services'
PS> git add -- server/internal/handler server/internal/middleware/visitor.go server/internal/config/config.go server/internal/server/server.go server/internal/server/server_test.go server/cmd/api/main.go server/tests server/go.mod server/go.sum scripts/check-generated.ps1 .github/workflows/ci.yml
PS> git commit -m 'feat(api): implement and register complete OpenAPI server'
PS> git push --set-upstream origin feat/content-api
```

PR 必须列出 47 个 operation 的路由合同结果、up 到 migration 5、race 测试、管理员冒烟和公开读取结果。CI 全绿后合并并清理分支。

## 07-32 本章停止点

- [ ] OpenAPI 所有 operation 注册，无 501；
- [ ] 公共文章只能读取 published，公共项目只能读取 active；
- [ ] 所有用户筛选值参数化，排序来自白名单；
- [ ] 文章标签、评论审核、点赞计数均在事务内；
- [ ] 评论最多两层且先清洗；
- [ ] 图片限制 MIME、6 MiB、解码和尺寸；
- [ ] RSS/Sitemap 是合法 XML 且不含草稿；
- [ ] Strict 编译断言、route contract、race、build、漂移检查全通过；
- [ ] main 已合并，本地工作区干净。
