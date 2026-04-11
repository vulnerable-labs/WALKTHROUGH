# The Leaky Pipeline

This guide outlines the intended path to solve the lab. It is written from the perspective of an external attacker without prior knowledge of the internal configuration.

## Step 1: Reconnaissance

The engagement begins with a standard port scan against the target IP address. This reveals that port 80 is open and hosting an HTTP web server.

Navigating to the website in a browser displays the public-facing landing page for a fictional startup called Pipeline Dynamics. The site appears to be a static marketing page offering automated CI/CD solutions.

While standard interaction with the site doesn't reveal any obvious input vectors or vulnerabilities, running a directory brute-forcing tool (like Gobuster or DirBuster) against the web root uncovers an exposed `.git/` directory. This is a common and critical misconfiguration where the web root was initialized as a Git repository and pushed directly to production.

## Step 2: Extracting the Source Code

With the `.git` directory exposed, the entire source code and version history of the website can be downloaded. 

Using a tool specifically designed for this purpose, like `git-dumper`, you can pull the repository to your local machine for analysis:

```bash
git-dumper http://<TARGET-IP>/.git/ local-source-code
```

Once the download is complete, inspecting the `local-source-code` directory reveals several internal files. Among the standard HTML and CSS assets, there are two items of significant interest:

1. **The CI Configuration:** A `.gitlab-ci.yml` file, which defines how the startup's code is built, tested, and deployed.
2. **The Git Configuration:** Inside the `.git/config` file, the `origin` remote URL indicates that the developers are pushing their code to an internal Git server located at `http://<TARGET-IP>:3000/dev-admin/startup-website.git`.

## Step 3: Analyzing the Pipeline

Reviewing the `.gitlab-ci.yml` file is the next logical step. The file contains a few concerning configuration blocks:

```yaml
services:
  - docker:dind

test_site:
  stage: test
  script:
    - echo "Running security and linter checks..."
    - echo "Check passed."
  only:
    - merge_requests
```

The presence of `docker:dind` (Docker-in-Docker) heavily implies that the CI runner operates with privileged containers, which often grant extensive access to the host environment or sensitive secrets. Furthermore, the `test_site` job is configured to run automatically whenever a new `merge_requests` (Pull Request) is opened. 

## Step 4: Exploiting the Pipeline

With the knowledge that a CI runner executes code upon receiving a merge request, the objective shifts to interacting with the internal Git server. 

1. Navigate to `http://<TARGET-IP>:3000` in a web browser. This brings up the login page for Gitea, a lightweight Git server.
2. Since the platform allows public registration, click "Register" and create a dummy account.
3. Once logged in, browse the public repositories to find `dev-admin/startup-website`. This is the upstream repository feeding the production website.
4. Fork the repository into your own account workspace.
5. In your forked copy, edit the `.gitlab-ci.yml` file. You need to inject a command that forces the CI runner to print its internal environment variables, as this is where cloud secrets are often temporarily stored during deployment tasks. 

Modify the script block of the `test_site` job to include the `env` command:

```yaml
test_site:
  stage: test
  script:
    - env
  only:
    - merge_requests
```

6. Commit the changed file to your fork.
7. Open a Pull Request from your repository back to the original `dev-admin/startup-website` repository.

## Step 5: Capturing the Flags

As soon as the Pull Request is submitted, a webhook silently triggers the backend custom CI bot. The bot clones your modified code, reads the `.gitlab-ci.yml` instructions, and spins up a privileged Docker container to execute the `env` command.

Because the CI pipeline is highly automated, the bot captures the output of the test scripts and posts it directly back to the Pull Request as a comment.

Navigating to the comments section of your Pull Request will reveal the raw command line output, which dumps the entire environment context of the runner. Scanning through this text reveals the production secrets injected into the CI environment, presenting the two required flags:

```text
GCP_SERVICE_ACCOUNT_KEY=VulnOS{pipeline_leak_gcp_token_x99}
PROD_DB_PASSWORD=VulnOS{db_pwned_via_ci_cd_42}
```
