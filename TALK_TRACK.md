# DAB Demo — Talk Track

Use this as a script to practice. Italics are stage directions (what you do on screen). Regular text is what you say out loud.

---

## Step 1: Project Structure (~3 min)

*Open your IDE with the project visible in the file explorer.*

"So let me show you what a Databricks Asset Bundle actually looks like in practice. It's just a folder of files — YAML config and your source code, living together."

*Open `databricks.yml`.*

"This is the heart of the bundle. At the top we name the bundle. Then we have an `include` that pulls in resource definitions from the `resources/` folder — this keeps things modular, you don't need one giant YAML file."

*Scroll down to the `variables` block.*

"Next we have variables. These are parameters that can change between environments. Catalog name, schema prefix, instance type, how many workers we want. Each one has a default, but we can override them per target — I'll show you that in a second."

*Scroll down to `targets`.*

"Targets are your deployment environments. We have `dev` and `prod`. Dev mode is special — Databricks automatically prefixes all your resources with your username and pauses any schedules. So you can deploy without stepping on anyone else's toes. Prod mode enforces best practices — it'll validate you're not deploying from a personal branch, it sets permissions, the whole thing."

"Notice dev overrides `max_workers` to 2 — we don't need a big cluster for development. Prod gets 4."

*Open `resources/my_project.job.yml`.*

"Here's our job definition. This is a workflow with three tasks chained together. First a notebook task, then a Lakeflow Declarative Pipeline refresh, then a Python wheel task. These run in sequence — each one depends on the previous."

*Point to `${resources.pipelines.my_project_pipeline.id}`.*

"See this reference here? The job doesn't hardcode a pipeline ID. It references the pipeline we define in another file. When you deploy, Databricks resolves these automatically. Everything stays in sync."

*Point to `${var.node_type_id}` and `${var.max_workers}`.*

"And here's where our variables show up. The cluster config uses `var.node_type_id` and `var.max_workers` — so dev and prod get different compute, from the same definition."

*Open `resources/my_project.pipeline.yml`.*

"Same idea here — the Lakeflow Declarative Pipeline uses `var.catalog` and builds the schema name dynamically from the target. So dev writes to `my_project_dev`, prod writes to `my_project_prod`. No manual schema management."

*Open `src/dlt_pipeline.ipynb`.*

"And this is just normal Lakeflow Declarative Pipeline code. You still `import dlt` — that's the Python module name — then define your tables with decorators. Nothing DAB-specific in here — it's the same code you'd write in the workspace. The bundle just manages how it gets deployed."

---

## Step 2: Validate (~1 min)

*Switch to your terminal.*

"Before we deploy anything, let's validate. This checks for YAML syntax errors, missing references, invalid variable types — catches problems before they hit the workspace."

*Run:*
```
databricks bundle validate -t dev
```

"Validation OK. If I had a typo in a variable name or a broken resource reference, this would catch it right here. You can also run this in CI — which I'll show you later."

---

## Step 3: Deploy (~3 min)

"Now let's deploy to dev."

*Run:*
```
databricks bundle deploy -t dev
```

*Wait for it to complete, then switch to the Databricks workspace UI.*

"Let's go look at what happened in the workspace."

*Navigate to Workflows.*

"Here's our job — notice the name: `[dev bart.prikker] my_project_job`. That prefix is automatic in dev mode. If my colleague deploys the same bundle, they'd get `[dev their_name]` — completely isolated."

*Click into the job, show the task DAG.*

"Three tasks, chained. Notebook, then pipeline, then the wheel task. All defined in YAML, deployed with one command."

*Navigate to Pipelines.*

"And here's the Lakeflow Declarative Pipeline — also prefixed with `[dev bart.prikker]`. If I look at the schedule on the job... it's paused. Dev mode does that automatically so you don't accidentally kick off scheduled runs during development."

---

## Step 4: Run (~3 min)

"Let's run it and see everything work end to end."

*Run:*
```
databricks bundle run my_project_job
```

*Switch to the workspace, show the job running.*

"The first task is running our notebook — it reads from the NYC taxi sample dataset. When that completes, it triggers the Lakeflow Declarative Pipeline which creates a filtered table. Then the wheel task runs our packaged Python code."

*Wait for completion (or narrate while it runs).*

"This is nice because you're testing the full production workflow — not just a notebook in isolation. The job, the pipeline, the dependencies, the cluster config — it's all the real thing."

---

## Step 5: Live Change & Redeploy (~4 min)

"Now here's where it gets really interesting. Let's say I need to change the business logic."

*Switch to the IDE. Open `src/dlt_pipeline.ipynb`. Find the filter line.*

