# 🛡️ domain-repo-monitor
Keeps an eye out for sneaky repo clones trying to steal your domain’s identity. A plug-and-play GitHub Actions watchdog for your name.

This repository contains a GitHub Actions workflow for monitoring **unauthorized copies of domain-named repositories**. It is designed to help independent researchers, creators, and developers detect misuse, impersonation, or license violations of their publicly published repositories.

---

## Purpose
Many researchers and creators publish content tied to a personal or project domain (e.g., `www.example.com`). When these repositories are cloned without attribution—or worse, used to impersonate the original author—it can create reputational and security risks.

This workflow:
- Scans GitHub for repositories with names matching key domain patterns
- Checks if they contain identity-signaling files (`CNAME`, `LICENSE`, or `index.md`)
- Opens a GitHub issue if a potential copy is found

---

## Files Included
- `.github/workflows/monitor.yml` – GitHub Actions workflow (see below)
- `LICENSE` – Released under the MIT License

> 🔗 **Live reference implementation:**  
> See it in use at  
> [rdcj-research.github.io – monitor.yml](https://github.com/rdcj-research/rdcj-research.github.io/blob/main/.github/workflows/monitor.yml)

---

## How to Use
1. Copy the example workflow below to a `monitor.yml` file to `.github/workflows/` in your own repository
2. Edit the list of target domain names here:
   ```bash
   declare -a targets=("www.example.com" "www.anotherdomain.org")
   ```
3. Replace the login exclusion check with your GitHub username or use:
   ```bash
   select(.owner.login != "${{ github.repository_owner }}")
   ```
4. Commit and push


## Setup Instructions (External Cron + GitHub Action)
**If needed, Uuse an external cron job to trigger this daily.**

### 1. Create a GitHub Personal Access Token
- Go to [https://github.com/settings/tokens](https://github.com/settings/tokens)
- Create a token (workflow → Read and write)
- Copy the token (you’ll only see it once)

---

### 2. Set Up a Cron Job (e.g. [cron-job.org](https://cron-job.org))
- Go to [https://cron-job.org](https://cron-job.org)
- Sign up and create a new cron job

**Request:**
- Method: `POST`
- URL: https://api.github.com/repos/YOUR_USERNAME/YOUR_REPO/actions/workflows/monitor.yml/dispatches
- Headers:
- `Accept: application/vnd.github+json`
- `Authorization: Bearer YOUR_PERSONAL_ACCESS_TOKEN`
- `Content-Type: application/json`
- Body:
```json
{
  "ref": "main"
}
```

Schedule: Daily at your preferred time

---

## Example Workflow
```yaml
name: Monitor for Suspicious Repo Copies

on:
  workflow_dispatch:

jobs:
  check-repos:
    runs-on: ubuntu-latest

    steps:
      - name: Search GitHub for similar repo names
        run: |
          echo "🔍 Searching for suspicious clones..."

          # ⬇️ Customize with your own domain-based repository names
          declare -a targets=("example.github.io" "my-personal-site" "project-website")

          touch matches.txt

          for repo_name in "${targets[@]}"; do
            echo "🔍 Searching for $repo_name..."
            curl -s -H "Accept: application/vnd.github+json" \
              "https://api.github.com/search/repositories?q=${repo_name}+in:name" > search_results.json

            jq -r '.items[] | select(.owner.login != "${{ github.repository_owner }}") | .full_name' search_results.json >> matches.txt
          done

          sort -u matches.txt -o matches.txt
          touch flagged_repos.txt

          while read repo; do
            echo "🔎 Checking $repo..."

            for file in CNAME LICENSE index.md; do
              status=$(curl -s -o /dev/null -w "%{http_code}" \
                -H "Accept: application/vnd.github+json" \
                "https://api.github.com/repos/$repo/contents/$file")

              if [ "$status" = "200" ]; then
                echo "⚠️ Found $file in $repo"
                echo "https://github.com/$repo (contains $file)" >> flagged_repos.txt
                break
              fi
            done

          done < matches.txt

          if [[ -s flagged_repos.txt ]]; then
            echo "found=true" >> $GITHUB_ENV
          else
            echo "found=false" >> $GITHUB_ENV
          fi

      - name: Open issue if clones are flagged
        if: env.found == 'true'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const matches = fs.readFileSync('flagged_repos.txt', 'utf8');

            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: "⚠️ Potential copy detected (CNAME / LICENSE / index.md)",
              body: `The following repositories were found with names similar to your identity or domain, and contain at least one of the following files: \`CNAME\`, \`LICENSE\`, or \`index.md\`.

${matches}

Please review them for potential misuse or unauthorized replication.

_Automated scan triggered by your GitHub Actions monitor._`
            });
```

---

## Why It Matters
This tool supports:
- Enforcement of attribution under licenses like CC BY 4.0
- Early warning of impersonation or malicious copies
- A lightweight way to preserve research trust online

---

## License
This project is released under the [MIT License](LICENSE).  
Feel free to adapt or extend it for your own monitoring needs.

---

## Questions or Feedback?
Developed by Rodrigo Jazinski as part of a broader symbolic and epistemological research initiative.  
More at [rdcjazinski.com](https://www.rdcjazinski.com).

