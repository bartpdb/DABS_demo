# DAB Demo — Talk Track

Use this as a script to practice. Italics are stage directions (what you do on screen). Regular text is what you say out loud.

---

## Step 1: Project Structure (~4 min)

*Open your IDE with the project visible in the file explorer.*

"So how do most teams manage their Databricks projects today? Usually it's a mix of notebooks in the workspace, jobs configured through the UI, maybe some Terraform on the side, and a lot of tribal knowledge about which notebook goes where. If someone leaves the team, good luck figuring out how it all fits together."

"Databricks Asset Bundles flip that. Everything — your code, your job definitions, your pipeline config, your permissions — lives in one folder, version-controlled, reviewable, deployable with a single command. Let me show you what that looks like."

*Open `databricks.yml`.*

"This is the heart of the bundle. At the top we name the bundle. Then we have an `include` that pulls in resource definitions from the `resources/` folder — this keeps things modular, you don't need one giant YAML file."

*Scroll down to the `variables` block.*

"Next we have variables. These are parameters that can change between environments — catalog name, schema prefix. Each one has a default, but we can override them per target — I'll show you that in a second."

*Scroll down to `targets`.*

"Targets are your deployment environments. We have `dev` and `prod`. Dev mode is special — Databricks automatically prefixes all your resources with your username and pauses any schedules. So you can deploy without stepping on anyone else's toes. Prod mode enforces best practices — it'll validate you're not deploying from a personal branch, it sets permissions, the whole thing."

*Point to `run_as` in the prod target.*

"Now this is important — see `run_as` on the prod target? This is set to a service principal. By default, a job runs as whoever deployed it. That creates a problem — if I deploy a production job and it runs as my personal account, it depends on *my* permissions to access storage, Unity Catalog tables, everything. If I go on vacation and my token expires, or I leave the company and my account gets deactivated — the production job breaks."

"With `run_as`, the job runs as a service principal — a non-human identity with its own permissions. The person who deploys and the identity that runs are decoupled. Your CI/CD pipeline deploys as one service principal, the job runs as another with just the permissions it needs. That's how you do production properly."

*Open `resources/my_project.job.yml`.*

"Here's our job definition. This is a workflow with three tasks chained together. First a notebook task, then a Lakeflow Declarative Pipeline refresh, then a Python wheel task. These run in sequence — each one depends on the previous."

"Notice there's no cluster configuration here — no instance types, no worker counts. This entire job runs on serverless compute. Databricks manages the infrastructure for you. For the notebook task, you just omit the cluster key and it defaults to serverless. For the Python wheel task, you reference an `environment_key` instead."

*Point to `${resources.pipelines.my_project_pipeline.id}`.*

"See this reference here? The job doesn't hardcode a pipeline ID. It references the pipeline we define in another file. When you deploy, Databricks resolves these automatically. Everything stays in sync."

*Open `resources/my_project.pipeline.yml`.*

"Same idea here — the Lakeflow Declarative Pipeline uses `var.catalog` and builds the schema name dynamically from the target. So dev writes to `my_project_dev`, prod writes to `my_project_prod`. No manual schema management. And notice `serverless: true` — the pipeline also runs without any cluster config."

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

"Here's our job — notice the name: `[dev bart.prikker] my_project_job`. That prefix is automatic in dev mode. If my colleague deploys the same bundle, they'd get `[dev their_name]` — completely isolated. Ten engineers can deploy the same bundle simultaneously and nobody steps on anyone else."

*Click into the job, show the task DAG.*

"Three tasks, chained. Notebook, then pipeline, then the wheel task. All defined in YAML, deployed with one command. And look — each of these is a different task type. A notebook for interactive exploration, a Lakeflow Declarative Pipeline for managed ETL, and a Python wheel for production-grade packaged code. Bundles handle all of them."

*Navigate to Pipelines.*

