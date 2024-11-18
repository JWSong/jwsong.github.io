---
layout: post
title: "Optimizing Rust CI Pipeline with GitHub Actions: A Deep Dive into Caching Strategies"
description: "How I optimized our Rust CI pipeline with GitHub Actions, focusing on caching strategies and Docker optimizations."
keywords: "rust, github-actions, docker, ci, cache, optimization, devops, cargo"
author: "Jungwoo Song"
show_in_post_list: true
include_in_header: false
include_in_footer: false
robots: index, follow
---

As our Rust project grew, we faced increasing build times in our CI pipeline. This post shares my journey of optimizing the CI/CD process using GitHub Actions, focusing on caching strategies and Docker optimizations. I'll walk you through the problems I encountered and how I solved them.

## Initial Challenges

When I started, our CI pipeline had several issues:
- Long build times (14+ minutes)
- Repeated dependency downloads
- Inefficient Docker layer caching

Let's look at how I addressed each of these issues.

## Docker Layer Optimization

Our initial Dockerfile was simple but inefficient:

{% highlight dockerfile %}
FROM rust:1.75-slim
WORKDIR /app
COPY . .
RUN cargo build --release
{% endhighlight %}

This approach rebuilt everything on every change. I optimized it by separating dependencies:

{% highlight dockerfile %}
FROM rust:1.75-slim as builder

WORKDIR /app

# Copy only dependency files first
COPY Cargo.toml Cargo.lock ./

# Create dummy src for dependency compilation
RUN mkdir src && \
    echo "fn main() {}" > src/main.rs && \
    cargo build --release && \
    rm -rf src

# Copy actual source code and rebuild
COPY . .

# Build the application
RUN cargo build --release
{% endhighlight %}

This separation ensures that dependencies are cached in a separate layer, rebuilding only when `Cargo.toml` or `Cargo.lock` changes. The key insight here is that Docker layer caching works best when frequently changing files are copied last.

## GitHub Actions Cache Strategy

Initially, we didn't have a cache strategy. Here's how I improved it:

{% highlight yaml %}
name: Rust CI

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Cache cargo dependencies
        uses: actions/cache/restore@v4
        id: cache-target-restore
        with:
          path: |
            ~/.cargo
            ./target
          key: {% raw %}${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}{% endraw %}
          restore-keys: |
            {% raw %}${{ runner.os }}-cargo-{% endraw %}
{% endhighlight %}

The key improvements here are:
- Separate restore and save actions
- Use Cargo.lock hash for cache key

## Docker Build Optimization with Cache Mounts

The game-changer in our optimization journey was implementing BuildKit's cache mounts. This feature allows us to maintain a persistent cache across builds without increasing the final image size.

{% highlight dockerfile %}
FROM rust:1.75-slim as builder

WORKDIR /app

# Copy dependency files first
COPY Cargo.toml Cargo.lock ./

# Create dummy src for dependency compilation
RUN mkdir src && \
    echo "fn main() {}" > src/main.rs && \
    cargo build --release && \
    rm -rf src

# Build dependencies without source code
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release

# Copy actual source code
COPY . .

# Build the application
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release
{% endhighlight %}

The cache mounts provide several benefits:
- Persistent caching of dependencies
- No impact on final image size
- Faster subsequent builds

To enable this in GitHub Actions, I needed to configure Buildx:

{% highlight yaml %}
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Build and Push web image
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: {% raw %}{tag name}{% endraw %}
          cache-from: type=registry,ref={% raw %}{cache name}{% endraw %}
          cache-to: type=registry,ref={% raw %}{cache name}{% endraw %}
{% endhighlight %}

## Performance Impact

Our optimizations led to significant improvements:

| Step | Without Cache | With Cache |
|------|--------------|------------------|
| Initial Build | 14m 43s | 6m 23s |
| Dependency Change Only | 14m 43s | 6m 39s |
| Source Code Change Only | 14m 43s | 6m 21s |

The most dramatic improvement was in subsequent builds, where I saw a 55% reduction in build time.

## Handling Branch-Specific Caching

One challenge I faced was cache invalidation on new branches. In our project, we use a trunk-based development strategy where feature branches are created from the main branch. The issue was that each feature branch would create its own isolated cache, leading to redundant caching and longer build times across branches.
This isolation occurs due to GitHub Actions' cache access restrictions, which create separate caches for each branch for security purposes ([GitHub Actions Cache Documentation](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache)). While this isolation can be beneficial for security, it wasn't optimal for our trunk-based development workflow.
I solved this by leveraging the fact that caches created in the default branch can be accessed by all branches. I created a separate GitHub Actions workflow specifically for cache management that runs only on the main branch:


{% highlight yaml %}
name: update-cache

on:
  push:
    branches:
      - main

jobs:
  update-cache:
    runs-on: 
      group: ci-cd-runner
      labels: self-hosted-linux-x64

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install toolchain
        uses: dtolnay/rust-toolchain@stable

      - name: Check cache existence
        id: cache-check
        uses: actions/cache/restore@v4
        with:
          path: |
            ./target
            ~/.cargo
          key: {% raw %}${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}{% endraw %}
          lookup-only: true  # Only check if cache exists, don't restore

      - name: Early exit if cache hit
        if: steps.cache-check.outputs.cache-hit == 'true'
        run: exit 0

      - name: Build
        run: cargo build
 
      - name: Update cache
        id: cache-target-save
        uses: actions/cache/save@v4
        with: 
          path: |
            ~/.cargo
            ./target
          key: {% raw %}${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}{% endraw %}
{% endhighlight %}

This configuration ensures that:
- New feature branches first attempt to use their own cache
- If no branch-specific cache exists, it falls back to the main branch's cache
- The cache key includes the runner OS and Cargo.lock hash for proper dependency tracking

## Best Practices and Lessons Learned

1. **Layer Organization**
   - Keep frequently changing files in later layers
   - Separate dependency installation from application code
   - Use multi-stage builds for smaller final images

2. **Cache Strategy**
   - Create dedicated cache management workflow for main branch
   - Use Cargo.lock hash for cache keys
   - Implement cache existence check to avoid redundant builds
   - Leverage default branch cache sharing for feature branches

3. **BuildKit Optimization**
   - Use cache mounts for cargo registry and target directory
   - Configure proper cache locations
   - Implement platform-specific caching

## Conclusion

Through these optimizations, I achieved:
- 55% faster CI pipeline
- More efficient resource usage
- Better developer experience
- Improved team development productivity
- Reduced CI costs

The key to success was understanding how different caching mechanisms work together and implementing them in a way that complements each other. While the initial setup required some effort, the long-term benefits in terms of build time and resource usage have made it worthwhile.

## References
- [GitHub Actions Cache Documentation](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows)
- [Docker BuildKit Documentation](https://docs.docker.com/build/buildkit/)
- [Rust Cargo Documentation](https://doc.rust-lang.org/cargo/)
