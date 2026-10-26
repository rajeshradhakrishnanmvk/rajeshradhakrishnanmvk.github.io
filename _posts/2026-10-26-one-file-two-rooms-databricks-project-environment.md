---
layout: post
tags:
  - databricks
  - lakehouse
  - architecture
  - python
---

# One File, Two Rooms: What a Databricks Project Environment Changes for an Architect

On 22 October 2026 the product notes on AWS, Azure, and Google Cloud mark two Betas behind one preview switch, Project-Scoped Shared Compute. A workspace admin turns it on from the Previews page. It stays off until then. One Beta lets a serverless notebook task or a Python script task take its job environment from a `pyproject.toml`. The other, Shared Serverless for Asset Bundles, lets notebooks and files in one bundle share serverless compute in the editor, and an environment from the file beside `databricks.yml`. I picked the file over the same day's Entra ID connector and over the 23 October Files API. The file decides who shares a process. The AWS and Google Cloud October headers still say 1 October, and the feature pages were last updated on 22 October. I did not find a Databricks blog post, a LinkedIn post, or a Hacker News thread this week. Git Folders already shared an editor process, in Beta on 2 September, and that note does not mention the switch. The serverless notes on AWS and Google Cloud still say last updated 22 October. On all three clouds the newest entry is the 23 October cache preview, with no project-environment line.

## The editor is one process, per person

Notebooks and files in the same Git folder or bundle attach to one serverless compute resource and one Python environment. Only assets inside that folder or bundle can attach. The root `pyproject.toml` is optional. When it is present, it replaces the per-notebook list in the Environment side pane. Shared Serverless requires environment version 5 or above. The editor example pins `"5"` and `simplejson==3.18.1`.

```toml
[project]
name = "my-project"
version = "0.1.0"
dependencies = [
  "simplejson==3.18.1",
]

[tool.databricks.environment]
environment_version = "5"
```

`%uv add cowsay` writes the root file, updates `uv.lock`, and updates the shared environment. The shared-compute page says you skip `%uv sync` after that add. A direct edit needs `%uv sync`, or Apply. The libraries page, also dated 22 October, keeps `%uv pip` on the notebook virtual environment, beside `%pip`, and sends a reproducible notebook environment to the side pane.

The process is shared. The Python names are not. A name defined in one notebook is absent from the next. A web terminal from an attached file runs on the same resource. The first user to attach is the only one who can run on it. Each collaborator should clone into a personal Git folder and exchange commits. A different serverless usage policy in the same folder gets a second compute resource. That is the only limitation the page prints. The Git Folders URL and the shared-serverless URL on AWS both returned this page, last updated 22 October, and both require the preview.

## The job keeps the file

Schedule, on a notebook attached to Shared Serverless, uses the root `pyproject.toml` as the job's project environment when the file exists. You select serverless compute, then check the job. On a Git folder, Project environment should point at the file. On a bundle, the editor opens a generated YAML under `resources/`, with `project_environment` under the environment `spec`, relative to that file. Deploy creates the job. Schedules are paused by default in development mode. The shared process stays in the editor.

The dependencies page, updated 22 October, also lets you point a task at the file directly. A project environment configures a serverless notebook task, a Python script task, or a Python wheel task from a workspace path such as `/Workspace/path/to/pyproject.toml`. Environment source offers Project only while the preview is on. If the choice is missing, the page says to follow the Standard steps. Project cannot also set Base environment or Dependencies. On the Jobs API and in a bundle, `project_environment` is the only environment-source field in `spec`. Omit `environment_version`, `base_environment`, and `dependencies`. A bundle needs Databricks CLI 1.21.0 or above. Deploy rewrites the path to a workspace path.

The job example pins `environment_version = "6"` and `cowsay==6.1`. The requirements line for both examples is 5 and above. On the environment-versions page, version 5 was released on 25 February 2026 with Databricks Connect 18. Version 6 was released on 3 September 2026 with Connect 19. Both are Python 3.12.3. I would pin one of them in the file and treat the two examples as two pins.

