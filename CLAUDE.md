# Claude Code Instructions for Digital Sovereignty Paper

## Project Overview

This is a **documentation repository** containing a comprehensive opinion paper on digital sovereignty, supply chain security, and cloud independence with Red Hat's enterprise-grade open source solutions.

**Type:** Technical documentation (Markdown)
**Target Audiences:** Executives, technical architects, engineering managers, developers
**Format:** Modular sections with executive summaries and technical deep dives

## Repository Structure

```
sovereignty-paper/
├── README.md                    # Project introduction and navigation
├── docs/
│   ├── OUTLINE.md              # Complete table of contents
│   ├── INDEX.md                # Navigation index
│   ├── WRITING_GUIDE.md        # **CRITICAL**: Writing standards and style guide
│   └── sections/               # Individual section files
│       ├── 00-executive-overview.md
│       ├── 01-introduction.md
│       ├── 02-base-images.md
│       ├── 03-supply-chain.md
│       ├── 04-kubernetes-platform.md
│       ├── 05-cicd.md
│       ├── 06-developer-experience.md
│       ├── 07-platform-engineering.md
│       ├── 08-confidential-containers.md
│       ├── 09-workload-identity.md
│       ├── 10-ai-platform.md
│       └── 11-conclusion.md
└── CLAUDE.md                   # This file
```

## Critical Workflow Rule: Branch-Per-Edit

**IMPORTANT:** Each major edit must be done in its own git branch.

### When to Create a Branch

Create a new branch for:
- Adding a new section
- Major revisions to an existing section
- Restructuring content
- Adding new subsections or significant content

### Branch Naming Convention

Use descriptive branch names:
```bash
# Good examples:
git checkout -b edit/add-supply-chain-examples
git checkout -b edit/revise-exec-summary
git checkout -b edit/update-kubernetes-section
git checkout -b feature/add-diagrams
git checkout -b fix/correct-technical-details

# Pattern: [type]/[brief-description]
# Types: edit, feature, fix, refactor
```

### Standard Workflow

1. **Before starting work:** Create and checkout a new branch
   ```bash
   git checkout -b edit/descriptive-name
   ```

2. **Make changes:** Edit the relevant files

3. **Commit changes:** Use descriptive commit messages
   ```bash
   git add <files>
   git commit -m "Brief description of changes"
   ```

4. **Push to remote:** (if working with remote repository)
   ```bash
   git push -u origin edit/descriptive-name
   ```

5. **User will merge:** The user will review and merge branches as appropriate

## Writing Standards

### MUST READ: WRITING_GUIDE.md

**Before editing any content, always read:** [docs/WRITING_GUIDE.md](docs/WRITING_GUIDE.md)

The writing guide is comprehensive and mandatory. Key points:

#### Section Structure (Required)

