## Design System
Always read DESIGN.md before making any visual or UI decisions.
All font choices, colors, spacing, and aesthetic direction are defined there.
Do not deviate without explicit user approval.
In QA mode, flag any code that doesn't match DESIGN.md.

Key rules:
- Headings: Fraunces variable serif (NOT Plus Jakarta Sans, NOT Inter, NOT system-ui)
- Body: Schibsted Grotesk
- Dollar amounts / financial figures: Geist Mono with tabular-nums, minimum 18px
- Primary color: #0F6B30 forest green (NOT blue)
- Amber (#B45309): urgency ONLY — permit warnings, flood zone flags, RRSP deadlines. Never casual use.
- Background: #F5F0E8 warm parchment (NOT #F3F4F6 cool gray, NOT #FFFFFF stark white)
- Brass (#A07840): confidence indicators and section dividers ONLY. Never overuse.

## Skill routing

When the user's request matches an available skill, invoke it via the Skill tool. When in doubt, invoke the skill.

Key routing rules:
- Product ideas/brainstorming → invoke /office-hours
- Strategy/scope → invoke /plan-ceo-review
- Architecture → invoke /plan-eng-review
- Design system/plan review → invoke /design-consultation or /plan-design-review
- Full review pipeline → invoke /autoplan
- Bugs/errors → invoke /investigate
- QA/testing site behavior → invoke /qa or /qa-only
- Code review/diff check → invoke /review
- Visual polish → invoke /design-review
- Ship/deploy/PR → invoke /ship or /land-and-deploy
- Save progress → invoke /context-save
- Resume context → invoke /context-restore
- Author a backlog-ready spec/issue → invoke /spec
