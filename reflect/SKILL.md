---
name: reflect
description: Analyze conversation sessions to extract corrections, success patterns, and improvement opportunities - edits skills, knowledge, and conventions
version: 1.0.0
last_updated: 2026-03-04
args: "skill-name"
when_to_use: After completing a coding/review session where user provided corrections, user explicitly asks to reflect on a skill, user approves work as best practice, after repeatedly giving similar feedback, or to iteratively improve skill quality
context_files: [skills/, knowledge/, conventions/]
---

# Skill Reflection

Analyze the current conversation to extract learnings and improve skills based on user corrections and approvals.

## Overview

This skill implements a manual reflection process where you:
1. Scan the conversation for signals (corrections and approvals)
2. Categorize findings by confidence level
3. Propose specific updates to **skills**, **knowledge**, and **conventions**
4. Apply changes with user approval
5. Commit to git with descriptive messages

**Note:** This skill edits files in `.claude/skills/`, `.claude/knowledge/`, and `.claude/conventions/` based on learnings.

## Analyzing the Conversation

Scan the **current conversation only** for these signals:

### HIGH Confidence: Explicit Corrections
User directly corrects or instructs:
- "Always check for X"
- "Never do Y"
- "You should have Z"
- "Don't forget to..."

**Example**: "Always check for SQL injection too" → Add SQL injection check to code-review skill

### MEDIUM Confidence: Success Patterns
User approves or confirms:
- "perfect"
- "great"
- "exactly"
- "that's what I wanted"
- Accepting proposed changes without modification

**Example**: User says "perfect" after you suggest a pattern → Document that pattern in the skill

### LOW Confidence: Observations
Patterns you notice but aren't explicitly confirmed:
- Repeated questions from user
- Similar corrections across multiple files
- Emerging patterns

**Note**: Only include LOW confidence if there's a clear pattern. When in doubt, ask the user.

## Confidence Level Guidelines

- **HIGH**: Explicit corrections that should be added as rules/requirements
- **MEDIUM**: Patterns that worked well and should be documented as best practices
- **LOW**: Observations to discuss with user before adding

## Output Format

Present findings in this structure:

```
Skill Reflection: {skill-name}

Signals detected: X corrections, Y successes

Proposed changes:

● [HIGH] + {Category}: "{specific rule or check}"

● [MED] + {Category}: "{best practice or pattern}"

● [LOW] + {Category}: "{observation to review}"

Commit: "{concise commit message}"
```

## Applying Changes

1. **Show proposed changes**: Display the full summary above
2. **Ask for approval**: "Apply these changes? [Y/n/change]"
3. **If 'Y'**: Edit the skill file with proposed changes
4. **If 'n'**: Abort without changes
5. **If 'change'**: Ask user what to modify

## File Updates (Skills, Knowledge, Conventions)

When applying changes:

1. **Read the target file first** to understand structure
2. **Determine the right location**:
   - `skills/` - Workflow instructions and automation
   - `knowledge/` - Component catalogs, references, patterns
   - `conventions/` - Coding standards, architecture rules
3. **Check for related files**: Search for existing related docs that may need cross-references or updates (don't create in isolation)
4. **Categorize additions** appropriately:
   - Add HIGH confidence items as explicit rules/requirements
   - Add MEDIUM confidence items as best practices or examples
   - Discuss LOW confidence items with user before adding
4. **Maintain existing structure**: Don't reorganize unless necessary
5. **Be specific**: Add concrete, actionable guidance
6. **Keep it concise**: Follow "concise is key" principle
7. **Add cross-references**: When creating new convention categories, update related existing conventions with "Related" sections linking to the new docs

## Git Integration

After successfully updating files:

1. **Check git status**: Verify the files were modified
2. **Stage changes**: `git add .claude/skills/... .claude/knowledge/... .claude/conventions/...`
3. **Commit**: Use the commit message from the reflection summary
4. **Ask about push**: "Push to remote? [Y/n]"
5. **Push if approved**: `git push`

## Important Notes

- Always show exact changes before applying
- Never modify skills without explicit user approval
- Commit messages should be concise and descriptive
- Push only after successful commit
- If no signals found, tell the user and don't force changes
- Focus on actionable, specific improvements

## Examples

### Example 1: Code Review Skill

**Conversation signals**:
- User: "Always check for SQL injection too"
- User: "perfect" (after you added the check)

**Output**:
```
Skill Reflection: code-review

Signals detected: 1 correction, 1 success

Proposed changes:

● [HIGH] + Security: "Check for SQL injection vulnerabilities"

● [MED] + Add: "Flag raw SQL queries for review"

● [MED] + Add: "Verify parameterized queries"

Commit: "code-review: add SQL injection checks"
```

### Example 2: Frontend Design Skill

**Conversation signals**:
- User: "Use gradients only when explicitly requested"
- User: "Dark backgrounds should use #000, not #1a1a1a"
- User: "perfect" (after applying CSS Grid)

**Output**:
```
Skill Reflection: frontend-design

Signals detected: 2 corrections, 1 success

Proposed changes:

● [HIGH] + Constraints/NEVER: "Use gradients unless explicitly requested"

● [HIGH] + Color & Theme: "Dark backgrounds: use #000, not #1a1a1a"

● [MED] + Layout: "Prefer CSS Grid for card layouts"

Commit: "frontend-design: no gradients, #000 dark"
```

## Tips for Effective Reflection

1. **Be objective**: Extract what the user actually said, not interpretations
2. **Be specific**: "Check for SQL injection" not "improve security"
3. **Group related items**: Combine similar corrections into one cohesive addition
4. **Prioritize HIGH**: Focus on explicit corrections first
5. **Ask when unclear**: If a signal is ambiguous, ask the user for clarification