"And here's the Lakeflow Declarative Pipeline — also prefixed with `[dev bart.prikker]`. If I look at the schedule on the job... it's paused. Dev mode does that automatically so you don't accidentally trigger scheduled runs during development. When you deploy to prod, the schedule activates. You don't have to remember to flip that switch — the bundle does it for you."

---

## Step 4: Run (~3 min)

"Let's run it and see everything work end to end."

*Run:*
```
databricks bundle run my_project_job
```

*Switch to the workspace, show the job running.*

"The first task is running our notebook — it reads from the NYC taxi sample dataset. When that completes, it triggers the Lakeflow Declarative Pipeline which creates a filtered table in Unity Catalog. Then the wheel task runs our packaged Python code."

*Wait for completion (or narrate while it runs).*

"This is important — I'm not just testing a notebook in isolation. I'm testing the full production workflow end to end. The task dependencies, the pipeline, the packaged wheel, the data flowing between steps. If something's going to break, I want to know now, not when it hits production."

"And because it's serverless, there's no cluster spin-up delay. I didn't have to specify instance types, Spark versions, worker counts — none of that. Databricks manages the compute. I just say what I want to run and it runs."

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
databricks bundle deploy -t dev --var="schema_prefix=demo_output"
```

"I just changed where the pipeline writes its output — a completely different schema — without touching any config files. One-time override from the command line."

"You can override any variable this way. For example, if I wanted to switch the catalog entirely..."

*Run:*
```
databricks bundle deploy -t dev --var="catalog=dev_sandbox"
```

*The CLI will warn about destructive actions. Don't approve — just show the message.*

"See what happened? Databricks is telling us that changing the catalog requires recreating the pipeline and doing a full refresh of the data. It won't do that without explicit approval. This is a safety guardrail — the CLI protects you from accidentally destroying production tables by changing a variable. In a real workflow you'd review this carefully before approving."

*Cancel and redeploy with defaults to reset:*
```
databricks bundle deploy -t dev
```

"This pattern becomes really powerful at scale. Think about it — your CI/CD pipeline can pass different variables depending on the context. Feature branch? Deploy to a sandbox catalog so your work is isolated. Merge to main? Deploy to the shared dev catalog. Promote to prod? Point at the production catalog with tighter permissions."

"Or think about a data team with multiple clients. Same pipeline code, but you override `schema_prefix` per client — `client_a`, `client_b` — each gets their own tables, same logic. No code duplication."

"You can also use this for cost control. Maybe you have a `compute_size` variable — in dev it's `small`, in prod it's `large`. Or a `sample_data` variable — `true` in dev so you only process a subset, `false` in prod so you get the full dataset. All from one codebase."

*Switch to `databricks.yml` in the IDE.*

"And if you look at the variables block — each one has a description. So when a new team member opens this project, they immediately see what's configurable, what the defaults are, and what each variable does. It's self-documenting infrastructure."

---

## Step 7: CI/CD with GitHub Actions (~4 min)

"Okay so we've been doing everything from the CLI. That's great for development, but in a real team you don't want deployments to depend on someone running commands from their laptop. You want it automated — push code, it deploys."

"And this is where bundles really shine compared to the old way. Think about what we'd need without bundles — custom scripts to upload notebooks, API calls to update job definitions, maybe Terraform for the infrastructure, some glue code to stitch it all together. With bundles, your CI/CD is literally three commands: install the CLI, validate, deploy. That's it."

*Open `.github/workflows/deploy.yml` in the IDE.*

"Here's what that looks like as a GitHub Actions workflow."

"Three jobs. First — validate. This runs on every pull request AND every push to main. It installs the Databricks CLI using the official setup action — that's a one-liner maintained by Databricks — then runs `bundle validate`. If your config is broken, the PR fails. Your team catches it in code review before it ever touches the workspace."

*Point to deploy-dev.*

"Second — deploy to dev. This only runs on pushes to main — so when a PR gets merged. It deploys the dev target. Your dev environment always reflects what's on main. No drift, no 'I forgot to deploy' — it's automatic."

*Point to deploy-prod.*

"Third — deploy to prod. This also runs on merge to main, but notice this line: `environment: production`. In GitHub, you can set up environment protection rules — require a manual approval, restrict who can approve, even require a certain number of reviewers. So merging the PR deploys to dev automatically, but prod requires someone to click approve. That's your change management process, built right into the pipeline."

"The auth is handled through GitHub secrets — `DATABRICKS_HOST` and `DATABRICKS_TOKEN`. No credentials in the repo. In a production setup, you'd use a service principal token here — so the CI/CD pipeline deploys as the service principal, and combined with the `run_as` setting we saw earlier, the job also runs as a service principal. No human identity in the loop at all."

*Show the GitHub repo briefly: https://github.com/bartpdb/DABS_demo*

"The repo is right here. And this isn't GitHub-specific. The same pattern works with Azure DevOps, GitLab CI, Bitbucket Pipelines — anything that can run a shell command can deploy a bundle. The Databricks CLI is the universal interface."

"So the full workflow looks like this: engineer opens a PR, the validate job runs automatically, team reviews the code and the YAML changes together, they merge, dev deploys, stakeholder approves prod, production updates. Full software engineering lifecycle — for your data pipelines."

---

## Step 8: Clean Up (~1 min)

"One last thing. When you're done, you can tear everything down with one command."

*Run:*
```
databricks bundle destroy -t dev
```

"This removes the job, the pipeline, all the deployed files — everything that was created by the bundle. Clean workspace. No orphaned resources."

---

## Step 9: Wrap Up (~2 min)

"So to recap — what did we just see?"

"We took a project with a job, a Lakeflow Declarative Pipeline, and packaged Python code. We validated it locally, deployed it to a workspace, ran it end to end on serverless compute, made a live code change, redeployed in seconds, and showed how variables let you target different environments. All from the command line, all version-controlled, all reproducible."

"If I had to boil down why this matters, it's this: Databricks Asset Bundles bring software engineering best practices to data engineering. The same workflow your application developers use — branches, pull requests, code review, automated testing, CI/CD — now works for your data pipelines, your ML models, your analytics jobs."

"And you saw how little configuration we needed. Serverless compute means no cluster management. Variables mean no copy-pasting config between environments. Dev mode means no stepping on your teammates. The CLI means no clicking through UIs."

"One more thing worth mentioning — everything I showed you today started from a template. You can create your own custom templates with your organization's standards baked in. Default permissions, required service principals, standard CI/CD config, naming conventions. A new team member runs `databricks bundle init your-company-template` and they get a project that already follows all your best practices from day one."

"Any questions?"

---

## Timing Guide

| Step | Topic | Time |
|------|-------|------|
| 1 | Project structure + the "why" | ~4 min |
| 2 | Validate | ~1 min |
| 3 | Deploy + workspace walkthrough | ~3 min |
| 4 | Run + serverless story | ~3 min |
| 5 | Live change & redeploy | ~4 min |
| 6 | Variables, overrides & guardrails | ~4 min |
| 7 | CI/CD + DevOps story | ~4 min |
| 8 | Clean up | ~1 min |
| 9 | Wrap up + custom templates | ~2 min |
| | **Total** | **~26 min** |

Buffer for serverless startup, Q&A, and navigating between screens: plan for **30-35 min**.

---

## Tips

- Pre-deploy before the demo — `databricks bundle deploy -t dev` and even kick off one run. Serverless is fast but the first run can still take a minute.
- Keep the workspace UI open in a browser tab, ready on the Workflows page.
- If a run takes too long, just narrate what would happen and move on.
- The live change in step 5 is the "wow" moment — practice the flow of edit, deploy, run so it feels smooth.
- Serverless = no cluster spin-up wait. This makes the demo flow much smoother than classic compute.
