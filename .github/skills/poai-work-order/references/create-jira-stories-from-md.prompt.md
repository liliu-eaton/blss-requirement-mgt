---
agent: "agent"
description: "Create Jira stories from markdown story definitions under a specified Epic, mapping fields per BLSS conventions."
---

# Create Jira Stories from Markdown Story Definitions

## Purpose

Read user story definitions from a markdown file and create corresponding Jira Story issues under a specified parent Epic, mapping structured sections to specific Jira fields.

## Input Required

1. **Source file**: Path to the markdown file containing story definitions (e.g., `Requirements/BDCSPM-62315/BDCSPM-75199-Stories.md`)
2. **Parent Epic key**: The Jira Epic key to link stories under (e.g., `BDCSPM-75199`)
3. **Project key**: Jira project key (default: `BDCSPM`)
4. **Cloud ID**: Atlassian site URL (default: `https://eaton-corp.atlassian.net`)

## Field Mapping Rules

For each story in the markdown file, map sections to Jira fields as follows:

| Markdown Section | Jira Field | Format |
|-----------------|------------|--------|
| Story Title (from `### STORY:` heading) | Summary | Plain text, exclude the Story ID prefix |
| **Description** (As a / I want / So that) + **Context & notes** + **Tasks** | Description | Markdown — see template below |
| **Acceptance Criteria** (AC1, AC2, ...) | Acceptance Criteria (`customfield_12200`) | ADF — see template below |
| **Definition of Ready checklist** | Checklists (`customfield_23603`) | Checklists format — see template below |

## Jira Description Field Template

Combine the "Description", "Context & notes", and "Tasks" sections into the Jira Description field using this format:

```markdown
As a [persona/role],
I want to [action/capability],
So that [business value / outcome].

### Context & Notes

- [bullet point 1]
- [bullet point 2]
- ...

### Tasks

- [ ] Task 1: [description]
- [ ] Task 2: [description]
- ...
```

## Jira Acceptance Criteria Field Template

Map the Given/When/Then acceptance criteria into the Jira Acceptance Criteria field (`customfield_12200`) as Atlassian Document Format (ADF), not a plain markdown string.

Each AC must be one bullet-list item with this visual format:

- `ACx:` is bold, uses the default text color (black), and is followed by a line break.
- `Given: ...` is on its own line and ends with a line break.
- `When: ...` is on its own line and ends with a line break.
- `Then: ...` is on its own line and ends with a line break.
- Do not condense Given/When/Then into a single sentence.

Rendered format:

```markdown
- **AC1:**
  Given: [precondition]
  When: [action]
  Then: [expected result]
- **AC2:**
  Given: ...
  When: ...
  Then: ...
```

ADF structure example for one AC item:

```json
{
  "type": "listItem",
  "content": [
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "AC1:", "marks": [{ "type": "strong" }] },
        { "type": "hardBreak" },
        { "type": "text", "text": "Given: [precondition]" },
        { "type": "hardBreak" },
        { "type": "text", "text": "When: [action]" },
        { "type": "hardBreak" },
        { "type": "text", "text": "Then: [expected result]" },
        { "type": "hardBreak" }
      ]
    }
  ]
}
```

## Jira Checklists Field Template

The Definition of Ready checklist maps to `customfield_23603` using Checklists markdown format:

```
## Checklists
- [ ] [checklist item 1]
- [ ] [checklist item 2]
- ...
```

## Execution Steps

1. Read the source markdown file
2. Parse each `### STORY:` section to extract:
   - Title (from heading, excluding ID prefix like "BDCSPM-75199-S1 —")
   - Description block (As a / I want / So that)
   - Context & notes (bullet list)
   - Tasks (checkbox list)
   - Acceptance Criteria (AC1, AC2, ...)
   - Definition of Ready checklist (checkbox list)
3. For each story, call `mcp_atlassian-mcp_createJiraIssue` with:
   - `projectKey`: from input
   - `issueTypeName`: "Story"
   - `summary`: Story title
   - `description`: Combined Description + Context & Notes + Tasks (markdown format)
   - `parent`: Parent Epic key
   - `contentFormat`: "markdown"
   - `additional_fields`:
     - `customfield_12200`: Acceptance Criteria (ADF format; bold black `ACx:` label, hard line breaks after `ACx:`, `Given:`, `When:`, and `Then:`)
     - `customfield_23603`: Checklists content for Definition of Ready
4. Log each created story's key and URL for confirmation

## Example Invocation

> Create Jira stories from `Requirements/BDCSPM-62315/BDCSPM-75199-Stories.md` under Epic BDCSPM-75199.

## Notes

- Skip any story that already exists in Jira (check by summary match if needed)
- Use `contentFormat: "markdown"` for the description field
- The Acceptance Criteria field (`customfield_12200`) uses ADF format — construct using a bullet list whose list items contain a paragraph with bold default-color `ACx:` text and `hardBreak` nodes after `ACx:`, `Given:`, `When:`, and `Then:`
- Checklists field (`customfield_23603`) expects a specific markdown-like syntax with `## Checklists` header and `- [ ]` items
- Respect the story order from the markdown file (S1, S2, S3, ...)
