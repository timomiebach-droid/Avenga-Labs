# Zapier Automation Setup
## Forward/Research — Weekly Auto-Publishing Pipeline

---

## Overview

Every Monday, Zapier:
1. Reads a research doc from Google Docs
2. Sends it to Claude (Anthropic API) to generate a full article
3. Pushes the article HTML to GitHub
4. Updates articles.json so the index page reflects the new article

No server. No Supabase. Just files.

---

## Zap Structure

### Step 1 — Trigger: Schedule by Zapier
- Frequency: Every week
- Day: Monday
- Time: 09:00 AM

---

### Step 2 — Google Docs: Get Document Content
- Action: Get Document Content
- Document: [your weekly research Google Doc]
- Output variable name: `research_content`

---

### Step 3 — Formatter by Zapier: Date/Time
- Action: Format Date
- Input: Today's date
- Format: YYYY-MM-DD
- Output variable name: `date_slug`  (e.g. 2026-03-09)

---

### Step 4 — Claude AI (Anthropic): Generate Article
- Model: claude-opus-4-6  (or claude-sonnet-4-6 for speed/cost)
- System Prompt: [copy from SYSTEM PROMPT section below]
- Message: [copy from USER MESSAGE TEMPLATE section below]
  - Replace {{RESEARCH_CONTENT}} with Step 2 output
  - Replace {{DATE}} with Step 3 output
- Max Tokens: 8000
- Output variable name: `article_html`

---

### Step 5 — Code by Zapier: Build Metadata
- Language: JavaScript
- Input data:
  - `html`: the `article_html` from Step 4
  - `date`: the `date_slug` from Step 3
- Code:
```javascript
// Extract metadata Claude embedded in the HTML
const titleMatch = inputData.html.match(/<title>(.*?)<\/title>/);
const h1Match = inputData.html.match(/<h1[^>]*>([\s\S]*?)<\/h1>/);
const subMatch = inputData.html.match(/class="sub[^"]*"[^>]*>([\s\S]*?)<\/p>/);

// Strip HTML tags from matches
const stripTags = s => s ? s.replace(/<[^>]+>/g, '').trim() : '';

const title = stripTags(h1Match?.[1] || titleMatch?.[1] || 'Untitled');
const subtitle = stripTags(subMatch?.[1] || '');

// Build slug from date + first words of title
const words = title.toLowerCase().replace(/[^a-z0-9\s]/g,'').split(/\s+/).slice(0,5).join('-');
const slug = `${inputData.date}-${words}`;

// Count sources in HTML
const sourcesCount = (inputData.html.match(/<li id="fn/g) || []).length;

output = { slug, title, subtitle, slug };
```
- Output variables: `slug`, `title`, `subtitle`

---

### Step 6 — GitHub: Create File (New Article)
- Action: Create File
- Repo: your-username/forward-research
- File Path: `articles/{{slug}}.html`  (use slug from Step 5)
- File Content: `{{article_html}}`  (use output from Step 4)
- Commit Message: `feat: add article {{slug}}`

---

### Step 7 — GitHub: Get File (Read current articles.json)
- Action: Get File Contents
- Repo: your-username/forward-research
- File Path: `articles.json`
- Output variable name: `current_json`

---

### Step 8 — Code by Zapier: Update articles.json
- Language: JavaScript
- Input data:
  - `current`: `current_json` from Step 7
  - `slug`: from Step 5
  - `title`: from Step 5
  - `subtitle`: from Step 5
  - `date`: from Step 3
- Code:
```javascript
const articles = JSON.parse(inputData.current);

// Format date as "March 9, 2026"
const d = new Date(inputData.date + 'T12:00:00');
const dateFormatted = d.toLocaleDateString('en-GB', {
  day: 'numeric', month: 'long', year: 'numeric'
});

// Prepend new article (newest first)
const newEntry = {
  slug: inputData.slug,
  title: inputData.title,
  subtitle: inputData.subtitle,
  date: inputData.date,
  dateFormatted: dateFormatted,
  readTime: "12 min",
  sources: 7,
  tags: ["AI", "Business Strategy"]
};

articles.unshift(newEntry);
output = { updated_json: JSON.stringify(articles, null, 2) };
```
- Output variable: `updated_json`

---

### Step 9 — GitHub: Update File (articles.json)
- Action: Update File
- Repo: your-username/forward-research
- File Path: `articles.json`
- File Content: `{{updated_json}}`  (from Step 8)
- Commit Message: `chore: update articles.json — add {{slug}}`

---

## SYSTEM PROMPT
*(paste this into Zapier's Claude "System Prompt" field)*

```
You are an expert business journalist and frontend developer. Your task is to transform raw business research into a polished, editorial-quality article published as a complete, self-contained HTML file.

WRITING STYLE:
- Write in flowing editorial prose, not bullet points or lists
- Open with a compelling lead paragraph that hooks the reader
- Structure content into 5–6 named sections (numbered 01–06)
- Extract key statistics and present them as visual callouts
- Include 2 pull quotes that capture the most striking insights
- End with a strong conclusion that has a clear point of view
- Cite all claims with inline superscript footnotes [1], [2] etc.
- Write at the level of The Economist or Harvard Business Review

HTML OUTPUT RULES:
- Output ONLY the raw HTML file, nothing else
- No markdown, no code fences, no explanation before or after
- Use the exact CSS classes from the template provided
- Keep all <style> and <script> blocks exactly as in the template
- Only change: title, h1, article content, sources list, articles.json metadata
- The slug format is: YYYY-MM-DD-short-title (e.g. 2026-03-09-ai-agent-economy)
```

---

## USER MESSAGE TEMPLATE
*(paste this into Zapier's Claude "Message" field, with Zapier variables substituted)*

```
Date: {{date_slug}}

Here is the research content to transform into a published article:

---
{{research_content}}
---

Now generate a complete HTML article page using the template below.
Replace ONLY the content sections. Keep all CSS, JS, and structure identical.

TEMPLATE:
[paste the full contents of article-template.html here]
```

---

## Notes

- The template file `article-template.html` contains placeholder comments
  Claude replaces only those sections
- GitHub Pages redeploys automatically within ~60 seconds of each push
- To test without waiting a week: run the Zap manually in Zapier
- To change the publishing schedule: edit Step 1
