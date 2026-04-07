# Task: Implement "Related Posts" Section

## Context
The user wants to increase user engagement by showing a "Related Articles" (or "You might also like") section at the bottom of each blog post.
Current Stack: Astro, TypeScript, Content Collections (`blog`).
Existing Component: `src/components/blog/BlogPostCard.astro` (We should reuse this if possible).

## Objective
1. Create a utility function to calculate and find relevant posts based on tags and categories.
2. Create a UI component to display these posts.
3. Integrate it into the `BlogPostLayout`.

## Plan of Action

### Step 1: Create Logic Helper `src/utils/post-utils.ts`
Create a function named `getRelatedPosts`.

**Function Signature:**
```typescript
export function getRelatedPosts(
  allPosts: CollectionEntry<'blog'>[],
  currentSlug: string,
  currentTags: string[],
  currentCategory: string,
  limit: number = 3
): CollectionEntry<'blog'>[]
```

**Algorithm Logic (Scoring System):**
1. **Filter:** Exclude the current post (using `currentSlug`).
2. **Score Calculation:** Iterate through `allPosts` and assign a score to each:
   - **Tags Match:** +2 points for *each* tag that matches `currentTags`.
   - **Category Match:** +1 point if `category` matches `currentCategory`.
3. **Sorting:**
   - Primary Sort: Score (Descending).
   - Secondary Sort: `pubDate` (Descending / Newest first) for tie-breaking.
4. **Return:** The top `limit` posts (e.g., top 3).
   - *Edge Case:* If no posts have a score > 0 (no relation found), fall back to the 3 most recent posts (excluding current), so the section is never empty.

### Step 2: Create Component `src/components/blog/RelatedPosts.astro`
This component will handle the layout.

**Props:**
- `relatedPosts`: `CollectionEntry<'blog'>[]`

**Implementation Details:**
- **Container:** Use a `<section>` with a top margin/border to separate it from the main content.
- **Heading:** Add a heading like "You might also like" or "Related Articles".
- **Grid Layout:** Use a CSS Grid (similar to `FeatureGrid` or the main blog index).
  - Columns: 1 on mobile, 3 on desktop.
  - Gap: `1.5rem`.
- **Cards:** **REUSE** the existing `<BlogPostCard post={post} />`. Do not duplicate card styling logic.
- **Responsiveness:** Ensure it looks good on mobile (stack vertically).

### Step 3: Integrate into `src/layouts/BlogPostLayout.astro`
1. **Fetch Data:**
   - Inside the frontmatter script, use `getCollection('blog')` to fetch all posts.
   - Import and call `getRelatedPosts` passing the necessary data from the current post (`post.data`).
2. **Render:**
   - Place the `<RelatedPosts />` component at the bottom of the layout.
   - **Placement:** It should be *below* the `<ShareButtons />` (if implemented) or below the `post-footer` (tags section), but *above* the site-wide `Footer`.
   - **Condition:** Always render (since we implemented a fallback in Step 1).

### Step 4: Configuration
- Add `'./src/components/blog/RelatedPosts.astro'` to `astro.config.mjs` in the `autoImport` list (if you want to use it elsewhere, though it's mostly for layouts).

## Styles & Visuals
- **Heading:** Use `h2` or `h3` styling consistent with `global.css`.
- **Spacing:** Ensure generous whitespace (margin-top: 4rem) so it feels like a distinct section, not part of the article body.
- **Background:** Optionally, give the section a slightly different background color (e.g., `var(--bg-color)` if the main content is white) to visually separate it, OR keep it clean and just use a separator line. *Decision: Keep it clean, use a top border separator.*

## Verification
- Check a post with unique tags: Should show posts with same tags.
- Check a post with no common tags but same category: Should show same category.
- Check a completely isolated post: Should show latest posts.

