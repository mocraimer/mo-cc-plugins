# gamified-code-review

A Claude Code plugin that turns pull request reviews into a gamified tower-climbing experience.

## What It Does

Review Tower transforms code reviews into an RPG-style game where each PR becomes a tower with floors to climb. Files are grouped into floors by directory, each assigned a difficulty level (Easy, Normal, Boss). You choose your review mode and earn XP, titles, and badges as you progress.

## Features

- **Challenge Mode** (2x multiplier): Review the code independently before seeing AI findings. Earn bonus points for Human Edge discoveries.
- **Scout Mode** (1x multiplier): Review AI-identified findings one by one, acknowledging or disputing each.
- **RPG Progression**: Earn XP across sessions, climb tiers from Squire to Archmage (or choose Cyberpunk/Academic themes).
- **Session Badges**: Earn badges like Bug Hunter, Sentinel, Eagle Eye, and more based on your review performance.
- **Export to PR**: Export your review findings and post them as PR comments directly from the Review Tower UI.
- **Floor Revisit**: Return to completed floors to review your findings and results.

## Usage

Invoke the skill with:

```
/review-tower <PR-URL-or-number>
```

Or use natural language triggers:
- "gamify code review"
- "review PR as a game"
- "tower review"

The skill fetches the PR diff, analyzes the code, builds a tower, and opens an interactive HTML review session in your browser.

## Stats Persistence

Your XP, tier, titles, badges, and session history persist across sessions at:

```
~/.claude/review-tower/stats.json
```

This file lives outside the plugin directory so your progress survives plugin updates and reinstalls.
