# Dynamic Project Summaries Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Transform projects page to display GitHub README summaries via GitHub Actions and Hugo data files.

**Architecture:** GitHub Action fetches README.md files from repos listed in front matter, extracts first paragraph, saves to data/projects.json. Hugo layout reads JSON and renders project cards with summaries.

**Tech Stack:** Hugo (static site generator), GitHub Actions (automation), Go templates (Hugo layouts), YAML (front matter)

---

## Task 1: Update Projects Index with Front Matter

**Files:**
- Modify: `content/projects/_index.md`

**Step 1: Read current content**

Run: `cat content/projects/_index.md`
Expected: Current simple list with links

**Step 2: Update front matter and content**

Replace entire file with:

```markdown
---
title: "Projects"
type: page
projects:
  - name: "TuWire - Juice Jack Apple Devices"
    repo: "whati001/tuwire"
  - name: "Morgeb - Wordclock replica"
    repo: "whati001/morgeb"
  - name: "Tuff-Tuff-Light - Wireless Trailer Light"
    repo: "whati001/tuff-tuff-light"
  - name: "Keksbox - NFC Tag Music Box"
    repo: "whati001/keksbox"
  - name: "CoffeeRat - SmartScale"
    url: "https://www.youtube.com/watch?v=EL8P_DzBC0U"
    description: "Smart scale for coffee brewing with real-time weight tracking"
  - name: "Stmk. Fischerprüfung Onlinefragenkatalog"
    url: "https://fisch.rehka.dev/"
    description: "Online quiz platform for Styrian fishing license exam"
  - name: "SipLine - VoIP sniffer"
    repo: "whati001/sipline"
---

This page displays my projects with automatically fetched summaries from GitHub.
```

**Step 3: Verify file updated**

Run: `cat content/projects/_index.md | grep -A 5 "projects:"`
Expected: See projects list in YAML front matter

**Step 4: Commit**

```bash
git add content/projects/_index.md
git commit -m "feat: add project metadata to front matter"
```

---

## Task 2: Create GitHub Action Workflow

**Files:**
- Create: `.github/workflows/fetch-projects.yml`

**Step 1: Create workflows directory**

Run: `mkdir -p .github/workflows`
Expected: Directory created

**Step 2: Write GitHub Action workflow**

Create `.github/workflows/fetch-projects.yml`:

```yaml
name: Fetch Project Summaries

on:
  workflow_dispatch:
  push:
    branches:
      - main
    paths:
      - 'content/projects/_index.md'

jobs:
  fetch-summaries:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install PyYAML requests

      - name: Fetch project summaries
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          python << 'EOF'
          import yaml
          import json
          import requests
          import os
          import re
          from datetime import datetime

          # Read front matter from _index.md
          with open('content/projects/_index.md', 'r') as f:
              content = f.read()

          # Extract YAML front matter
          match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
          if not match:
              print("Error: No front matter found")
              exit(1)

          front_matter = yaml.safe_load(match.group(1))
          projects_config = front_matter.get('projects', [])

          # Fetch summaries
          projects_data = []
          headers = {'Authorization': f"token {os.environ['GITHUB_TOKEN']}"}

          for project in projects_config:
              project_entry = {
                  'name': project.get('name', 'Unnamed Project'),
                  'url': project.get('url', ''),
                  'summary': project.get('description', '')
              }

              # If repo specified, fetch README
              if 'repo' in project:
                  repo = project['repo']
                  project_entry['repo'] = repo

                  # Default to GitHub URL if not specified
                  if not project_entry['url']:
                      project_entry['url'] = f"https://github.com/{repo}"

                  # Fetch README from GitHub API
                  api_url = f"https://api.github.com/repos/{repo}/readme"
                  try:
                      response = requests.get(api_url, headers=headers)
                      if response.status_code == 200:
                          readme_data = response.json()
                          # Get download URL for raw content
                          download_url = readme_data.get('download_url')
                          if download_url:
                              readme_response = requests.get(download_url)
                              if readme_response.status_code == 200:
                                  readme_content = readme_response.text

                                  # Extract first paragraph
                                  # Remove front matter if present
                                  readme_content = re.sub(r'^---\n.*?\n---\n', '', readme_content, flags=re.DOTALL)
                                  # Remove leading/trailing whitespace
                                  readme_content = readme_content.strip()
                                  # Split by double newline or heading
                                  paragraphs = re.split(r'\n\n+|\n#+\s', readme_content)
                                  # Find first non-empty paragraph
                                  for para in paragraphs:
                                      para = para.strip()
                                      # Skip if it's just a heading, image, or link
                                      if para and not para.startswith('#') and not para.startswith('!') and not para.startswith('['):
                                          project_entry['summary'] = para
                                          break

                                  print(f"✓ Fetched summary for {project['name']}")
                      else:
                          print(f"⚠ Could not fetch README for {repo}: {response.status_code}")
                  except Exception as e:
                      print(f"⚠ Error fetching {repo}: {str(e)}")

              # Add timestamp
              project_entry['fetched_at'] = datetime.utcnow().isoformat() + 'Z'
              projects_data.append(project_entry)

          # Write to data file
          os.makedirs('data', exist_ok=True)
          with open('data/projects.json', 'w') as f:
              json.dump({'projects': projects_data}, f, indent=2)

          print(f"\n✓ Generated data/projects.json with {len(projects_data)} projects")
          EOF

      - name: Commit changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/projects.json
          if git diff --staged --quiet; then
            echo "No changes to commit"
          else
            git commit -m "chore: update project summaries

          🤖 Generated with GitHub Actions"
            git push
          fi
```

