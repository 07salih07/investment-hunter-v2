# Contributing to Investment Hunter v2.0

## Branch Strategy
- `main`: Production-ready code
- `develop`: Integration branch
- `feature/[task-id]-[description]`: Feature branches
- `hotfix/[description]`: Critical fixes

## Commit Convention
Follow Conventional Commits:
- `feat:` New features
- `fix:` Bug fixes
- `docs:` Documentation
- `chore:` Maintenance

## Development Workflow
1. Create feature branch from `develop`
2. Make changes with descriptive commits
3. Test: `npm run build && npm run dev`
4. Create PR to `develop`
5. Merge with `--no-ff`

## Code Quality
- No mock data in components
- TypeScript strict mode
- ESLint compliance
- Performance optimization
