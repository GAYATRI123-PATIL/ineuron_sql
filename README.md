# ineuron_sql
assignment
name: release

on:
  push:
    tags:
      - v*

jobs:

  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        go_version:
          - 1.19
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - uses: actions/setup-go@v3
        with:
          go-version: ${{ matrix.go_version }}

      # Cache go build cache, used to speedup go test
      - name: Setup Golang caches
        uses: actions/cache@v3
        with:
          path: |
            /go/pkg/.cache/go-build
            /go/pkg/mod
          key: ${{ runner.os }}-golang-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-golang-
      - name: GoReleaser
        uses: goreleaser/goreleaser-action@v3
        with:
          version: latest
          args: release --rm-dist
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  docker-release:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        target:
          - Dockerfile: build/Dockerfile
    steps:
      - uses: actions/checkout@v3

      - name: Prepare
        id: prepare
        run: |
          TAG=${GITHUB_REF#refs/tags/}
          DATE=$(date +'%Y-%m-%d_%H-%M-%S')
          echo ::set-output name=full_tag_name::${TAG}
          echo ::set-output name=full_date_tag::${DATE}
          echo ::set-output name=latest_tag::latest
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v1

      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@master

      - name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_SECRET }}

      - name: Build and publish ${{ matrix.target.Dockerfile }}
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          builder: ${{ steps.buildx.outputs.name }}
          file: ${{ matrix.target.Dockerfile }}
          platforms: linux/amd64,linux/arm64,linux/arm
          cache-from: type=gha,scope=${{ github.workflow }}
          cache-to: type=gha,mode=max,scope=${{ github.workflow }}
          tags: |
            zc2638/ylgy:${{ steps.prepare.outputs.full_tag_name }}
            zc2638/ylgy:${{ steps.prepare.outputs.full_date_tag }}
            zc2638/ylgy:${{ steps.prepare.outputs.latest_tag }}