Every section must include:
1. **Executive Summary** (≤300 words)
2. **Introduction/Context**
3. **Technical Deep Dive** (with code examples)
4. **Use Cases / Practical Applications**
5. **The Upstream Connection** (Red Hat's contributions)
6. **Key Benefits Summary** (Technical Teams / Organizations / Digital Sovereignty)
7. **References and Further Reading**

#### Style Guidelines

**Do:**
- Use active voice
- Be specific with metrics ("80% faster" not "significantly faster")
- Include code examples with comments
- Define technical terms on first use
- Link to upstream projects and documentation
- Connect everything back to digital sovereignty themes

**Don't:**
- Use marketing hyperbole ("revolutionary", "game-changing")
- Make unfounded claims
- Use jargon without explanation
- Bash competitors
- Create content without evidence

#### Code Examples

All code examples must:
- Specify language in code fences (```yaml not ```)
- Include explanatory comments
- Have context before and explanation after
- Be realistic and tested

#### Digital Sovereignty Connection

Every section must address at least one of:
1. **Freedom from Lock-in** - Cloud portability, vendor independence
2. **Transparency and Control** - Open source visibility, audit ability
3. **Regulatory Compliance** - Data sovereignty, industry regulations
4. **Strategic Independence** - Geopolitical risk reduction, technology autonomy

## Working with This Repository

### When Editing Content

1. **Always create a branch first** (see workflow above)
2. **Read the section** you're editing completely before making changes
3. **Consult WRITING_GUIDE.md** for structure and style requirements
4. **Check related sections** for consistency and cross-references
5. **Validate any code examples** you add or modify
6. **Update cross-references** if you change section organization
7. **Run the review checklist** (see WRITING_GUIDE.md line 394)

### When Adding New Sections

1. **Create branch:** `git checkout -b edit/add-[section-name]`
2. **Create section file:** `docs/sections/##-section-name.md`
3. **Follow the section structure template** from WRITING_GUIDE.md (lines 10-96)
4. **Update OUTLINE.md** to include the new section
5. **Update README.md** if needed for navigation
6. **Commit and push** the branch

### When Making Major Revisions

1. **Create branch:** `git checkout -b edit/revise-[section-name]`
2. **Read existing content** fully
3. **Check for cross-references** from other sections
4. **Make revisions** following writing guide
5. **Update any affected cross-references** in other sections
6. **Run review checklist** before committing

## File Linking Conventions

### Internal Links (within docs/)

```markdown
# From one section to another:
As discussed in [Supply Chain Security](03-supply-chain.md)...

# To specific subsection:
See [Confidential Containers](08-confidential-containers.md#use-cases)

# From sections/ to parent docs/:
See the [Writing Guide](../WRITING_GUIDE.md)
```

### External Links

```markdown
- [Project Name](https://url.com) - Brief description
- [Specific Documentation](https://url.com/docs/feature)
```

## Review Checklist (from WRITING_GUIDE.md)

Before completing any section edit:

- [ ] Executive summary under 300 words, non-technical
- [ ] Technical deep dive includes code/config examples
- [ ] Real-world use case included
- [ ] "Upstream Connection" section complete
- [ ] "Key Benefits Summary" follows standard format
- [ ] At least 3 external references/links
- [ ] Digital sovereignty connection is explicit
- [ ] No marketing hyperbole
- [ ] All code examples have comments
- [ ] Technical terms defined on first use
- [ ] All claims backed by evidence

## Common Tasks

### Task: Add a code example to a section

```bash
# 1. Create branch
git checkout -b edit/add-code-example-[section]

# 2. Read the section file
# 3. Add code example with:
#    - Context before (what problem it solves)
#    - Code block with language specified and comments
#    - Explanation after (key elements, best practices)

# 4. Commit
git add docs/sections/[section].md
git commit -m "Add [description] code example to [section] section"
```

### Task: Update executive summary

```bash
# 1. Create branch
git checkout -b edit/update-exec-summary-[section]

# 2. Read current summary and full section
# 3. Revise to ≤300 words, ensuring:
#    - What is this about? (1 sentence)
#    - Why does it matter? (2-3 sentences)
#    - Key takeaway (1-2 sentences)

# 4. Commit
git add docs/sections/[section].md
git commit -m "Update executive summary for [section] section"
```

### Task: Add upstream contribution details

```bash
# 1. Create branch
git checkout -b edit/add-upstream-info-[section]

# 2. Research Red Hat's contributions to relevant projects
# 3. Add "The Upstream Connection" section with:
#    - Specific contributions (quantified)
#    - Leadership roles
#    - Engineering investment
#    - Community benefits

# 4. Commit
git add docs/sections/[section].md
git commit -m "Add upstream contribution details to [section] section"
```

## Key Themes to Emphasize

Throughout all content:

1. **Open Source First** - Every solution anchored in upstream projects
2. **Global IP Freedom** - Freedom from geographical and vendor restrictions
3. **Enterprise Grade** - Difference between community and production-ready
4. **Upstream Contributions** - Red Hat's engineering investment
5. **Digital Sovereignty** - Strategic independence and control

## Red Hat Positioning

When discussing Red Hat:
- Emphasize upstream-first approach
- Quantify contributions (maintainers, commits, leadership)
- Show engineering investment (FTEs, projects)
- Connect community benefits to business value
- Balance advocacy with objectivity
- Show trade-offs honestly

## Questions to Ask Before Editing

1. **So what?** - Why should the reader care?
2. **How does this enable sovereignty?** - Connect to core theme
3. **What can readers do with this information?** - Make it actionable
4. **Is this true?** - Can I back this up with evidence?
5. **Would a non-expert understand the executive summary?** - Test it
6. **Would an expert find the technical content valuable?** - Don't oversimplify

## Getting Unstuck

If struggling with a section:
1. Start with a real-world example first
2. Check Red Hat blogs for similar explanations
3. Look at upstream project documentation
4. Ask "why" five times to get to root value
5. Simplify first, then add complexity

## Important Notes

- **This is an opinion paper** - It should educate and persuade, not just describe
- **Multiple audiences** - Executive summaries for leaders, technical depth for engineers
- **Consistency matters** - Check other sections for terminology and style
- **Evidence required** - All claims need backing (metrics, examples, sources)
- **Branch per edit** - Never work directly on main branch for major edits

## Memory and Learning

As you work with this repository:
- Record common patterns in memory (e.g., "Red Hat contribution sections typically include...")
- Note terminology conventions (e.g., "OpenShift not 'OCP'", "Kubernetes not 'K8s' in formal text")
- Track which sections reference each other
- Remember the balance between executive and technical content

## Quick Reference

- **Main branch:** `main`
- **Writing guide:** [docs/WRITING_GUIDE.md](docs/WRITING_GUIDE.md)
- **Section template:** WRITING_GUIDE.md lines 10-96
- **Review checklist:** WRITING_GUIDE.md lines 394-410
- **Style guidelines:** WRITING_GUIDE.md lines 100-158
- **Workflow:** Always branch, then edit, then commit

---

**Remember:** This paper represents Red Hat's technical leadership and upstream contributions. Quality, accuracy, and clear communication are paramount. When in doubt, read WRITING_GUIDE.md and check existing sections for patterns.