**Step 3: Verify workflow file created**

Run: `cat .github/workflows/fetch-projects.yml | head -20`
Expected: See workflow YAML content

**Step 4: Commit**

```bash
git add .github/workflows/fetch-projects.yml
git commit -m "feat: add GitHub Action to fetch project summaries"
```

---

## Task 3: Create Hugo Layout for Projects Page

**Files:**
- Create: `layouts/projects/list.html`

**Step 1: Create layouts directory**

Run: `mkdir -p layouts/projects`
Expected: Directory created

**Step 2: Write Hugo layout template**

Create `layouts/projects/list.html`:

```html
{{ define "main" }}
<div class="post-title">
  <h1>{{ .Title }}</h1>
</div>

{{ if .Content }}
<div class="post-content">
  {{ .Content }}
</div>
{{ end }}

{{ if .Site.Data.projects }}
<style>
.projects-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.5rem;
  margin-top: 2rem;
}

.project-card {
  border: 1px solid var(--border-color, #e1e4e8);
  border-radius: 8px;
  padding: 1.5rem;
  background: var(--card-background, #ffffff);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.project-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.project-card h3 {
  margin-top: 0;
  margin-bottom: 0.75rem;
  font-size: 1.25rem;
}

.project-card h3 a {
  text-decoration: none;
  color: var(--heading-color, #24292e);
}

.project-card h3 a:hover {
  color: var(--link-color, #0366d6);
}

.project-summary {
  color: var(--text-muted, #586069);
  line-height: 1.6;
  margin-bottom: 1rem;
  font-size: 0.95rem;
}

.project-link {
  display: inline-block;
  font-size: 0.9rem;
  color: var(--link-color, #0366d6);
  text-decoration: none;
  font-weight: 500;
}

.project-link:hover {
  text-decoration: underline;
}

.no-summary {
  color: var(--text-muted, #586069);
  font-style: italic;
}

/* Dark mode support */
@media (prefers-color-scheme: dark) {
  .project-card {
    --border-color: #30363d;
    --card-background: #161b22;
    --heading-color: #c9d1d9;
    --text-muted: #8b949e;
    --link-color: #58a6ff;
  }
}

/* Responsive: single column on mobile */
@media (max-width: 768px) {
  .projects-grid {
    grid-template-columns: 1fr;
  }
}
</style>

<div class="projects-grid">
  {{ range .Site.Data.projects.projects }}
  <div class="project-card">
    <h3>
      <a href="{{ .url }}" target="_blank" rel="noopener noreferrer">
        {{ .name }}
      </a>
    </h3>

    {{ if .summary }}
    <div class="project-summary">
      {{ .summary }}
    </div>
    {{ else }}
    <div class="project-summary no-summary">
      No description available
    </div>
    {{ end }}

    {{ if .repo }}
    <a href="{{ .url }}" class="project-link" target="_blank" rel="noopener noreferrer">
      View on GitHub →
    </a>
    {{ else }}
    <a href="{{ .url }}" class="project-link" target="_blank" rel="noopener noreferrer">
      Visit Project →
    </a>
    {{ end }}
  </div>
  {{ end }}
</div>

{{ else }}
<div class="post-content">
  <p><em>No project data available. Run the GitHub Action to fetch project summaries.</em></p>
</div>
{{ end }}

{{ end }}
```

**Step 3: Verify layout file created**

Run: `cat layouts/projects/list.html | head -20`
Expected: See Hugo template content

**Step 4: Commit**

```bash
git add layouts/projects/list.html
git commit -m "feat: add custom Hugo layout for projects page with card grid"
```

