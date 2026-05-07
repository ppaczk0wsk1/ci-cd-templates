# Contributing

Thanks for wanting to improve these templates! Here's how to help.

## Adding a New Language Template

1. Create the workflow file in `workflows/ci-<language>.yml`
2. Create a matching doc in `docs/<language>.md`
3. Update the table in `README.md`
4. Open a PR with a clear description

## Guidelines

- **Keep it generic.** Templates should work for any project in that language - avoid framework-specific steps (e.g., don't hardcode Next.js or Django commands).
- **Comment optional sections.** The core CI job should be uncommented and ready to use. Docker, deploy, and release should be commented out.
- **Use latest stable action versions.** Prefer `actions/checkout@v4`, etc.
- **Include caching.** Every template should cache its language's dependencies.
- **Add a matrix.** Test across at least 2-3 recent language versions where applicable.
- **Document it.** The docs page should explain every stage, show how to customise common things, and list any required secrets.

## Style

- Use `# ======` section dividers between jobs for readability
- Keep inline comments short and helpful
- Use `--if-present` or `|| true` for optional steps so workflows don't fail on minimal projects

## Testing

Before submitting, verify your workflow is valid YAML:

```bash
# Quick syntax check
python -c "import yaml; yaml.safe_load(open('workflows/ci-yourlang.yml'))"
```

If you have [act](https://github.com/nektos/act) installed, you can test locally:

```bash
act -W workflows/ci-yourlang.yml --dryrun
```
