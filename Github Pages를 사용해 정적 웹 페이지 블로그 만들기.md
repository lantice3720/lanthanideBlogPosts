---
title: Github Pages를 사용해 정적 웹 페이지 블로그 만들기
description: Github Pages에 Hugo의 정적 웹 페이지를 올려 개인용 블로그를 만들어 보자.
tags:
  - GithubPages
  - Blog
categories:
  - Blog
keywords:
  - Github
  - Github Pages
  - Hugo
date: 2024-07-11T04:36:44+09:00
draft: false
weight: 10
build:
  list: always
  publishResources: true
  render: always
---
# 기술 소개

## Github Pages
Github에서 제공하는 기능으로, 간단한 웹사이트를 deploy할 수 있다. 웹서버를 따로 구동할 수가 없어 정적 웹페이지를 만들어야 한다.<br>
따라서 Jekyll, Hugo 등을 이용하게 되는데 필자는 몇 달 전 Jekyll로 블로그를 만들었으나 후술할 문제점들 때문에 Hugo를 사용하기로 했다.

## Github Actions
다양한 작업들을 자동화해주는 기능이다. 코드 lint나 간단한 디버그, deploy에 자주 사용된다.<br>
이번에는 자동으로 submodule을 업데이트하고 page를 생성하는 데 사용할 것이다.

