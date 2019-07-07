<p align="center">
❗❗❗❗❗❗❗❗❗❗❗❗❗❗❗ DO NOT USE YET. Wait for 1.0.0 tag release ❗❗❗❗❗❗❗❗❗❗❗❗❗❗❗❗❗
</p>

You could participate in testing using `@stable` version or any of already released tags from release page.

# Github actions for scoop buckets

Set of automated actions you will ever need as bucket maintainer.

## Implemented actions

### Excavator (`Excavator | Excavate`)

- Periodically execute automatic updates for all manifests

### Pull requests (`Pull requests | PullRequestHandler`)

- When pull request is created there will be executed these tests for all changed manifests.
    - Required properties
        - License
        - Description
    - Hashes
    - Checkver functionality
    - Autoupdate functionality
- All checks could be executed with `/verify` comment. (<https://github.com/Ash258/GithubActionsBucketForTesting/pull/66>)

### Issues (`Issues | IssueHandler`)

- **Hash check fails**
    1. Run checkhashes and check if there were some changes
        1. Yes
            1. List newest pull requests with name `<manifest>: Hash fix`
                1. If there are some
                    1. Select the latest one
                    1. Update PR description with closing directive of new issue
                    1. Comment on issue about this PR
                1. If none
                    1. Create new branch `<manifest>-hash-fix-<random>`
                    1. Commit changes
                    1. Create new PR
        1. No
            1. Comment on issue about hashes being right
            1. Remove label `hash-fix-needed`
            1. Close issue
- **Download failed**
    1. Manifest will be loaded
    1. Iterate in all architectures
        1. Get url for specific architecture
        1. Iterate in all of these urls
            1. Try to download one
            1. If there is error, add current url to list of broken urls
    1. Comment will be posted to issue

## Example workflow for everything you will ever need as bucket maintainer

```hcl
workflow "Issues" {
    on = "issues"
    resolves = ["IssueHandler"]
}

workflow "Pull requests" {
    resolves = ["PullRequestHandler"]
    on = "pull_request"
}

workflow "Pull requests comment" {
    resolves = ["PullRequestHandler"]
    on = "issue_comment"
}

workflow "Excavator" {
    on = "schedule(0 * * * *)" # Run auto-pr every hour, See: https://developer.github.com/actions/managing-workflows/creating-and-cancelling-a-workflow/#scheduling-a-workflow
    resolves = ["Excavate"]
}

action "IssueHandler" {
    uses = "Ash258/Scoop-GithubActions@stable"
    args = "Issue"
    env = {
        "GITH_EMAIL" = "youremail@email.com" # Email is needed for pushing to repository within action container
    }
    secrets = ["GITHUB_TOKEN"]
}

action "PullRequestHandler" {
    uses = "Ash258/Scoop-GithubActions@stable"
    args = "PR"
    env = {
        "GITH_EMAIL" = "youremail@email.com"
    }
    secrets = ["GITHUB_TOKEN"]
}

action "Excavate" {
    uses = "Ash258/Scoop-GithubActions@stable"
    args = "Scheduled"
    env = {
        "SPECIAL_SNOWFLAKES" = "curl,brotli,jx" # Optional parameter
        "SKIP_UPDATED" = "1" # Optional parameter, Could be anything as value
        "GITH_EMAIL" = "youremail@email.com"
    }
    secrets = ["GITHUB_TOKEN"]
}
```

## How to debug locally

Save this snippet as `LocalTestEnvironment.ps1`

```powershell
# Try to avoid all real requests into repository
#    but GithubActionsBucketForTesting so feel free to do whatever you want with this repo
[System.Environment]::SetEnvironmentVariable('GITHUB_TOKEN', '<yourtoken>', 'Process')
[System.Environment]::SetEnvironmentVariable('GITHUB_EVENT_PATH', "$PSScriptRoot\cosi.json", 'Process')
[System.Environment]::SetEnvironmentVariable('GITHUB_REPOSITORY', 'Ash258/GithubActionsBucketForTesting', 'Process')
$DebugPreference = 'Continue'
git clone 'https://github.com/Ash258/GithubActionsBucketForTesting.git' '/github/workspace'
# Uncomment debug entries in Dockerfile
```

Execute `docker run -ti (((docker build -q .) -split ':')[1])`.

## Issues

1. On issues.created
    1. Parse issue title
        1. `manifest@version: PROBLEM`
            1. `extract_dir error`
                1. Extract
                    1. If there is problem
                        1. Add label package-fix-needed and verified
                    1. If no, comment on issue and close it
        1. $env:GITHUB_EVENT_PATH, <https://developer.github.com/actions/creating-github-actions/accessing-the-runtime-environment/#environment-variables>
