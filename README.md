<div align="center">
 name: Run Dependabot
on:
  workflow_dispatch:

jobs:
  run-dependabot:
    permissions:
      # Important not to give Dependabot write access in case it runs arbitrary
      # code as some ecosystems do.
      contents: read
    runs-on: ubuntu-latest
    steps:
        # Checkout code to get the Dependabot CLI input file
      - name: Checkout code
        uses: actions/checkout@v4
        # Checkout to a directory to keep the working dir clean for the update.
        with:
          path: repo

      - name: Download CLI
        env:
          # To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable.
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release download --repo dependabot/cli -p "*linux-amd64.tar.gz"
          tar xzvf *.tar.gz >/dev/null 2>&1
          ./dependabot --version

      - name: Run Dependabot
        env:
          # GITHUB_TOKEN shows an example of how Dependabot CLI can be used with secrets.
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # Run Dependabot CLI with options:
          # -f: the path to the job input
          # --local: use the locally checked out code instead of cloning each time
          # --timeout: the maximum time to wait for a job to finish
          ./dependabot update \
            -f repo/.github/dependabot/go.yml \
            --local repo \
            --timeout 20m >> result.jsonl || true
          ./dependabot update \
            -f repo/.github/dependabot/bundler.yml \
            --local repo \
            --timeout 20m >> result.jsonl || true

      - name: Upload result
        uses: actions/upload-artifact@v4
        with:
          name: dependabot-result
          path: result.jsonl

  create-prs:
    permissions:
      # This job creates PRs, so it needs write access.
      contents: write
      pull-requests: write
    runs-on: ubuntu-latest
    needs: run-dependabot
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download result
        uses: actions/download-artifact@v4
        with:
          name: dependabot-result

      - name: Create PRs
        env:
          # To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable.
          GH_TOKEN: ${{ github.token }}
        run: bash create.sh result.jsonl   <picture>
        <source media="(prefers-color-scheme: light)" srcset="https://user-images.githubusercontent.com/7659/174594540-5e29e523-396a-465b-9a6e-6cab5b15a568.svg">
        <source media="(prefers-color-scheme: dark)" srcset="https://user-images.githubusercontent.com/7659/174594559-0b3ddaa7-e75b-4f10-9dee-b51431a9fd4c.svg">
        <img src="https://user-images.githubusercontent.com/7659/174594540-5e29e523-396a-465b-9a6e-6cab5b15a568.svg" alt="Dependabot" width="336">
    </picture>
</div>

# Example Dependabot CLI Usage

This repo serves as an example of how to use Dependabot CLI for updates. It is intended as a starting point for advanced users to run a self-hosted version of Dependabot within their own projects.

For a hassle-free Dependabot experience, check out the hosted [Dependabot Service](https://docs.github.com/en/github/administering-a-repository/about-dependabot-version-updates).

This repo uses an Action which downloads and runs Dependabot CLI. To run the Action you would go to [the Action in the Actions tab](https://github.com/dependabot/example-cli-usage/actions/workflows/example.yml), and run it.

To see what the results look like, go check out the Pull Requests.

While this example repo runs on GitHub Actions, none of the Dependabot bits are specific to GitHub, so it should be straightforward to port it to any other CI system.

## Implementation details

The Action is defined at [.github/workflows/example.yml](.github/workflows/example.yml).

It contains two jobs, the first downloads and runs Dependabot CLI. The inputs for the CLI runs are in [.github/dependabot](.github/dependabot). See the [Dependabot CLI repo](https://github.com/dependabot/cli) for more info on inputs such as credentials and groupings. 

The results are redirected to a file and uploaded as artifacts.

The second job downloads the artifact and creates PRs from it using the script [create.sh](create.sh).

The reason there are two jobs is Dependabot CLI should only run with read-only tokens as some ecosystems may execute arbitrary code. To achieve that in Actions we must use two jobs with `permissions` defined differently.

Also take a look at the [Dependabot Smoke Tests repo](https://github.com/dependabot/smoke-tests/tree/main/tests) for example inputs and expected outputs.

## Where can I go for help?

If you are having issues with the updates to a specific ecosystem, head over to [dependabot-core](https://github.com/dependabot/dependabot-core).

If there is a problem with running the Dependabot CLI, report that in the [CLI repo](https://github.com/dependabot/cli).

We do not provide direct support for the scripts and workflows in this repo, this is only to serve as an example.

