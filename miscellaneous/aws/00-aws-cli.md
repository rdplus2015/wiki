# AWS CLI — Setup & Concepts

Theory and rationale behind AWS CLI usage: what it is, why it matters, and how
identity/credentials work under the hood.

---

## 1. Overview

The AWS CLI (Command Line Interface) is a program that lets you interact with
AWS services by typing commands instead of clicking through the web console.
Every command you run is translated into one or more calls to the AWS API,
signed with your credentials, and sent over HTTPS.

---

## 2. Terminal, shell, Bash, AWS CLI, CloudShell — untangling the layers

These five terms get used almost interchangeably in casual conversation, but
they are five different things stacked on top of each other. Understanding
the separation matters because it explains *where* your credentials live,
*what* persists between sessions, and *why* the same command can behave
differently depending on where you type it.

| Layer | What it actually is | Analogy |
|---|---|---|
| **Terminal** | The application/window that displays text input and output. It has no logic of its own — it's just a screen + keyboard interface. | The phone itself |
| **Shell** | The program that reads what you type, interprets it, and executes it. Examples: `bash`, `zsh`, `sh`, PowerShell. | The person you're talking to on the phone |
| **Bash** | One specific, very common shell implementation (Bourne Again Shell). When people say "shell" on Linux/macOS, they usually mean Bash by default. | A specific person's way of speaking |
| **AWS CLI** | A program *installed inside* a shell. It's not a shell itself — it's a tool you invoke by typing `aws ...`, just like `git` or `curl`. It translates your command into signed AWS API calls. | A specific topic you can ask that person about |
| **AWS CloudShell** | A full environment: AWS-hosted, browser-based terminal + shell (Bash) + AWS CLI pre-installed and pre-authenticated with your console identity — no local install needed. | A phone AWS lends you, pre-configured, with the right person already on the line |

---

## 3. Why it matters

---

## 4. Use cases

- **Automation & scripting** — any repeatable infra task (spinning up resources, syncing files, tagging) that would be tedious or error-prone by hand in the console.
- **CI/CD pipelines** — deployment steps in GitHub Actions, GitLab CI, etc. call the AWS CLI (directly or via SAM/Terraform/CDK) using credentials injected as environment variables or an assumed role.
- **Multi-account / multi-client work** — a freelancer or agency managing several AWS accounts (personal, client A, client B, training sandbox) relies on named profiles to switch contexts safely without mixing them up.
- **Quick debugging** — checking a resource's current state (`aws s3 ls`, `aws lambda get-function`, `aws logs tail`) is often faster from the CLI than navigating multiple console pages.
- **Emergency/no-local-setup access** — CloudShell lets you run CLI commands from any browser, already authenticated, without configuring anything locally — useful when you're not on your own machine.

*Example from practice*: verifying the active identity (`aws sts get-caller-identity --profile <name>`) before every `sam deploy` on a project, specifically to avoid deploying against the wrong AWS account — a habit built after working with AWS Academy sandbox accounts, where the account ID silently changes every session.

---

## 5. Installation

---

## 6. Identity & credentials model

---

## 7. Configuration

---

## 8. Temporary credentials

---

## 9. References

- [AWS CLI v2 user guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
- [AWS CLI configuration basics](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
- [Understanding and getting your AWS credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)
- [AWS STS overview](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html)
- [AWS CloudShell user guide](https://docs.aws.amazon.com/cloudshell/latest/userguide/welcome.html)
