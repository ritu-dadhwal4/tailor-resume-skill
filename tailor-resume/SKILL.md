---
name: tailor-resume
description: Turn the user's resume into a job-specific, ATS-friendly Word resume with colour highlights recruiters can skim - yellow for behaviour proved, green for achievements in numbers, blue for the job's own language. Use when the user wants to tailor, rewrite or create a resume for a job, shares a job link / job-description screenshot / JD PDF, or says "tailor my resume", "resume for this job", "apply to this job".
---

# Tailor resume

Goal: a finished, tailored resume in **under 2 minutes** of the user's time. The user applies to many jobs, so ask little, ask everything at once, and do the work in one pass. Do not deliberate over wording for long; one solid draft, then let the user react.

## Colour codes (fixed)

| Code | Colour | Marks | Example |
|---|---|---|---|
| `{y:...}` | Yellow | Behaviour you proved - leadership, ownership, teamwork, initiative shown as an action | `{y:Mentored 3 junior developers}` |
| `{g:...}` | Green | Achievements in numbers - %, counts, time, money, scale | `cut API response time by {g:25%}` |
| `{b:...}` | Blue | The job's own language - exact skills, tools and phrases from the job description | `{b:deployed on AWS with Kubernetes}` |

The highlighted file IS the file the user sends; the colours exist so recruiters can skim. No legend by default (a legend saying "the job's language" tells the recruiter keywords were copied). Add `--legend` only if the user asks.

## Step 1 - Master resume (once)

Look for `~/.tailor-resume/master.md`.

- **Found:** use it. Do not ask for the resume again unless the user says it changed or attaches a new one.
- **Not found:** ask for their current resume (PDF, Word or screenshots). Read PDFs and images directly; for `.docx` run `python3 <skill>/scripts/read_resume.py <file>`. Save the full content, faithfully and completely (every role, date, bullet, number, skill, link), to `~/.tailor-resume/master.md`. This is the only source of facts from now on.

If the user attaches a new resume later, replace master.md with it.

## Step 2 - Job description

Accept any of: a link, screenshots, a PDF, or pasted text.

- **Link:** fetch it once. If the page is behind a login, blocked, empty, or clearly not the full JD (LinkedIn, Naukri, Indeed, Workday often are), **stop and ask for screenshots of the job description**. Do not retry, use other scrapers, or try workarounds.
- **Screenshots / PDF:** read them directly.

Pull out: company, role title, location, must-have skills, nice-to-haves, repeated phrases, seniority.

## Step 3 - Ask once (one AskUserQuestion call, two questions)

1. **Highlights** (multiSelect): "Which highlights do you want?" options: `Yellow - behaviour you proved`, `Green - achievements in numbers`, `Blue - the job's language`. Mention all three is the default.
2. **Length**: "One page or two?" Put the recommended option first with "(Recommended)". Recommend **1 page** unless the master resume has 8+ years of relevant experience or the content genuinely cannot fit without losing strong, relevant material - then recommend 2.

Ask this in the same turn the JD is received (fetch/read the JD in parallel when possible). If `~/.tailor-resume/prefs.json` exists, put the saved choice first in each question, labelled "(last time)". Save the answers to prefs.json afterwards.

Paper size: infer, do not ask. US/Canada jobs → `--paper letter`; everything else → `--paper a4`.

## Step 4 - Write the resume spec

Write `~/Documents/Tailored Resumes/<First>-<Last>-<Company>-<Role>.json` (create the folder; slug-safe names), following `<skill>/examples/sample-spec.json`. Shape:

```json
{
  "name": "...", "contact": ["email", "phone", "city, country", "linkedin/portfolio"],
  "summary": "...",
  "skills": [{"label": "Languages", "items": "..."}],
  "experience": [{"company": "...", "title": "...", "location": "...", "dates": "...", "bullets": ["..."]}],
  "projects": [{"name": "...", "text": "..."}],
  "education": [{"school": "...", "degree": "...", "location": "...", "dates": "...", "bullets": []}],
  "certifications": ["..."],
  "extra_sections": [{"heading": "Awards", "bullets": ["..."]}],
  "section_order": ["summary", "skills", "experience", "projects", "education", "certifications", "extra"]
}
```

Leave out any section with nothing strong in it. For freshers, put education and projects before experience.

### Writing rules

**Truth first**
- Never invent employers, titles, dates, tools, numbers or achievements. Every fact comes from master.md.
- Rephrasing toward the job's wording is fine only when it describes the same real work. A JD skill the user does not have stays out (list it as a gap in the review).
- Keep real job titles. Highlight the title in blue only if it genuinely matches the JD's title.
- If a strong bullet has no number, you may leave a placeholder like `{g:[X%]}` or `{g:[N] users}` - at most 3 per resume - and tell the user to fill them.

**Bullets**
- Pattern: action verb + what you did + result. One line where possible, never more than two.
- Most recent / relevant role: 3-5 bullets. Older roles: 1-3. Roles older than ~10 years or irrelevant: one line or drop.
- Reorder bullets so the ones matching the JD come first. Reorder skills so JD skills lead; drop skills irrelevant to this job.
- Summary: 2 lines max, mirrors the role title and 2-3 top JD requirements.

**Highlights - light, so it stays easy to skim**
- Highlight short phrases (2-6 words), never whole sentences.
- At most 2 highlights per bullet; around 60-70% of bullets carry at least one; some bullets stay plain.
- Blue goes on exact JD terms (tool names, the JD's phrasing), not generic words.
- Yellow goes on a demonstrated action ("Led a team of 4"), never on adjectives ("hard-working").
- Green wraps the number with its unit ("40% fewer bugs", "2x traffic"), not the bare digit alone.
- Always mark all three codes in the spec; the script drops the colours the user turned off.

**Not crowded**
- White space beats squeezing. If over length, cut the weakest bullets first, then old roles, then skills lines - never shrink the font or margins.

## Step 5 - Build

```bash
python3 <skill>/scripts/build_resume.py "<spec>.json" "<spec>.docx" --highlights y,g,b --pages 1 --paper a4
```

`--highlights` takes the chosen codes (`y`, `g`, `b`, comma separated). If python-docx is missing, run `python3 -m pip install --user python-docx` first.

Read the script's output:
- `WARNING: likely over N page(s)` → cut lines as told and rebuild. Do this silently, do not ask.
- `PLACEHOLDERS TO FILL` → mention them in the review.

## Step 6 - Review (short)

On macOS make a preview image and show it: `qlmanage -t -s 1400 -o "<folder>" "<file>.docx"` then read the PNG. (The preview ignores right-aligned dates and some fonts; Word and Google Docs show them correctly.)

Then reply in no more than ~8 lines:
- file path
- JD keywords covered (blue) and **gaps** - JD must-haves not in the resume
- placeholders to fill, if any
- "Reply with changes, or say **open** to edit in Google Docs."

Apply changes by editing the JSON and rebuilding - never hand-edit the docx.

## Step 7 - Open in Google Docs

When the user approves:

1. `open -R "<file>.docx"` (reveals it in Finder) and `open "https://drive.google.com/drive/my-drive"`.
2. Tell them: drag the file into Drive → double-click it → **Open with Google Docs**. Edit there, then **File → Download → Microsoft Word (.docx)** or PDF to apply.

On Windows use `explorer /select,"<file>"` and `start https://drive.google.com/drive/my-drive`; on Linux `xdg-open` the folder and the URL.