## Hugo
[Hugo 공식 웹사이트](https://gohugo.io/)
정적 웹페이지를 만들어주는 툴이다. Jekyll과는 달리 Ruby 대신 Go를 사용하며, 무엇보다 **git submodule을 제대로 인식한다.**<br>
Jekyll의 경우 알 수 없는 이유로 submodule 사용 시 `_posts` 폴더를 정상적으로 읽어오지 못했고, 이에 Hugo를 사용하기로 결정했다.<br>
테마의 경우 다양한 것이 있었으나 [Stacks](https://themes.gohugo.io/themes/hugo-theme-stack/ )가 깔끔하고 기능이 다양해 이것으로 결정하였다.

# 프로젝트 구성
프로젝트의 목표는 Obsidian 에디터에서 포스팅을 편집 후 Git 플러그인을 사용해 Github에 포스팅을 업로드하고, 최종적으로 Github Pages에 deploy까지 자동으로 완료하는 것이다.<br>
이를 위해 `BlogPosts` 레포와 `<username>.github.io` 레포를 생성하고, git submodule을 사용해 `BlogPosts`가 `content/posts` 경로에 들어가도록 한다.<br>
그리고 `BlogPosts`에 push가 들어오면 Github Actions를 통해 REST 요청을 하고, `<username>.github.io`의 Actions 중 submodule을 업데이트하는 것과 Github Pages에 deploy하는 것이 실행되도록 한다.

### 첨부


## 레포 생성
미리 `<username>.github.io` 와 `BlogPosts` 레포를 생성해 두자. 중요한 건 아니지만 미리 만들어두면 차근차근 진행할 수 있어 순서를 헷갈리는 일이 적어질 것이다.

## Hugo, Github Pages 셋업
[Quick Start](https://gohugo.io/getting-started/quick-start/) 문서를 참고해 진행하면 된다. 여기서 설치 시 extended 버전을 사용하는 것이 편리했다.<br>
테마 적용도 끝났다면 `git init` 후에 `git submodule add <레포> content/posts` 를 입력하자. 이 때 posts 폴더를 지워야 submodule 추가가 가능하니 git 작업 전 미리 지워두는 편을 추천한다.<br>
git remote 설정까지 끝났다면 `.github/workflows` 폴더를 만들고 yml 파일 둘을 추가하자. 하나는 Github Pages에 deploy하는 역할, 하나는 submodule을 업데이트하는 역할이다.<br>

`deploy_pages.yml`
```yml
# Sample workflow for building and deploying a Hugo site to GitHub Pages
name: Deploy Hugo site to Pages

on:
  # Runs on pushes targeting the default branch
  push:
    branches:
      - <branch name>

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.128.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb          
      - name: Install Dart Sass
        run: sudo snap install dart-sass
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"
      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: America/Los_Angeles
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"          
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```
몇몇 예시들을 혼합한 코드이다. 구문을 하나하나 분석해 보자.<br>
- `on:` 은 해당 action이 실행되는 조건을 설정한다. 위의 경우 `<branch_name>` 브랜치에 push가들어올 때 실행하며, 수동/REST를 이용한 실행도 가능하다.
- `permissions:` 는 아래의 작업들을 위한 권한을 미리 명시한다.
- `concurrency:` 는 확인해보진 않았으나, 이 action을 동시에 여러 개 실행할 수 있는지를 결정한다.
- `defaults:` 는 모든 명령이 bash에서 실행되는 것을 기본 설정으로 둔다.
- **`jobs:` 가 바로 핵심이다.** 이 action은 build와 deploy의 두 개의 job으로 구성된다.

먼저 build에서는 Hugo와 Dart Sass의 설치 후 Github에서 제공하는 action을 실행하고 Hugo 빌드 후 artifact로 만들어 둔다.<br>
deploy에서는 이 artifact를 또한 Github에서 제공하는 action을 통해 Github Pages에 deploy 한다.

`submodule_sync.yml`
```yml
name: 'Submodules Sync'

on:
  # Allows you to run this workflow manually from the Actions tab or through HTTP API
  workflow_dispatch:

jobs:
  sync:
    name: 'Submodules Sync'
    runs-on: ubuntu-latest

    # Use the Bash shell regardless whether the GitHub Actions runner is ubuntu-latest, macos-latest, or windows-latest
    defaults:
      run:
        shell: bash

    steps:
    # Checkout the repository to the GitHub Actions runner
    - name: Checkout
      uses: actions/checkout@v2
      with:
        token: ${{ secrets.CI_TOKEN }}
        submodules: true

    # Update references
    - name: Git Submodule Update
      run: |
        git pull --recurse-submodules
        git submodule update --remote --recursive

    - name: Commit update
      run: |
        git config --global user.name 'Git bot-lanthanide'
        git config --global user.email 'bot@noreply.github.com'
        git remote set-url origin https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/${{ github.repository }}
        git commit -am "Auto updated submodule references" && git push || echo "No changes to commit"
```
이번 action은 submodule을 업데이트해 새로운 post를 읽어오는 역할을 수행한다. `on:`을 보면 알 수 있듯 REST API를 통한 실행이 가능해 `BlogPosts` 레포의 action이 이 action을 실행할 것이다.<br>
마지막 줄에서 보이듯 git push를 수행하는데, 이에 따라 deploy를 위한 action의 실행 조건이 충족되어 연쇄적으로 action의 실행이 일어난다.

## 포스팅 관리
포스팅 목록을 업데이트하고 빌드해 deploy까지 하는 시스템이 완성되었으니 이제 이 포스팅이 업데이트될 때에 deploy 시스템의 시작점이 되어줄 REST 요청을 넣어주어야 한다.<br>
이것을 물론 수동으로 할 수도 있겠지만, 위에서 보았다시피 Github Actions는 push를 감지할 수 있으므로 `BlogPosts`가 업데이트 될 때, 즉 push가 일어날 때 자동으로 REST 요청을 넣는 action을 작성해 보자.

`notify_update.yml`
```yml
name: 'Submodule Notify Parent'

on:
  push:
    branches:
      - main    

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

jobs:
  notify:
    name: 'Submodule Notify Parent'
    runs-on: ubuntu-latest

    # Use the Bash shell regardless whether the GitHub Actions runner is ubuntu-latest, macos-latest, or windows-latest
    defaults:
      run:
        shell: bash

    steps:
    - name: Github REST API Call
      env:
        CI_TOKEN: ${{ secrets.CI_TOKEN }}
        PARENT_REPO: <username>/<username>.github.io
        PARENT_BRANCH: <branch name>
        WORKFLOW_ID: <workflow id>
      run: |
        curl -fL --retry 3 -X POST -H "Accept: application/vnd.github.v3+json" -H "Authorization: token ${{ env.CI_TOKEN }}" -H "X-GitHub-Api-Version: 2022-11-28" https://api.github.com/repos/${{ env.PARENT_REPO }}/actions/workflows/${{ env.WORKFLOW_ID }}/dispatches -d '{"ref":"${{ env.PARENT_BRANCH }}"}' 
```
여기서 눈여겨볼 점은 `CI_TOKEN`의 사용이다. `env.CI_TOKEN`은 `secret.CI_TOKEN`에서 오는데, 이는 레포의 설정에서 설정할 수 있다. 또한 이 토큰 자체는 계정 설정의 developer settings에서 생성할 수 있다.<br>
또 `WORKFLOW_ID`는 deploy 부분을 먼저 만든 이유 중 하나이다. github api를 이용하면 workflow 목록을 볼 수 있는데, 그 목록에 나타나는 id가 바로 이 값이다.<br>

### 주의
따로 확인해보지는 않았으나, workflow를 한 번도 수동으로 실행한 적이 없으면 github api를 통한 REST요청으로 workflow 실행이 불가한 현상이 있을 수 있다고 한다.

## 테스트
Github에서 직접 파일을 올리든, Obsidian Git을 사용하든 해서 `BlogPosts` 레포를 업데이트 해보자. 자동으로 action에 작업들이 실행될 것이다. 참고로, action이 리턴값 22로 끝나는 것은 curl의 문제이다. 대개는 `WORKFLOW_ID`를 잘못 입력했을 것이다.

## 추가: Android 기기에서 편집하기
Obsidian의 Git 플러그인은 무려 안드로이드를 지원한다. 하지만 git은 안드로이드를 지원하지 않기 때문에, Termux에서 `termux-setup-storage` 명령어를 사용해 공용 폴더에 git clone을 한 뒤 Obsidian에서 여는 것을 추천한다. 추가로, 플러그인에 패스워드를 입력할 땐 `CI_TOKEN`처럼 생성한 토큰을 사용해야 한다. 일반 패스워드는 작동하지 않는다.