# DAB Demo Checklist – Ready for Tomorrow

## ✅ Already installed & configured

| Component | Status |
|-----------|--------|
| **Databricks CLI** | v0.251.0 ✓ |
| **Bundle commands** | Available ✓ |
| **Authentication** | Configured (default profile → e2-demo-field-eng) ✓ |
| **Demo project** | Created in `my_project/` ✓ |

## Quick demo flow

1. **Validate** (no workspace needed):
   ```bash
   cd my_project
   databricks bundle validate -t dev
   ```

2. **Deploy** to dev workspace:
   ```bash
   databricks bundle deploy -t dev
   ```

3. **Run** the job or pipeline:
   ```bash
   databricks bundle run my_project_job
   # or
   databricks bundle run my_project_pipeline
   ```

4. **Optional**: Show `databricks bundle summary` after deploy.

## Project structure (my_project)

- `databricks.yml` – bundle config, targets (dev/prod)
- `resources/` – job and Lakeflow Declarative Pipeline definitions
- `src/` – notebooks and Python code (including a Lakeflow Declarative Pipeline)
- `tests/` – unit tests

## Useful commands

```bash
databricks bundle validate -t dev    # Validate config
databricks bundle deploy -t dev      # Deploy to dev
databricks bundle run <job_or_pipeline_name>
databricks bundle destroy -t dev     # Tear down
databricks bundle summary -t dev     # List deployed resources
```

## Docs

- [DAB Overview](https://docs.databricks.com/dev-tools/bundles/index.html)
- [Bundle Configuration](https://docs.databricks.com/dev-tools/bundles/settings)
- [Deployment Modes](https://docs.databricks.com/dev-tools/bundles/deployment-modes)
