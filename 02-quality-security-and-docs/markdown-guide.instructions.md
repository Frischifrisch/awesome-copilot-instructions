---
description: 'Markdown formatting: CommonMark, GFM, accessibility and content creation'
applyTo: '**/*.md'
---


# CommonMark Markdown

Apply these rules per the [CommonMark spec 0.31.2](https://spec.commonmark.org/0.31.2/) when writing or reviewing `.md` files. CommonMark spec for reference only. Do not download CommonMark spec.

## Preliminaries

- A line ends at a newline (`U+000A`), carriage return (`U+000D`), or end of file. A blank line contains only spaces or tabs.
- Tabs behave as 4-space tab stops for block structure but are not expanded in content.
- Replace `U+0000` with the replacement character `U+FFFD`.
- **Backslash escapes**: `\` before any ASCII punctuation character renders the literal character. Not recognized in code spans, code blocks, or autolinks.
- **Entity and numeric character references**: `&amp;`, `&#123;`, `&#x7B;` — valid HTML5 entities only. Not recognized in code spans or code blocks. Cannot replace structural characters.

## Leaf Blocks

- **Thematic breaks**: 3+ matching `-`, `_`, or `*` characters on a line with 0–3 spaces indent. Only spaces or tabs allowed on the line otherwise. Can interrupt a paragraph.
- **ATX headings**: 1–6 `#` characters followed by a space or end of line. Optional closing `#` sequence (preceded by a space). 0–3 spaces indent allowed.
- **Setext headings**: Text underlined with `=` (level 1) or `-` (level 2). Cannot interrupt a paragraph — blank line required after a preceding paragraph.
- **Indented code blocks**: Lines indented 4+ spaces. Cannot interrupt a paragraph. Content is literal text, not parsed as Markdown.
- **Fenced code blocks**: Open with 3+ backticks or tildes (do not mix). Closing fence must use same character with at least the same count. Info string after backtick fence cannot contain backticks. Specify language identifier after the opening fence. Content is literal text.
- **HTML blocks**: Seven types defined by start/end tag conditions. Types 1–5 end at their matching end pattern. Type 6 ends at a blank line. Type 7 cannot interrupt a paragraph and ends at a blank line.
- **Link reference definitions**: `[label]: destination "title"`. Case-insensitive label matching (Unicode case fold). First definition wins for duplicate labels. Cannot interrupt a paragraph.
- **Paragraphs**: Consecutive non-blank lines not interpretable as other block constructs. Leading spaces up to 3 are stripped.
- **Blank lines**: Ignored between blocks; determine whether a list is tight or loose.

## Container Blocks

- **Block quotes**: Lines prefixed with `>` (optionally followed by a space). Lazy continuation allowed for paragraph text only. A blank line separates consecutive block quotes.
- **List items**: Bullet markers (`-`, `+`, `*`) or ordered markers (1–9 digits + `.` or `)`). Content column determined by marker width + spaces to first non-whitespace (1–4 spaces after marker). Sublists must be indented to the content column. An ordered list interrupting a paragraph must start with `1`.
- **Lists**: Sequence of same-type list items. Changing bullet character or ordered delimiter starts a new list. A list is loose if any item is separated by a blank line.

## Inlines

- **Code spans**: Backtick-delimited inline code. Line endings convert to spaces. Leading and trailing space stripped when both present (unless content is all spaces). Backslash escapes are literal inside code spans.
- **Emphasis and strong emphasis**: `*`/`_` for `<em>`, `**`/`__` for `<strong>`. `_` is not allowed for intraword emphasis. Left-flanking / right-flanking delimiter run rules apply. Delimiter run length sum must not be a multiple of 3 when one delimiter can both open and close (unless both lengths are multiples of 3).
- **Links**: Inline `[text](url "title")` or reference `[text][label]` / `[text][]` / `[text]`. Link text may contain inlines but not other links. Destination in `<…>` allows spaces; without angle brackets, balanced parentheses allowed. No whitespace between link text and `(` or `[`.
- **Images**: `![alt](src "title")` — same syntax as links prefixed with `!`. Alt text is the plain-string content of the description.
- **Autolinks**: `<URI>` or `<email>` in angle brackets. Scheme must be 2–32 characters starting with an ASCII letter. Bare URLs are not auto-linked in CommonMark (requires angle brackets).
- **Raw HTML**: Open/close tags, comments (`<!--` … `-->`), processing instructions (`<?` … `?>`), declarations (`<!` … `>`), CDATA (`<![CDATA[` … `]]>`) are passed through as literal HTML.
- **Hard line breaks**: Two+ trailing spaces or `\` before a line ending. Not recognized in code spans or HTML tags. Does not work at end of a block.
- **Soft line breaks**: A line ending not preceded by two+ spaces or `\`. Rendered as a space in browsers.

## Validation Checklist

- [ ] ATX headings use 1–6 `#` followed by a space.
- [ ] Fenced code blocks specify a language identifier and use matching fence characters and counts.
- [ ] Backtick fence info strings do not contain backtick characters.
- [ ] Indented code blocks are preceded by a blank line (they cannot interrupt a paragraph).
- [ ] Emphasis uses `*` for intraword; `_` only at word boundaries.
- [ ] Links use `[text](url)` or reference syntax with no whitespace before `(` or `[`.
- [ ] Images include non-empty alt text.
- [ ] Autolinks use angle brackets (`<URL>`); bare URLs are not CommonMark autolinks.
- [ ] No unbalanced parentheses in bare link destinations (use `<…>` or escape).
- [ ] HTML block type 7 (custom/inline-level tags) is preceded by a blank line when following a paragraph.

---


# GitHub Flavored Markdown (GFM)

Apply these rules per the [GFM spec](https://github.github.com/gfm/) when writing or reviewing `.md` files. GFM is a strict superset of CommonMark. GFM spec for reference only. Do not download GFM Spec.

## Preliminaries

- A line ends at a newline (`U+000A`), carriage return (`U+000D`), or end of file. A blank line contains only spaces or tabs.
- Tabs behave as 4-space tab stops for block structure but are not expanded in content.
- Replace `U+0000` with the replacement character `U+FFFD`.

## Leaf Blocks

- **Thematic breaks**: 3+ matching `-`, `_`, or `*` characters on a line with 0–3 spaces indent. No other characters on the line. Can interrupt a paragraph.
- **ATX headings**: 1–6 `#` characters followed by a space or end of line. Optional closing `#` sequence (preceded by a space). 0–3 spaces indent allowed.
- **Setext headings**: Text underlined with `=` (level 1) or `-` (level 2). Cannot interrupt a paragraph — blank line required after a preceding paragraph.
- **Indented code blocks**: Lines indented 4+ spaces. Cannot interrupt a paragraph. Content is literal text, not parsed as Markdown.
- **Fenced code blocks**: Open with 3+ backticks or tildes (do not mix). Closing fence must use same character with at least the same count. Specify language identifier after the opening fence. Content is literal text.
- **HTML blocks**: Seven types defined by start/end tag conditions. Types 1–6 can interrupt paragraphs; type 7 cannot. Content is passed through as raw HTML.
  - Type 1: `<script>`, `<pre>`, or `<style>` (case-insensitive) — ends at matching closing tag.
  - Type 2: `<!--` comment — ends at `-->`.
  - Type 3: `<?` processing instruction — ends at `?>`.
  - Type 4: `<!` + uppercase letter (e.g., `<!DOCTYPE>`) — ends at `>`.
  - Type 5: `<![CDATA[` — ends at `]]>`.
  - Type 6: Block-level HTML tags (`<div>`, `<table>`, `<p>`, `<h1>`–`<h6>`, `<ul>`, `<ol>`, `<section>`, etc.) — ends at a blank line.
  - Type 7: Any other complete open or closing tag on its own line — ends at a blank line. Cannot interrupt a paragraph.
- **Link reference definitions**: `[label]: destination "title"`. Case-insensitive label matching. First definition wins for duplicate labels. Cannot interrupt a paragraph.
- **Paragraphs**: Consecutive non-blank lines not interpretable as other block constructs. Leading spaces up to 3 are stripped.
- **Blank lines**: Ignored between blocks; determine whether a list is tight or loose.
- **Tables** *(extension)*: Header row, delimiter row (`---`, `:---:`, `---:`), zero or more data rows. Delimit cells with `|`. Escape literal pipe as `\|`. Header and delimiter must have matching column count. Broken at first blank line or other block-level structure.

## Container Blocks

- **Block quotes**: Lines prefixed with `>` (optionally followed by a space). Lazy continuation allowed for paragraph text only. A blank line separates consecutive block quotes.
- **List items**: Bullet markers (`-`, `+`, `*`) or ordered markers (1–9 digits + `.` or `)`). Content column determined by marker width + spaces to first non-whitespace. Sublists must be indented to the content column. An ordered list interrupting a paragraph must start with `1`.
- **Task list items** *(extension)*: `- [ ]` (unchecked) or `- [x]` (checked) at the start of a list item paragraph. Space between `-` and `[` is required. May be nested.
- **Lists**: Sequence of same-type list items. Changing bullet character or ordered delimiter starts a new list. A list is loose if any item is separated by a blank line.

## Inlines

- **Backslash escapes**: `\` before any ASCII punctuation character renders the literal character. Not recognized in code spans, code blocks, or autolinks.
- **Entity and numeric character references**: `&amp;`, `&#123;`, `&#x7B;` — valid HTML5 entities. Not recognized in code spans or code blocks. Cannot replace structural characters.
- **Code spans**: Backtick-delimited inline code. Line endings convert to spaces. Leading and trailing space stripped when both present. Backslash escapes are literal inside code spans.
- **Emphasis and strong emphasis**: `*`/`_` for `<em>`, `**`/`__` for `<strong>`. `_` is not allowed for intraword emphasis. Left-flanking / right-flanking delimiter run rules apply. Delimiter run length sum must not be a multiple of 3 when one delimiter can both open and close (unless both lengths are multiples of 3).
- **Strikethrough** *(extension)*: `~~text~~` — one or two tildes. Does not span across paragraphs. Three or more tildes do not create strikethrough.
- **Links**: Inline `[text](url "title")` or reference `[text][label]` / `[text][]` / `[text]`. Link text may contain inlines but not other links. Destination in `<…>` allows spaces. No whitespace between link text and `(` or `[`.
- **Images**: `![alt](src "title")` — same syntax as links prefixed with `!`. Alt text is the plain-string content of the description.
- **Autolinks**: `<URI>` or `<email>` in angle brackets. Scheme must be 2–32 characters starting with an ASCII letter.
- **Autolinks** *(extension)*: Bare `http://`, `https://`, `www.` URLs and bare email addresses auto-link without angle brackets. Trailing punctuation excluded; parentheses balanced.
- **Raw HTML**: Open/close tags, comments (`<!-- -->`), processing instructions (`<? ?>`), declarations (`<!…>`), CDATA (`<![CDATA[…]]>`) are passed through.
- **Disallowed raw HTML** *(extension)*: `<title>`, `<textarea>`, `<style>`, `<xmp>`, `<iframe>`, `<noembed>`, `<noframes>`, `<script>`, `<plaintext>` have their leading `<` replaced with `&lt;`.
- **Hard line breaks**: Two+ trailing spaces or `\` before a line ending. Not recognized in code spans or HTML tags.
- **Soft line breaks**: A line ending not preceded by two+ spaces or `\`. Rendered as a space in browsers.

## Validation Checklist

- [ ] ATX headings use 1–6 `#` followed by a space.
- [ ] Fenced code blocks specify a language identifier and use matching fence characters and counts.
- [ ] Tables include header and delimiter rows with matching column count. Alignment set with `:` in the delimiter.
- [ ] Task list items have a space between `-` and `[ ]` or `[x]`.
- [ ] Emphasis uses `*` for intraword; `_` only at word boundaries.
- [ ] Strikethrough uses exactly `~~` (not 3+ tildes).
- [ ] Links use `[text](url)` or reference syntax with no whitespace before `(` or `[`.
- [ ] No disallowed raw HTML tags (`<script>`, `<style>`, `<title>`, `<textarea>`, `<xmp>`, `<iframe>`, `<noembed>`, `<noframes>`, `<plaintext>`).

---


# Markdown Accessibility Review Guidelines

When reviewing markdown files, check for the following accessibility issues based on GitHub's [5 tips for making your GitHub profile page accessible](https://github.blog/developer-skills/github/5-tips-for-making-your-github-profile-page-accessible/) and the Smashing Magazine article [Improving The Accessibility Of Your Markdown](https://www.smashingmagazine.com/2021/09/improving-accessibility-of-markdown/). Flag violations and suggest fixes with clear explanations of the accessibility impact.

## 1. Descriptive Links

- Flag generic link text such as "click here," "here," "this," "read more," or "link."
- Link text must make sense when read out of context, because assistive technology can present links as an isolated list.
- Flag multiple links on the same page that share identical text but point to different destinations.
- Bare URLs in prose should be converted to descriptive links.

Bad: `Read my blog post [here](https://example.com)`
Good: `Read my blog post "[Crafting an accessible resume](https://example.com)"`

## 2. Image Alternative (alt) Text

- Flag images with empty alt text (e.g., `![](path/to/image.png)`) unless they are explicitly decorative.
- Flag alt text that is a filename (e.g., `img_1234.jpg`) or generic placeholder (e.g., `screenshot`, `image`).
- Alt text should be succinct and descriptive. Include any text visible in the image.
- Use "screenshot of" where relevant, but do not prefix with "image of" since screen readers announce that automatically.
- For complex images (charts, infographics), suggest summarizing the data in alt text and providing longer descriptions via `<details>` tags or linked content.
- When suggesting alt text improvements, present them as recommendations for the author to review. Alt text requires understanding of visual content and context that only the author can properly assess.

## 3. Heading Hierarchy

- There must be only one H1 (`#`) per document, used as the page title. Note: in projects where H1 is auto-generated from front matter, start content at H2.
- Headings must follow a logical hierarchy and never skip levels (e.g., `##` followed by `####` is a violation).
- Flag bold text (`**text**`) used as a visual substitute for a proper heading.
- Proper heading structure allows assistive technology users to navigate by section and helps sighted users scan content.

## 4. Plain Language

- Flag unnecessarily complex or jargon-heavy language that could be simplified.
- Favor short sentences, common words, and active voice.
- Flag long, dense paragraphs that could be broken into smaller sections or lists.
- When describing UI navigation, write actions as sequential steps in plain language first (e.g., "open Settings, then select Preferences"). Use generic, stable labels rather than icon names or visual descriptions.
- A parenthetical visual reference may follow as supplemental context (e.g., "(gear icon > Preferences)"), but never use visual breadcrumb notation or icon names as the sole way to describe a navigation path.
- When suggesting plain language improvements, present them as recommendations for the author to review. Language decisions require understanding of audience, context, and tone.

## 5. Lists and Emoji Usage

### Lists

- Flag emoji or special characters used as bullet points instead of proper markdown list syntax (`-`, `*`, `+`, or `1.`).
- Flag sequential items in plain text that should be structured as a proper list.
- Proper list markup allows screen readers to announce list context (e.g., "item 1 of 3").

### Emoji

- Flag multiple consecutive emoji, which are disruptive to screen reader users since each emoji name is read aloud in full (e.g., "rocket" "sparkles" "fire").
- Flag emoji used to convey meaning that is not also communicated in text.
- Emoji should be used sparingly and thoughtfully.

## 6. Multimedia

- Provide captions for videos and transcripts for recorded audio.
- Do not auto-play audio and video.
- It's recommended that animated images and other animations are paused on page load.

## 7. Other

- Links: Avoid opening links in a new tab or window.
- Bold and Italics: Screen readers often don't announce bold or italic emphasis, so critical information should not rely on this styling alone.
- Tables: Use tables for data only. Do not use tables for page layout. Avoid nested tables. Avoid complex tables as they are difficult to represent in an accessible format in standard Markdown.

## Review Priority

When multiple issues exist, prioritize in this order:

1. Missing or empty alt text on images
2. Skipped heading levels or heading hierarchy issues
3. Non-descriptive link text
4. Emoji used as bullet points or list markers
5. Plain language improvements
6. Multimedia
7. Other

## Review Tone

- Explain the accessibility impact of each issue, specifying which users are affected (e.g., screen reader users, people with cognitive disabilities, non-native speakers).
- Do not remove personality or voice from the writing. Accessibility and engaging content are not mutually exclusive.
- Keep suggestions actionable and specific.

---


# Markdown Content Rules

The following markdown content rules are enforced in the validators:

1. **Headings**: Use appropriate heading levels (H2, H3, etc.) to structure your content. Do not use an H1 heading, as this will be generated based on the title.
2. **Lists**: Use bullet points or numbered lists for lists. Ensure proper indentation and spacing.
3. **Code Blocks**: Use fenced code blocks for code snippets. Specify the language for syntax highlighting.
4. **Links**: Use proper markdown syntax for links. Ensure that links are valid and accessible.
5. **Images**: Use proper markdown syntax for images. Include alt text for accessibility.
6. **Tables**: Use markdown tables for tabular data. Ensure proper formatting and alignment.
7. **Line Length**: Limit line length to 400 characters for readability.
8. **Whitespace**: Use appropriate whitespace to separate sections and improve readability.
9. **Front Matter**: Include YAML front matter at the beginning of the file with required metadata fields.

## Formatting and Structure

Follow these guidelines for formatting and structuring your markdown content:

- **Headings**: Use `##` for H2 and `###` for H3. Ensure that headings are used in a hierarchical manner. Recommend restructuring if content includes H4, and more strongly recommend for H5.
- **Lists**: Use `-` for bullet points and `1.` for numbered lists. Indent nested lists with two spaces.
- **Code Blocks**: Use triple backticks (```) to create fenced code blocks. Specify the language after the opening backticks for syntax highlighting (e.g., `csharp`).
- **Links**: Use `[link text](URL)` for links. Ensure that the link text is descriptive and the URL is valid.
- **Images**: Use `![alt text](image URL)` for images. Include a brief description of the image in the alt text.
- **Tables**: Use `|` to create tables. Ensure that columns are properly aligned and headers are included.
- **Line Length**: Break lines at 80 characters to improve readability. Use soft line breaks for long paragraphs.
- **Whitespace**: Use blank lines to separate sections and improve readability. Avoid excessive whitespace.

## Validation Checklist

Ensure compliance with the following validation requirements:

### Front Matter

- [ ] `post_title`: The title of the post.
- [ ] `author1`: The primary author of the post.
- [ ] `post_slug`: The URL slug for the post.
- [ ] `microsoft_alias`: The Microsoft alias of the author.
- [ ] `featured_image`: The URL of the featured image.
- [ ] `categories`: The categories for the post. These categories must be from the list in /categories.txt.
- [ ] `tags`: The tags for the post.
- [ ] `ai_note`: Indicate if AI was used in the creation of the post.
- [ ] `summary`: A brief summary of the post. Recommend a summary based on the content when possible.
- [ ] `post_date`: The publication date of the post.

### Content and Formatting

- [ ] Content follows the markdown content rules specified above.
- [ ] Content is properly formatted and structured according to the guidelines.
- [ ] Validation tools have been run to check for compliance with the rules and guidelines.