A missing or stale `uv.lock`, for a job that installs only project dependencies, falls back to `uv pip install -r pyproject.toml`. That path skips locked versions and does not install the project package. The page's install path is Hatchling plus a current lockfile, building `src/my_project/__init__.py` for the name `my-project`. Generate the lock with `uv lock`. The check, from a notebook in the project directory, is `%uv lock --check`.

The same page says each job task runs in an isolated environment. Two tasks can name one file and remain two environments. Python wheel tasks refuse a relative path and ask for an absolute workspace path. The `../pyproject.toml` example is a Python file task, with `python_file` at `../src/main.py`, both relative to `resources/my_project.job.yml`. The dbt steps and the JAR steps do not offer Project. A separate Beta, managed base environments, chooses Base environment or an Environment version drop-down. The bundles library page, last updated 11 September, still shows an editable install of the deployed folder, tells pipelines to list dependencies in `pipeline.yml`, and builds with `uv build --wheel`. It never mentions `project_environment`. I would leave a Lakeflow pipeline there.

## Where the pages disagree

The October notes stop at notebook tasks and Python script tasks. Wheel tasks live on the dependencies page. The Jobs API reference I opened still requires `environment_version` or `base_environment`, and it describes `dependencies`, with no `project_environment` field. A notebook or a job can land on `aarch64` or `x86_64`, and that architecture can change between runs, so a native wheel needs a pure-Python build or both builds, marked with `platform_machine`. I did not find a compliance-profile exclusion, a government-cloud exclusion, or a separate meter. A release can take a week or more to reach an account.

<img src="/assets/images/project-environment-one-file-two-rooms.svg" alt="The editor room shares one serverless process per user. The job room gives each task an isolated environment and reads the same pyproject.toml through project_environment. The 22 October notes say Beta. The Jobs API reference still omits the field." title="One file, two rooms. The schedule carries the file. It leaves the process in the editor." style="width:100%; height:auto;">

## Where it fits, and where it does not

I would use the shared editor when several notebooks in one bundle, or one Git folder, should install one locked set, and each person works from their own clone. I would use `project_environment` when the job should install that file, on CLI 1.21.0 or above, with one pinned `environment_version` and a lockfile `%uv lock --check` accepts. Hatchling belongs in the file when the task must import the project package. I would leave a single notebook on the side pane, a value that must cross notebooks in a table, a teammate off the compute I started, and dbt tasks, JAR tasks, and Lakeflow pipelines on the pages that still omit this field.

## What goes in the design doc

Write Beta, off until an admin enables Project-Scoped Shared Compute, with the staged rollout beside it. The editor room is one process per user. The 2 September note is the Git folder predecessor. The job room is an isolated task environment whose only environment-source field is `project_environment`. Name the wheel-path rule, the lockfile fallback, the Connect 18 and Connect 19 pins, and the 11 September pipeline page. Leave the Jobs API reference and the serverless notes in the doc until they list the field. 22 October is a Beta date.

## References

