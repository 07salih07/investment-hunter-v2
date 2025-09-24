# Branch Protection Rules

## Protected Branches
- `main`: Production branch
- `develop`: Integration branch

## Rules for `main`
- Require pull request reviews
- Dismiss stale reviews when new commits are pushed
- Require status checks to pass
- Require branches to be up to date
- Restrict pushes to specific people/teams

## Rules for `develop`
- Require pull request reviews
- Require status checks to pass
- Allow force pushes (for emergency fixes)

## Merge Strategy
- Use "Create a merge commit" (--no-ff)
- Squash commits only for small fixes
- Maintain clear commit history

## Status Checks Required
- Build successful
- Tests passing
- No TypeScript errors
- ESLint compliance
