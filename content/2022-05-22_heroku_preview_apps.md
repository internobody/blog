+++
title = "Heroku Preview Apps without Pipelines"
description = "Create a preview app without Pipelines using Github Actions"
path = "2022/05/22/heroku-preview-apps"
[taxonomies]
tags = ["heroku", "github", "actions"]
+++

If your workflow has been stymied by Heroku's [recent security issue](https://status.heroku.com/incidents/2413) and their subsequent disabling of the Github integration, you no doubt wish for the magic of Preview Apps back.
<!-- more -->

Until the Github integration is re-enabled, however, you can get by adding these Github actions to your workflows. These are heavily cribbed from [Create Heroku app on Github Pull Request](https://remarkablemark.org/blog/2022/04/23/heroku-github-actions-pull-request/?msclkid=d17b60e6c55b11ec9851034a7a2f7816), with some modifications. Instead of creating an app as soon as a PR is submitted, these actions will create one when you add or remove a label named `deploy` to your Pull Request.

## Deploy Preview App
Add a label named `deploy` (or use another label of your choosing) to a PR and the `deploy-preview-app` job will create an app in Heroku. `sync-preview-app` will do a push to an existing Heroku app when new commits are pushed to the PR.

```yaml
# .github/workflows/deploy_preview_app.yml
name: Deploy Preview App
on:
  pull_request:
    types: [labeled, synchronize]

jobs:
  deploy-preview-app:
    runs-on: ubuntu-latest
    if: ${{ github.event.action == 'labeled' && contains(github.event.*.labels.*.name, 'deploy') }}
    env:
      HEROKU_APP_NAME: stage-pr-${{ github.event.number }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          ref: ${{ github.head_ref }}

      - name: Login to Heroku
        uses: akhileshns/heroku-deploy@v3.12.12
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_email: ${{ secrets.HEROKU_EMAIL }}
          heroku_app_name: ${{ env.HEROKU_APP_NAME }}
          justlogin: true

      - name: Create Heroku app
        run: heroku apps:create ${{ env.HEROKU_APP_NAME }} --team=<your team> --stack=<your stack>

      - name: Set up buildpacks
        run: |
          heroku buildpacks:add heroku/nodejs -a ${{ env.HEROKU_APP_NAME }}
          heroku buildpacks:add heroku/ruby -a ${{ env.HEROKU_APP_NAME }}

      - name: Provision addons
        run: heroku addons:create heroku-redis:hobby-dev -a ${{ env.HEROKU_APP_NAME }}

      - name: Add Heroku app to pipeline
        run: heroku pipelines:add <your-pipeline> --app=${{ env.HEROKU_APP_NAME }} --stage=development

      - name: Add Heroku remote
        run: heroku git:remote --app=${{ env.HEROKU_APP_NAME }}

      - name: Push to Heroku
        run: git push heroku ${{ github.head_ref }}:master --force

      - name: Set dyno types
        run: heroku ps:type worker=standard-1x web=standard-1x -a ${{ env.HEROKU_APP_NAME }}

      - name: Restart heroku app
        run: heroku ps:restart -a ${{ env.HEROKU_APP_NAME }}

      - name: Add comment to PR
        run: |
          gh pr comment ${{ github.event.number }} --body '[Heroku app](https://dashboard.heroku.com/apps/${{ env.HEROKU_APP_NAME }}): https://${{ env.HEROKU_APP_NAME }}.herokuapp.com deployed'
        env:
          GITHUB_TOKEN: ${{ secrets.BOT_PERSONAL_ACCESS_TOKEN }}

  sync-preview-app:
    runs-on: ubuntu-latest
    if: ${{ github.event.action == 'synchronize' && contains(github.event.*.labels.*.name, 'deploy') }}
    env:
      HEROKU_APP_NAME: stage-pr-${{ github.event.number }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          ref: ${{ github.head_ref }}

      - name: Login to Heroku
        uses: akhileshns/heroku-deploy@v3.12.12
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_email: ${{ secrets.HEROKU_EMAIL }}
          heroku_app_name: ${{ env.HEROKU_APP_NAME }}
          justlogin: true

      - name: Add Heroku remote
        run: heroku git:remote --app=${{ env.HEROKU_APP_NAME }}

      - name: Push to Heroku
        run: git push heroku ${{ github.head_ref }}:master --force
```

## Destroy Preview App

This workflow will destroy an app either when the PR is closed, the label is removed, or can be run manually through the Github Actions interface.

```yaml
# .github/workflows/destroy_preview_app.yml
name: Destroy Preview App
on:
  pull_request:
    types: [unlabeled, closed]
  workflow_dispatch:
    inputs:
      force_destroy:
        description: 'Enter "force" to force destroy preview app'
        required: false
        default: "no"
      pr_number:
        description: "Specify the PR to destroy"
        required: false
        default: ""

jobs:
  destroy-preview-app:
    runs-on: ubuntu-latest
    if: ${{ github.event.label.name == 'deploy' || (github.event.inputs.force_destroy == 'force' && github.event.inputs.pr_number != '') }}
    env:
      HEROKU_APP_NAME: stage-pr-${{ github.event.number }}

    steps:
      - name: Set Heroku app name
        if: ${{ (github.event.inputs.force_destroy == 'force' && github.event.inputs.pr_number != '') }}
        run: |
          if (github.event.inputs.pr_number != ''); then
            echo "HEROKU_APP_NAME=stage-pr-${{ github.event.inputs.pr_number }}" >> $GITHUB_ENV
          fi

      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          ref: ${{ github.head_ref }}

      - name: Login to Heroku
        uses: akhileshns/heroku-deploy@v3.12.12
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_email: ${{ secrets.HEROKU_EMAIL }}
          heroku_app_name: ${{ env.HEROKU_APP_NAME }}
          justlogin: true

      - name: Scale down app
        run: heroku ps:scale web=0 worker=0 -a ${{ env.HEROKU_APP_NAME }}

      - name: Destroy Heroku app
        run: heroku apps:destroy --app=${{ env.HEROKU_APP_NAME }} --confirm=${{ env.HEROKU_APP_NAME }}

      - name: Add comment to PR
        run: |
          gh pr comment ${{ github.event.number }} --body '[Heroku app](https://dashboard.heroku.com/apps/${{ env.HEROKU_APP_NAME }}): https://${{ env.HEROKU_APP_NAME }}.herokuapp.com destroyed.'
        env:
          GITHUB_TOKEN: ${{ secrets.BOT_PERSONAL_ACCESS_TOKEN }}
```
