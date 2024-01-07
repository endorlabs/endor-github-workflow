# endorlabs-github-supervisor
A GitHub Actions-based Supervisor and Shared Workflow for Endor Labs Scans

This template assumes use of GitHub Cloud; it will require some adaptation to use with Enterprise Server.

> [!WARNING]
> This is a _template_ and is in a pre-release state. You must make a copy and adjust it to your environment, and it may contain bugs as it has not yet been thoroughly tested

## Using the shared workflow on a per-repo basis

Enable GitHub Actions OIDC connection in Endor Labs to use that authentication.

If you have Enterprise Server, see the [scan-with-endorlabs.yml file](.github/workflows/scan-with-endorlabs.yml) for setting up API-based auth, and configure variables there to identify your Enterprise Server URLs/etc.

As a shared workflow, `scan-with-endorlabs.yml` runs as a sidecar job in parallel with your other builds. You can insert a job like the `endor-scan` one below into one of your existing workflow files, or you can create an independent workflow file defining when you'd like to scan a monitored version (recommendation: scan on release or when pushing to a release branch, PR scan on all pull requests), like so:

```yaml
name: Endor Labs Scan
on:
  workflow_dispatch:
  push:
    branches: ["main", "release/*"]
  pull_request:

jobs:
  endor-scan:
    # NOTE -- alter this to point to your fork of this repo!
    uses: YOUR_ORG/endor-github-workflow/.github/workflows/scan-with-endorlabs.yml@main
    permissions:
      id-token: write      # allows authentication to Endor Labs using Actions OIDC JWT Toke
      issues: write        # allows scanner to publish SARIF, if requested
      contents: read       # allows this job to clone org repos
    with:
      namespace: ${{ vars.ENDOR_NAMESPACE }}  # set this as an org-level variable, preferably

      git-url: "https://github.com/${{ github.repository }}.git"
      git-branch: ${{ github.ref_name }}
      upload-sarif: false   # set TRUE to publish SARIF to GitHub; requires public repo or GHAS license; default is false
      upload-json: true     # upload the scan results as a ZIP of a JSON file; default is false
      upload-logs: false    # set TRUE to upload the host-check and scan logs as a ZIP; default is false
      is-pr: ${{ github.event_name == 'pull_request' && true || false }}
```

## Using the whole-org scanner

See [whole-org-scan.yml](.github/workflows/whole-org-scan.yml) for an example.

A whole-org scan leverages the [find-unscanned.yml](.github/workflows/find-unscanned.yml) shared workflow to find existing projects in Endor Labs that haven't had a recent scan (defauult: last 24 hours), and don't have a scan in progress. If you have more than 255 such projects, this will only return the first 255 (due to GitHub Matrix limits). However, we _strongly recommend_ that you reduce this limit to match the number of parallel scans you're willing to perform in your matrix strategy.

Running a whole-org scan on, say, an hourly basis will result in incrementally scanning your entire org. If you have a very large number of repos, be sure to think through your frequency carefully, to avoid unnecessary runner costs.

A few notes about following this example:

1. recent and in-progress scans will mean a repo is ignored; this allows for incremental scans
2. a side effect of (1) is that if you have an active repo that's regularly scanning, the whole-org scan will skip it. This is intentional, and is almost certainly what you want to happen
3. as configured here, a PR scan will count as a scan -- if a project is doing PR scans, they've probably integrated and don't need to be in the whole-org scan process

### adding new projects to your whole-org scan

You can perform an org sync with an option, which will identify any new projects in your org and add them to Endor Labs; once added, they'll then be scanned as they won't have a recent scan associated

However, we _do not recommend_ peforming this action on every whole-org scan execution. It won't break anything, but it's a waste of resources and also causes problems for short-lived projects or projects just spooling up that are unlikely to get successful scans yet anyhow.

### per-project overrides

You can set `ENDOR_` environment variables by adding them in the `ENDOR_NAME=value` format, one per line, in a given repo at `.endorlabs/environment`; these settings will override general configuration done by the shared workflow, and can be used to add includes/excludes, provide location hints, etc. See the Endor Labs documenation for information about relevant environment variables for controlling the scan.

## runner configuration

In order for the auto-scan to work, the runner must have the toolchains needed to build your projects. While the default `ubuntu-22.04` has several common language and packaging environments already configured, you will likely need to modify the workflow to set up proper environments for your scans.

For example, if you scan projects that use java and Gradle, the runner used will have to have java and Gradle installed, in versions that will allow your project to build with 'gradle build' or 'gradlew', and which Endor Labs supports.

The best way to have maximum success in the shared-workflow mode is to ensure your project has a runner that's _pre-configured_ with the toolchains it needs, and have both its build and the scan job use the same runner.

The best way is not always feasible, so the alternative is to establish a "scanning runner" and modify the shared workflow to install components that are likely to work for most projects in your organization; then, for projects that cannot scan with that runner, eschew the shared workflow in favor of using the [Endor Labs Github Action](//github.com/endorlabs/github-action) directly, as a step within a build-and-test job.

### using self-hosted runners

You may be able to add a `runner:` line to your job `with:` stanza to specify your self-hosted runner, as this value gets passed to `runs-on:` in the shared workflow. However, it's a better idea to modify the shared workflow to use a reasonable default runner; your individual projects can specify a different one if needed.

The self-hosted runner used for scanning has all the same configuration considerations as GitHub-hosted runners, above