---

## Task 4: Create Initial Data File (Manual)

**Files:**
- Create: `data/projects.json`

**Step 1: Create data directory**

Run: `mkdir -p data`
Expected: Directory created

**Step 2: Create placeholder data file**

This will be replaced by GitHub Action, but having it allows Hugo to build:

Create `data/projects.json`:

```json
{
  "projects": []
}
```

**Step 3: Verify file created**

Run: `cat data/projects.json`
Expected: Empty projects array

**Step 4: Commit**

```bash
git add data/projects.json
git commit -m "feat: add initial projects data file"
```

---

## Task 5: Test Hugo Build Locally

**Files:**
- None (testing only)

**Step 1: Run Hugo build**

Run: `hugo --quiet`
Expected: Build completes without errors

**Step 2: Start Hugo server**

Run: `hugo server` (in background or separate terminal)
Expected: Server starts on localhost:1313

**Step 3: Verify projects page renders**

Check output: Look for "Web Server is available at http://localhost:1313/"
Manual check: Open http://localhost:1313/projects/ in browser
Expected: See "No project data available" message (data file is empty)

**Step 4: Stop Hugo server**

Run: Press Ctrl+C
Expected: Server stops

**Step 5: Document test results**

No commit needed - this is verification only

---

## Task 6: Push Feature Branch and Create Pull Request

**Files:**
- None (git operations only)

**Step 1: Push feature branch**

Run: `git push -u origin feature/dynamic-project-summaries`
Expected: Branch pushed to remote

**Step 2: Create pull request**

Run:
```bash
gh pr create --title "feat: add dynamic project summaries from GitHub" --body "$(cat <<'EOF'
## Summary
- Add front matter to projects index with repo metadata
- Create GitHub Action to fetch README summaries
- Create custom Hugo layout with card grid design
- Initial data file for Hugo builds

## Test Plan
- [x] Hugo builds successfully
- [ ] GitHub Action runs and generates data/projects.json
- [ ] Projects page displays cards with summaries
- [ ] Responsive design works on mobile/tablet/desktop
- [ ] Dark mode styling works correctly

## How to Test
1. Merge this PR
2. Manually trigger "Fetch Project Summaries" workflow in Actions tab
3. Wait for workflow to complete
4. Check data/projects.json has been updated
5. Redeploy site and verify projects page displays summaries

🤖 Generated with Claude Code
EOF
)"
```
Expected: PR created with URL

**Step 3: Document PR URL**

The PR URL will be displayed - save it for reference

---

## Task 7: Post-Merge Testing (After PR is Merged)

**Files:**
- None (testing only)

**Step 1: Merge pull request**

Manual: Merge PR via GitHub UI or:
Run: `gh pr merge --merge`
Expected: PR merged to main

**Step 2: Trigger GitHub Action manually**

Manual: Go to Actions tab > "Fetch Project Summaries" > Run workflow
Or run: `gh workflow run fetch-projects.yml`
Expected: Workflow starts

**Step 3: Wait for workflow completion**

Run: `gh run list --workflow=fetch-projects.yml --limit 1`
Expected: See completed status

**Step 4: Verify data file updated**

Run: `git pull && cat data/projects.json | jq '.projects | length'`
Expected: Number > 0 (projects with summaries)

**Step 5: Check summaries are present**

Run: `cat data/projects.json | jq '.projects[0]'`
Expected: See project with name, url, summary, fetched_at

**Step 6: Deploy site and verify**

Manual: Deploy site (depends on deployment method)
Expected: Site rebuilds with new data

**Step 7: Visual verification**

Manual: Visit https://whati001.rehka.dev/projects/
Expected:
- See project cards in grid layout
- Each card has title, summary, and link
- Responsive layout works
- Dark mode works

---

## Success Criteria

- [ ] Projects page displays cards instead of simple list
- [ ] GitHub Action successfully fetches README summaries
- [ ] First paragraph extracted correctly from each README
- [ ] data/projects.json contains valid summaries
- [ ] Card grid is responsive (1/2/3 columns based on screen size)
- [ ] Dark mode styling works
- [ ] Projects without repos still display (with custom URL/description)
- [ ] Hugo builds without errors
- [ ] No JavaScript required on frontend

## Rollback Plan

If issues occur:
1. Revert PR: `git revert <commit-sha>`
2. Delete workflow: Remove `.github/workflows/fetch-projects.yml`
3. Delete layout: Remove `layouts/projects/list.html`
4. Restore original `content/projects/_index.md`
5. Site falls back to theme's default list template
