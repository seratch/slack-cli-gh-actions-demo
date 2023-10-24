# Demonstrating Slack CLI Deployment Using GitHub Actions

This tutorial provides guidance on setting up a Slack CLI project to facilitate automatic deployments when code modifications are pushed into the main branch.

## Create a GitHub repo

Let's start by creating a blank GitHub repository for this automation. Feel free to use any name you desire for it, and you don't need to add any files at this time.

## Install Slack's CLI

Before you enable automated deployments, it is necessary to execute several Slack CLI commands on your local machine.
Therefore, you should install the CLI in your terminal.
The following command will install the CLI if you are working on macOS or Linux.
For other platforms, step-by-step instructions can be found in the [Quickstart Guide](https://api.slack.com/automation/quickstart).

```bash
curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash
```

To ensure that there are no issues with the installation, you can execute the `slack doctor` command. If everything is fine, you will see `Errors: 0` at the bottom of the output.

## Create a new app

Run `slack create` command to generate a new Slack app project:

```
$ slack create gh-actions-demo
? Select a template to build from: [Use arrows to move]

❱ Issue submission (default sample)
  Basic app that demonstrates an issue submission workflow

  Scaffolded template
  Solid foundation that includes a Slack datastore

  Blank template
  A, well.. blank project

  View more samples

  Guided tutorials can be found at api.slack.com/automation/samples
```

Select the default sample (Issue submissions) this time:

```
$ slack create gh-actions-demo
? Select a template to build from: Issue submission (default sample)
⚙️  Creating a new Slack app in ~/my-awesome-repo/gh-actions-demo

📦 Installed project dependencies

✨ gh-actions-demo successfully created

🧭 Explore the documentation to learn more
   Read the README.md or peruse the docs over at api.slack.com/automation
   Find available commands and usage info with slack help

📋 Follow the steps below to begin development
   Change into your project directory with cd gh-actions-demo/
   Develop locally and see changes in real-time with slack run
   When you're ready to deploy for production use slack deploy
```

## Obtain the app's service token

To automate the deployment, you need to acquire this app's service token to execute the `slack deploy` command without requiring any interactions on the terminal window.

Go to the root directory of the project and run the `slack auth token` command as illustrated below. You will then be prompted to run a slash command named `/slackauthticket` with an argument in a Slack workspace/organization. Copy this command and implement it in the Slack workspace/organization where you wish to install this app.

```
$ cd gh-actions-demo/
$ slack auth token

📋 Run the following slash command in any Slack channel or DM
   This will open a modal with user permissions for you to approve
   Once approved, a challege code will be generated in Slack

/slackauthticket N2JkOGVjZDAtMTRhZC00NmUwLThjNj******************

? Enter challenge code
```

You will be given a challenge code, which could be a random string such as `iBz5TFyc`.
Input this code in the terminal. You will see a service token (xoxp-...) appearing in the output:

```
$ slack auth token

📋 Run the following slash command in any Slack channel or DM
   This will open a modal with user permissions for you to approve
   Once approved, a challege code will be generated in Slack

/slackauthticket N2JkOGVjZDAtMTRhZC00NmUwLThjNj******************

? Enter challenge code iBz5TFyc

✅ You've successfully authenticated! 🎉
   Service token:

     xoxp-1563*********-1556*********-6081*********-0a32e593af6370******************

   Make sure to copy the token now and save it safely.

💡 Get started by creating a new app with slack create my-app
   Explore the details of available commands with slack help
```

You remember you've created a new GitHub repo as the first step. Head to the repo's settings page **Security** > **Secrets and variables** > **Actions** and click the "New repository secret" button to add a new secret:
* Name: SLACK_SERVICE_TOKEN
* Secret: the `xoxp-` prefixed service token string you've obtained in the preivous step

## Add a GH Actions workflow for deploying the app

Add a new file `.github/workflows/deployment.yml` with the following content:

```yaml
name: Slack App Deployment

# Deploy the latest revision of this repo to Slack platform when:
on:
  push:
    branches: [ main ]

jobs:
  build:
    # You can go with any other Linux options if you prefer
    runs-on: ubuntu-latest

    # The deployment process itself usually takes less than 1 minute
    # When your app's code base is larger in the future, you may want to adjust this duration
    timeout-minutes: 5

    steps:
    - uses: actions/checkout@v4

    # Slack CLI requires Deno runtime
    - name: Install Deno runtime
      uses: denoland/setup-deno@v1
      with:
        # Using the latest stable version along with Slack CLI is recommend
        deno-version: v1.x

    - name: Cache Slack CLI installation
      id: cache-slack
      uses: actions/cache@v3
      with:
        path: |
          /usr/local/bin/slack
          ~/.slack/bin/slack
        key: ${{ runner.os }}-slack
    - name: Install Slack CLI
      if: steps.cache-slack.outputs.cache-hit != 'true'
      run: |
        curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash

    - name: Deploy the app
      env:
        # you can obtain this token string (xoxp-...) by running `slack auth token`
        # Note that this token is connected to a specific Slack workspace/org
        SLACK_SERVICE_TOKEN: ${{ secrets.SLACK_SERVICE_TOKEN }}
      run: |
        # If you place the code in the top directory, there's no need to use this command
        cd gh-actions-demo/
        slack upgrade
        slack deploy --token $SLACK_SERVICE_TOKEN
```

## Push commits to main branch

Push the changes into the main branch and see how it goes.
If the GitHub Actions job successfully completes, your app is now available in the Slack workspace :tada:

## Create a link trigger

Lastly, you need to create a link trigger to use the installed workflow.
This could be a bit confusing but you don't use GitHub Actions to generate triggers this time.
Run `slack trigger create` comamnd with the service token you've obtained as below:

```
$ slack trigger create --token xoxp-...(replace with your token)
⚡ Searching for trigger definition files under 'triggers/*'...
   Found 1 trigger definition file

? Choose a trigger definition file: triggers/submit_issue.ts
Selecting team 'acme-corp' with token belonging to 'acme-corp'...
? Choose an app environment Deployed

📚 App Manifest
   Created app manifest for "gh-actions-demo" in "Acme Corp"

🏠 App Install
   Installing "gh-actions-demo" app to "Acme Corp"
   Updated app icon: assets/default_new_app_icon.png
   Finished in 4.4s

⚡ Trigger successfully created!

   Submit an issue Ft********** (shortcut)
   Created: 2023-10-24 15:06:06 +09:00 (0 seconds ago)
   Collaborators:
     Kazuhiro Sera @kaz U01GCJ79JT0
   Can be found and used by:
     everyone in the workspace
   https://slack.com/shortcuts/Ft**********/36f28af0a74b********************
```

Copy the `slack.com` URL and share it in a Slack channel or on a canvas document.
You will see a button to start the workflow:

<img width="600" src="https://user-images.githubusercontent.com/19658/277564461-c2d692a5-6df4-4e47-9abb-5b477d7d9bcb.png"/>

## Wrap up

Here are the steps you've learned through this tutorial:
* Create a new GitHub repository
* Create a new Slack app using the `slack create` command
* Obtain a service token for a Slack workspace/organization
* Add the service token to the GitHub repository as a secret
* Add GitHub Actions workflow to automate deployments using `slack deploy` with the service token

Your GitHub repository is now set up for team collaboration! Your team members can review code changes by submitting PRs, and once these are merged into the main branch, the changes will be deployed automatically, requiring no manual operations. Should you need to reverse a deployment, you can create a reverting PR and quickly merge that.

Enjoy building workflows using Slack's automation platform!