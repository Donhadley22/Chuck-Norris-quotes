# Chuck Norris Quotes

![Node.js](https://img.shields.io/badge/Node.js-20.x-339933?style=for-the-badge&logo=node.js&logoColor=white)
![Express](https://img.shields.io/badge/Express.js-4.x-000000?style=for-the-badge&logo=express&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![AWS S3](https://img.shields.io/badge/AWS-S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)
![OWASP](https://img.shields.io/badge/OWASP-Dependency--Check-000000?style=for-the-badge&logo=owasp&logoColor=white)

A small Node.js and Express application that displays random Chuck Norris jokes from the public Chuck Norris API. The project includes a GitHub Actions CI/CD pipeline that validates, scans, packages, uploads, and deploys the app to an AWS EC2 instance.

## Table of Contents

- [Application Overview](#application-overview)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Run Locally](#run-locally)
- [CI/CD Pipeline](#cicd-pipeline)
- [Required GitHub Secrets](#required-github-secrets)
- [AWS OIDC Authentication](#aws-oidc-authentication)
- [EC2 Requirements](#ec2-requirements)
- [Deployment Flow](#deployment-flow)
- [Troubleshooting](#troubleshooting)

## Application Overview

The application serves a simple web page that renders a random joke and allows the user to fetch another joke without refreshing the page.

Main routes:

| Route | Method | Description |
| --- | --- | --- |
| `/` | `GET` | Renders the home page with a random joke. |
| `/joke` | `GET` | Returns a random joke as JSON. |

## Tech Stack

- Node.js
- Express.js
- Pug templates
- Axios
- Morgan logging
- PM2 on EC2
- GitHub Actions
- AWS S3
- AWS EC2
- AWS IAM OIDC
- OWASP Dependency-Check

## Project Structure

```text
.
+-- .github/
|   +-- workflows/
|       +-- ci-cd-ec2.yml
+-- public/
|   +-- css/
|   |   +-- style.css
|   +-- javascript/
|       +-- script.js
+-- views/
|   +-- default.pug
|   +-- home.pug
+-- package-lock.json
+-- package.json
+-- README.md
+-- server.js
```

## Run Locally

Clone the repository:

```bash
git clone https://github.com/Donhadley22/Chuck-Norris-quotes.git
cd Chuck-Norris-quotes
```

Install dependencies:

```bash
npm install
```

Start the app:

```bash
npm start
```

Open the app:

```text
http://localhost:3000
```

Run the available test command:

```bash
npm test
```

Run a syntax check:

```bash
node --check server.js
```

Run a dependency audit:

```bash
npm audit --audit-level=high
```

## CI/CD Pipeline

The pipeline is defined in:

```text
.github/workflows/ci-cd-ec2.yml
```

It runs automatically on pushes to `main` and can also be started manually from GitHub Actions.

Pipeline stages:

| Stage | Name | Purpose |
| --- | --- | --- |
| 1 | Checkout source | Pulls the repository into the GitHub runner. |
| 2 | Setup Node.js | Installs Node.js 20 and enables npm caching. |
| 3 | Verify Node and npm | Prints runtime versions. |
| 4 | Install dependencies | Runs `npm ci`. |
| 5 | Validate application syntax | Runs `node --check server.js`. |
| 6 | Run unit tests | Runs `npm test`. |
| 7 | Dependency security scan | Runs `npm audit --audit-level=high`. |
| 8 | OWASP Dependency-Check | Scans dependencies for known CVEs and fails on CVSS 7+. |
| 9 | Upload OWASP report | Uploads the OWASP report as a workflow artifact. |
| 10 | SonarQube analysis | Optional static analysis when Sonar secrets are configured. |
| 11 | SonarQube quality gate | Optional quality gate when Sonar secrets are configured. |
| 12 | Archive build artifact | Creates a compressed app artifact. |
| 13 | Upload GitHub build artifact | Stores the artifact in GitHub Actions. |
| 14 | Configure AWS credentials | Uses GitHub OIDC to assume an AWS IAM role. |
| 15 | Upload artifact to S3 | Uploads the compressed artifact to S3. |
| 16 | Deploy to target EC2 | SSHs into EC2, downloads the artifact, installs dependencies, and starts the app with PM2. |

## Required GitHub Secrets

Add these secrets in:

```text
GitHub repository > Settings > Secrets and variables > Actions
```

| Secret | Required | Description |
| --- | --- | --- |
| `AWS_ROLE_TO_ASSUME` | Yes | IAM role ARN assumed by GitHub Actions using OIDC. |
| `AWS_REGION` | Yes | AWS region, for example `us-east-1`. |
| `S3_BUCKET` | Yes | S3 bucket name only. Do not include `s3://`. |
| `EC2_HOST` | Yes | EC2 public IP address or DNS name. |
| `EC2_USER` | Yes | SSH user, for example `ubuntu`. |
| `EC2_SSH_PRIVATE_KEY` | Yes | Private key matching the EC2 authorized public key. |
| `EC2_PORT` | No | SSH port. Defaults to `22`. |
| `EC2_APP_DIR` | No | App deployment directory. Defaults to `/opt/chucknorris`. |
| `APP_PORT` | No | Application port. Defaults to `3000`. |
| `SONAR_HOST_URL` | No | SonarQube or SonarCloud URL. |
| `SONAR_PROJECT_KEY` | No | Sonar project key. |
| `SONAR_TOKEN` | No | Token for Sonar analysis. |

## AWS OIDC Authentication

This project uses GitHub OIDC instead of long-lived AWS access keys.

The workflow requires:

```yaml
permissions:
  contents: read
  id-token: write
```

The AWS credentials action assumes the configured IAM role:

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
    role-session-name: github-actions-chucknorris-${{ github.run_id }}
    aws-region: ${{ secrets.AWS_REGION }}
```

The IAM role trust policy should restrict access to this repository and branch:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:Donhadley22/Chuck-Norris-quotes:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

The OIDC role needs S3 access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::YOUR_BUCKET/chucknorris/*"
    }
  ]
}
```

`s3:GetObject` is required because the workflow creates a pre-signed S3 URL for the EC2 instance to download the artifact.

## EC2 Requirements

The target EC2 instance should have:

- Ubuntu or another Linux distribution
- SSH access enabled
- Node.js 20 or newer
- npm
- curl
- tar
- outbound internet access

Install Node.js 20 on Ubuntu:

```bash
sudo apt update
sudo apt install -y curl tar
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node --version
npm --version
```

The workflow installs PM2 automatically if it is missing:

```bash
sudo npm install -g pm2
```

Security group rules:

- Allow inbound SSH on port `22` or the configured `EC2_PORT`.
- Allow inbound application traffic on port `3000` or the configured `APP_PORT`.
- Allow outbound internet access for npm installs and external API calls.

## Deployment Flow

The deployment does not require AWS credentials on the EC2 instance.

Flow:

```text
GitHub Actions assumes AWS role with OIDC
        |
        v
GitHub Actions uploads artifact to S3
        |
        v
GitHub Actions creates a 15-minute pre-signed S3 URL
        |
        v
GitHub Actions connects to EC2 with SSH
        |
        v
EC2 downloads artifact with curl
        |
        v
EC2 extracts release into /opt/chucknorris/releases/<commit-sha>
        |
        v
EC2 updates /opt/chucknorris/current symlink
        |
        v
PM2 starts or restarts the app
        |
        v
Health check validates http://127.0.0.1:3000/
```

## Troubleshooting

### `fatal error: Unable to locate credentials`

This happens if the EC2 instance tries to run `aws s3 cp` without AWS credentials.

Current fix:

- The workflow now generates a pre-signed S3 URL in GitHub Actions.
- EC2 downloads the artifact with `curl`.
- EC2 does not need AWS credentials for artifact download.

### `curl: (22) The requested URL returned error: 403`

This means S3 rejected the pre-signed URL.

Common cause:

- The GitHub OIDC role has `s3:PutObject` but does not have `s3:GetObject`.

Fix:

```json
"Action": [
  "s3:PutObject",
  "s3:GetObject"
]
```

### App is not reachable in the browser

Check:

- PM2 process is running.
- EC2 security group allows inbound traffic on the app port.
- The app is listening on the expected port.
- The EC2 instance has outbound access to `https://api.chucknorris.io`.

Useful EC2 commands:

```bash
pm2 status
pm2 logs chucknorris
curl http://127.0.0.1:3000/
```

## License

This project is intended for DevOps CI/CD learning and demonstration.
