# GitHub Action 同步

## 说明

这个Action用于同步模板文件到组织的所有仓库中，支持拆分配置文件，以实现增量同步。
需提前配置好git使用的ssh密钥，配置方法见例子

## 输入

```yaml
inputs:
  files:
    description: "config files"
    required: true
  message:
    description: "commit message"
    required: false
    default: "chore: Sync by .github"
```

## 例子

```yaml
name: Sync
on:
  push:
    paths:
      - ".github/workflows/sync.yml"
      - "repos/**"
      - "workflow-templates/**"
  workflow_dispatch:
    inputs:
      dry_run:
        description: "dry run"
        required: false
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          fetch-depth: 0
      # 判断是否更改了配置文件，根据变动的配置文件增量同步
      - name: Get changed configs
        id: changed-configs
        uses: tj-actions/changed-files@v16
        with:
          separator: " "
          files: repos/**
      
      # 判断是否更改了模板文件或工作流文件，如果更改了模板文件，则使用全量同步
      - name: Get changed files
        id: changed-files
        uses: tj-actions/changed-files@v16
        if: steps.changed-configs.outputs.any_changed != 'true'
        with:
          files: |
            workflow-templates/**
            .github/workflows/sync.yml

    - name: Git config
        env:
          SSH_KEY: ${{secrets.SYNC_SSH_KEY}}
          KNOWN_HOSTS: ${{secrets.SYNC_SSH_KNOWN_HOSTS}}
        run: |
          mkdir ~/.ssh
          echo "$KNOWN_HOSTS" > ~/.ssh/known_hosts
          echo "$SSH_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          git config --global user.name sync-bot
          git config --global user.email sync-bot@deepin.org
      
      // 增量同步
      - name: Sync changed configs
        uses: linuxdeepin/action-sync@main
        if: steps.changed-configs.outputs.any_changed == 'true'
        with:
          files: "${{ steps.changed-configs.outputs.all_changed_files }}"
          message: "chore: Sync by peeweep-test/.github"

      - name: Get all configs
        id: all-configs
        if: steps.changed-files.outputs.any_changed == 'true'
        run: |
          all_configs=`find repos -type f | xargs`
          echo all configs $all_configs
          echo "::set-output name=ALL_CONFIGS::$all_configs"
      // 全量同步
      - name: Sync all files
        uses: linuxdeepin/action-sync@main
        if: steps.changed-files.outputs.any_changed == 'true'
        with:
          files: "${{ steps.all-configs.outputs.ALL_CONFIGS }}"
          message: "chore: Sync by peeweep-test/.github"
```

## 同步配置

在 repos 中创建任意文件，文件内容格式为 JSON,例子如下

```json
[
  {
    "src": "workflow-templates/check.yml",
    "dest": "peeweep-test/test-action/.github/workflows/check.yml",
    // 可选项，默认同步文件到所有分支
    "branches": ["main"],
    // 可选项，是否从仓库删除文件
    "delete": false,
  }
]
```

以上配置将 workflow-templates/check.yml 同步到 peeweep-test/test-action 仓库的 .github/workflows 目录下。

虽然配置可写在任意位置，但为了便于维护，建议以仓库路径为文件名

例如同步到 peeweep-test/test-action 仓库的配置文件建议放置到 repos/peeweep-test/test-action.json
