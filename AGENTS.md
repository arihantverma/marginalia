# Marginalia

A collection of programming learnings, insights, and discoveries captured from daily work.

## Purpose

When the user asks to "dump learnings" or "save what I learned" from a conversation, create an markdown file in this directory capturing the insights.

## File Naming Convention

```
YYYY-MM-DD-<slug>.md
```

Example: `2026-01-30-typescript-and-react-patterns.md`

- Use the current date
- Slug should be lowercase, hyphenated, descriptive of the main topic(s)
- A single file can contain multiple learnings from the same conversation

## MD File Structure

```md
---
title: "Brief, descriptive title"
date: YYYY-MM-DD
tags: ["tag1", "tag2"]
---

## Context

Brief description of what prompted these learnings (problem being solved, code being written, etc.)

## Learnings

### Topic/Concept Name

Detailed explanation of the learning. Include:

- Code snippets if relevant
- Why it matters
- Common pitfalls or misconceptions clarified
- Step-by-step reasoning where applicable

### Another Topic (if applicable)

Continue with additional learnings from the same conversation.

## References

Links to docs, articles, or resources if applicable.
```

## Guidelines

- Be detailed—capture the full explanation, not a summary
- Multiple learnings from one conversation go in the same file
- Include the problem or question that led to the learning, not just the answer
- Prefer concrete examples over abstract explanations
- Include code snippets generously with explanations
- Tag generously for future discoverability
- Write for future discoverability, assume that when I came back to the file later, I might have forgotten some things, so the content should be self-contained and easy to read to remember.
