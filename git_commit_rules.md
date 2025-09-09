# The Seven Rules of Git Commit Messages

A well-crafted Git commit message is the best way to communicate context about a change to fellow developers and your future self. Here are the seven core rules:

## 1. Separate subject from body with a blank line

```
Summarize changes in around 50 characters or less

More detailed explanatory text, if necessary. Wrap it to about 72
characters or so. The blank line separating the summary from the body
is critical; various tools like log, shortlog and rebase can get
confused if you run the two together.
```

## 2. Limit the subject line to 50 characters

50 characters is a rule of thumb (72 is the hard limit). This ensures readability and forces concise explanations.

**Good:** `Fix typo in introduction to user guide`

## 3. Capitalize the subject line

Begin all subject lines with a capital letter.

**Good:** `Accelerate to 88 miles per hour`  
**Bad:** `accelerate to 88 miles per hour`

## 4. Do not end the subject line with a period

Trailing punctuation is unnecessary and wastes precious space.

**Good:** `Open the pod bay doors`  
**Bad:** `Open the pod bay doors.`

## 5. Use the imperative mood in the subject line

Write as if giving a command or instruction. Your commit should complete this sentence:

> "If applied, this commit will **[your subject line here]**"

**Examples:**
- `Refactor subsystem X for readability`
- `Update getting started documentation`  
- `Remove deprecated methods`
- `Release version 1.0.0`

**Bad examples:**
- `Fixed bug with Y`
- `Changing behavior of X`
- `More fixes for broken stuff`

## 6. Wrap the body at 72 characters

Git never wraps text automatically. Manually wrap at 72 characters so Git has room to indent while keeping everything under 80 characters total.

## 7. Use the body to explain what and why vs. how

Focus on:
- **What** changed
- **Why** you made the change
- What was wrong with the old way
- How things work now
- Why you solved it this way

Don't explain **how** the change was made - the code shows that. If the code is complex, use source comments instead.

## Example Format

```
Fix issue with user authentication timeout

The previous timeout value of 30 seconds was too short for users
on slow connections, causing frequent re-authentication requests.

This change increases the timeout to 5 minutes and adds a visual
indicator when the session is about to expire, improving the user
experience for all connection speeds.

Resolves: #123
See also: #456, #789
```

## Quick Reference

**When to use body:**
- Complex changes requiring explanation
- Breaking changes or API modifications  
- Bug fixes where the cause isn't obvious
- New features or significant functionality
- Changes affecting multiple components
- Any time the "why" isn't clear from code

**Conventional Commits format:**
- ✅ Use required format: `<type>[scope]: [PROJECT-ID] <description>`
- ✅ Include mandatory JIRA ticket: `[TXB-1234]`, `[DEV-5678]`, etc.
- ✅ Use `feat:` for new features, `fix:` for bug fixes
- ✅ Add `!` or `BREAKING CHANGE:` footer for breaking changes
- ✅ Use descriptive scopes when helpful: `feat(auth): [TXB-1234] ...`

**General formatting rules:**
- ✅ Separate subject and body with blank line
- ✅ Subject line ≤ 50 characters  
- ✅ Lowercase the description (after colon)
- ✅ No period at end of subject
- ✅ Use imperative mood ("add feature" not "added feature")
- ✅ Wrap body at 72 characters
- ✅ Explain what and why, not how