"Right now we're filtering for taxi fares under 30 dollars. Let's say the business wants to tighten this to under 10."

*Change `fare_amount < 30` to `fare_amount < 10`.*

"I've made my change. Now watch how fast the iteration loop is."

*Run:*
```
databricks bundle deploy -t dev
```

"Deploy picks up the change. It only syncs what's different — it doesn't redeploy everything from scratch."

*Run:*
```
databricks bundle run my_project_job
```

"And now when the pipeline runs, it'll use the updated filter. No clicking through the UI, no manually uploading notebooks. Change code locally, deploy, run. That's the inner loop."

*Show the pipeline results in the workspace if time allows.*

---

## Step 6: Variables & Overrides (~3 min)

"Let me show you something else that's powerful. We already saw that dev and prod have different variable values. But you can also override at deploy time."

*Run:*
```
databricks bundle deploy -t dev --var="max_workers=1"
```

"I just deployed with a single worker. Maybe I'm testing something small and don't want to burn compute. I didn't touch any config files — it's a one-time override."

"This is really useful in CI too. You could pass different variables per branch, per environment, whatever you need. One codebase, infinite configurations."

*Switch to `databricks.yml` in the IDE.*

"And if you look at the variables block — each one has a description. So when a new team member opens this project, they can see what's configurable and what the defaults are. It's self-documenting."

---

## Step 7: CI/CD with GitHub Actions (~3 min)

"Okay so we've been doing everything from the CLI. But in a real team, you want this automated."

*Open `.github/workflows/deploy.yml` in the IDE.*

"This is a GitHub Actions workflow. Let me walk through it."

"Three jobs. First — validate. This runs on every pull request AND every push to main. It installs the Databricks CLI using the official setup action, then runs `bundle validate`. If your config is broken, the PR fails. You catch it before it merges."

*Point to deploy-dev.*

"Second — deploy to dev. This only runs on pushes to main — so when a PR gets merged. It deploys the dev target. Your dev environment always reflects what's on main."

*Point to deploy-prod.*

"Third — deploy to prod. This also runs on merge to main, but notice this line: `environment: production`. In GitHub, you can set up environment protection rules — require a manual approval, restrict to certain people. So merging the PR deploys to dev automatically, but prod requires someone to click approve."

"The auth is handled through GitHub secrets — `DATABRICKS_HOST` and `DATABRICKS_TOKEN`. No credentials in the repo."

*Show the GitHub repo briefly: https://github.com/bartpdb/DABS_demo*

"The repo is right here. In a real setup, you'd open a PR, the validate job runs, your team reviews, you merge, dev deploys, someone approves prod. Full software engineering workflow for your data pipelines."

---

## Step 8: Clean Up (~1 min)

"One last thing. When you're done, you can tear everything down with one command."

*Run:*
```
databricks bundle destroy -t dev
```

"This removes the job, the pipeline, all the deployed files — everything that was created by the bundle. Clean workspace. No orphaned resources."

---

## Step 9: Wrap Up (~1 min)

"So to recap — Databricks Asset Bundles give you:"

"One: everything as code. Jobs, pipelines, clusters, permissions — all in version-controlled YAML alongside your source code."

"Two: fast iteration. Validate locally, deploy in seconds, run immediately. No UI clicking."

"Three: multi-environment support. Dev and prod from one codebase, with variables that adjust per target."

"Four: CI/CD ready out of the box. GitHub Actions, Azure DevOps, whatever your team uses — it's just CLI commands."

"Five: clean lifecycle management. Deploy, update, destroy. No orphaned resources."

"Any questions?"

---

## Timing Guide

| Step | Topic | Time |
|------|-------|------|
| 1 | Project structure | ~3 min |
| 2 | Validate | ~1 min |
| 3 | Deploy | ~3 min |
| 4 | Run | ~3 min |
| 5 | Live change | ~4 min |
| 6 | Variables | ~3 min |
| 7 | CI/CD | ~3 min |
| 8 | Clean up | ~1 min |
| 9 | Wrap up | ~1 min |
| | **Total** | **~22 min** |

Buffer for cluster spin-up time, Q&A, and navigating between screens: plan for **30 min**.

---

## Tips

- Pre-deploy before the demo so the cluster is warm — `databricks bundle deploy -t dev` and even kick off one run. That way step 4 won't have a cold start delay.
- Keep the workspace UI open in a browser tab, ready on the Workflows page.
- If a run takes too long, just narrate what would happen and move on.
- The live change in step 5 is the "wow" moment — practice the flow of edit, deploy, run so it feels smooth.