- Databricks, "October 2026" release notes (AWS). Serverless job environments from `pyproject.toml` and Shared Serverless for Asset Bundles are Beta on 22 October. The page header still says 1 October. The Microsoft Entra ID connector is also Beta on 22 October. [https://docs.databricks.com/aws/en/release-notes/product/2026/october](https://docs.databricks.com/aws/en/release-notes/product/2026/october)
- Microsoft, "October 2026" release notes (Azure Databricks). The same 22 October pair. [https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/october](https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/october)
- Databricks, "October 2026" release notes (Google Cloud). The page header still says 1 October. [https://docs.databricks.com/gcp/en/release-notes/product/2026/october](https://docs.databricks.com/gcp/en/release-notes/product/2026/october)
- Databricks, "September 2026" release notes (AWS). Shared Serverless for Git Folders is Beta on 2 September. The note does not mention Project-Scoped Shared Compute. [https://docs.databricks.com/aws/en/release-notes/product/2026/september](https://docs.databricks.com/aws/en/release-notes/product/2026/september)
- Databricks, "Use Shared Serverless for Git folders and bundles" (AWS), last updated 22 October 2026. [https://docs.databricks.com/aws/en/compute/serverless/notebooks/shared-serverless](https://docs.databricks.com/aws/en/compute/serverless/notebooks/shared-serverless)
- Databricks, the Git Folders URL, which returned the same combined page on 26 October 2026. [https://docs.databricks.com/aws/en/compute/serverless/notebooks/git-folder-serverless](https://docs.databricks.com/aws/en/compute/serverless/notebooks/git-folder-serverless)
- Microsoft, "Use Shared Serverless for Git folders and bundles" (Azure Databricks). The banner says Important, and the text says Beta. [https://learn.microsoft.com/en-us/azure/databricks/compute/serverless/notebooks/shared-serverless](https://learn.microsoft.com/en-us/azure/databricks/compute/serverless/notebooks/shared-serverless)
- Databricks, "Use Shared Serverless for Git folders and bundles" (Google Cloud), last updated 22 October 2026. [https://docs.databricks.com/gcp/en/compute/serverless/notebooks/shared-serverless](https://docs.databricks.com/gcp/en/compute/serverless/notebooks/shared-serverless)
- Databricks, "Configure the serverless environment" (AWS), last updated 22 October 2026. Project-environment requirements, the lockfile fallback, the Hatchling example, and the bundle paths. [https://docs.databricks.com/aws/en/compute/serverless/dependencies](https://docs.databricks.com/aws/en/compute/serverless/dependencies)
- Microsoft, "Configure the serverless environment" (Azure Databricks). Python wheel tasks are included, and a relative project-environment path is refused for them. [https://learn.microsoft.com/en-us/azure/databricks/compute/serverless/dependencies](https://learn.microsoft.com/en-us/azure/databricks/compute/serverless/dependencies)
- Databricks, "Configure the serverless environment" (Google Cloud). [https://docs.databricks.com/gcp/en/compute/serverless/dependencies](https://docs.databricks.com/gcp/en/compute/serverless/dependencies)
- Databricks, "Notebook-scoped Python libraries" (AWS), last updated 22 October 2026. [https://docs.databricks.com/aws/en/libraries/notebooks-python-libraries](https://docs.databricks.com/aws/en/libraries/notebooks-python-libraries)
- Databricks, "Declarative Automation Bundles library dependencies" (AWS), last updated 11 September 2026. [https://docs.databricks.com/aws/en/dev-tools/bundles/library-dependencies](https://docs.databricks.com/aws/en/dev-tools/bundles/library-dependencies)
- Databricks, Jobs API reference. The environment `spec` documents `environment_version`, `base_environment`, and `dependencies`. [https://docs.databricks.com/api/jobs/v2](https://docs.databricks.com/api/jobs/v2)
- Databricks, "Serverless compute release notes" (AWS), last updated 22 October 2026. [https://docs.databricks.com/aws/en/release-notes/serverless/](https://docs.databricks.com/aws/en/release-notes/serverless/)
- Microsoft, "Serverless compute release notes" (Azure Databricks). [https://learn.microsoft.com/en-us/azure/databricks/release-notes/serverless/](https://learn.microsoft.com/en-us/azure/databricks/release-notes/serverless/)
- Databricks, "Serverless compute release notes" (Google Cloud), last updated 22 October 2026. [https://docs.databricks.com/gcp/en/release-notes/serverless/](https://docs.databricks.com/gcp/en/release-notes/serverless/)
- Databricks, "Environment versions" (Google Cloud). Version 5 on 25 February 2026 with Connect 18. Version 6 on 3 September 2026 with Connect 19. [https://docs.databricks.com/gcp/en/release-notes/serverless/environment-version/](https://docs.databricks.com/gcp/en/release-notes/serverless/environment-version/)
- Microsoft, "Environment versions" (Azure Databricks). The same version table. [https://learn.microsoft.com/en-us/azure/databricks/release-notes/serverless/environment-version/](https://learn.microsoft.com/en-us/azure/databricks/release-notes/serverless/environment-version/)
