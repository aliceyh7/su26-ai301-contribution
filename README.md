# Contribution [1]: llama-server hot swapping cvectors via API like LoRA adapters

**Contribution Number:** [1]  
**Student:** [Alice]  
**Issue:** https://github.com/ggml-org/llama.cpp/issues/10685  
**Status:** [Phase I]

---

## Why I Chose This Issue

I chose llama.cpp issue #10685, "Feature Request: llama-server hot swapping cvectors via API like we can do with LoRA adapters now," because it connects directly to my interest in LLM systems, inference infrastructure, and model steering. The issue proposes exposing control-vector loading/unloading and scaling through the `llama-server` API, similar to the existing LoRA adapter API, so users can adjust model behavior without restarting the server.

This issue stood out to me because it has both practical product value and strong technical learning value. From reading the issue thread, I understand that control vectors can already be used through command-line flags, but changing their scale or toggling them currently requires restarting the server. A useful contribution would be to add API endpoints such as `GET /cvectors` and `POST /cvectors`, following the existing `/lora-adapters` pattern. This matches my interest in post-training-adjacent workflows, runtime model adaptation, and LLM serving systems, while giving me a chance to work in a widely used C/C++ AI codebase.

I also liked that the issue is labeled `enhancement` and `good first issue`, has a clear user motivation, and provides a concrete possible implementation. The main risk is that someone previously commented that they planned to work on it, so I plan to confirm that there is no active PR or assignee before fully committing.

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
