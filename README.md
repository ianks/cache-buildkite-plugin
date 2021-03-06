# Cache Buildkite Plugin [![Version badge](https://img.shields.io/badge/cache-v2.3.5-blue?style=flat-square)](https://buildkite.com/plugins) [![Build status](https://badge.buildkite.com/eb76936a02fe8d522fe8cc986c034a6a8d83c7ec75e607f7bb.svg)](https://buildkite.com/gencer/buildkite-cache)


### Tarball, Rsync & S3 Cache Kit for Buildkite. Supports Linux, macOS and Windows*

&ast; Windows requires **Git for Windows 2.25 and later**.


A [Buildkite plugin](https://buildkite.com/docs/agent/v3/plugins) to restore and save
directories by cache keys. For example, use the checksum of a `.resolved` or `.lock` file
to restore/save built dependencies between independent builds, not just jobs.

With tarball or rsync, if source folder has changes, this will not fail your build, instead will surpress, notify and continue.

For S3, Instead of sync thousands of files, It just creates a tarball before S3 operation then copy this tarball to s3 at one time. This will reduce both time and cost on AWS billing.

Plus, In addition to tarball & rsync, we also do not re-create another tarball for same cache key if it's already exists.

# Backends

Please see `lib/backends/*.sh` for available backends. You can fork, add your backend then send a PR here.

Available backends and their requirements:

| **Backend** | **Linux (GNU)**                             | **macOS (BSD)**                             | **Windows**     |
| ----------- | ------------------------------------------- | ------------------------------------------- | --------------- |
| `tarball`   | tar<br />sha1sum                            | tar<br />shasum                             | Same as Linux   |
| `rsync`     | rsync<br />sha1sum                          | rsync <br />shasum                          | Same as Linux*  |
| `s3`        | aws-cli (`>= 1, ~> 2`)<br />tar<br/>sha1sum | aws-cli (`>= 1, ~> 2`)<br />tar<br />shasum | Same as Linux   |

### Windows Support

If you install **Git for Windows 2.25 and later**, you will benefit all features of Cache on Windows. Make sure you've added `bash.exe` into your `PATH`.

&ast; Rsync on Windows requires https://itefix.net/cwrsync. To be clear, except `rsync`, you can use `s3` and `tarball` on Windows without an additional app.

# S3

This plugin uses AWS S3 `cp` to cache the paths into a bucket as defined by environment
variables defined in your agent. Content of the paths will be packed with `tar` before upload `cp` and single tarball will be copied to s3.

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.5:
        backend: s3
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        s3:
          profile: "other-profile" # Optional. Defaults to `default`.
          bucket: "s3-bucket"
          compress: true # Create tar.gz instead of .tar (Compressed) Defaults to `false`.
          class: STANDARD # Optional. Defaults to empty which is usually STANDARD or based on policy.
          args: '--option 1' # Optional. Defaults to empty. Any optional argument that can be passed to aws s3 cp command.
        paths:
          - 'Pods/'
          - 'Rome/'
```

The paths are synced using Amazon S3 into your bucket using a structure of
`organization-slug/pipeline-slug/cache_key.tar`, as determined by the Buildkite environment
variables.

### Storage Class

You can pass `class` option for the following classes:,

- `STANDARD`
- `STANDARD_IA`
- `ONEZONE_IA`
- `INTELLIGENT_TIERING`

### Additional Arguments

You can pass `args` argument with required options. This arguments will be added to the end of `s3 cp` command. Therefore please do not add following options:

- `--storage-class`
- `--profile`

However, If you do not specify `profile` and `class` via YAML, then you can pass those arguments to the `args`.

# rsync

You can also use rsync to store your files using the `rsync` backend. Files will neither compressed nor packed.

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.5:
        backend: rsync
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        rsync:
          path: '/tmp/buildkite-cache'
        paths:
          - 'Pods/'
          - 'Rome/'
```

The paths are synced using `rsync_path/cache_key/path`. This is useful for maintaining a local
cache directory, even though this cache is not shared between servers, it can be reused by different
agents/builds.

# tarball

You can also use tarballs to store your files using the `tarball` backend. Files will not be compressed but surely packed into single archive.

**This is the Default and Recommended backend for cache**

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.5:
        backend: tarball
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        tarball:
          path: '/tmp/buildkite-cache'
          max: 7 # Optional. Removes tarballs older than 7 days.
          compress: true # Create tar.gz instead of .tar (Compressed) Defaults to `false`.
        paths:
          - 'Pods/'
          - 'Rome/'
```

The paths are synced using `tarball_path/cache_key.tar`. This is useful for maintaining a local
cache directory, even though this cache is not shared between servers, it can be reused by different
agents/builds.

### Cache Key Templates

The cache key is a string, which support a crude template system. Currently `checksum` is
the only command supported for now. It can be used as in the example above. In this case
the cache key will be determined by executing a _checksum_ (actually `sha1sum`) on the
`Gemfile.lock` file, prepended with `v1-cache-{{ runner.os }}-`.

### Hashing (checksum) against directory

Along with lock files, you can calculate directory that contains multiple files or recursive directories and files.

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.5:
        backend: tarball
        key: "v1-cache-{{ runner.os }}-{{ checksum './app/javascript' }}" # Calculate whole 'app/javascript' recursively
        tarball:
          path: '/tmp/buildkite-cache'
          max: 7 # Optional. Removes tarballs older than 7 days. 
          compress: true # Create tar.gz instead of .tar (Compressed) Defaults to `false`.
        paths:
          - 'Pods/'
          - 'Rome/'
```

For example, you can calculate total checksum of your javascript folder to skip build, If the source didn't changed.

Note: Before hashing files, we do "sort". This provides exact same sorted and hashed content against very same directory between builds.

### Skip Cache on PRs

You can skip caching on Pull Requests (Merge Requests) by simply adding `pr: false` to the cache plugin. For example;

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.5:
        backend: s3
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        pr: false # Default to `true` which is do cache on PRs.
        s3:
          profile: "other-profile" # Optional. Defaults to `default`.
          bucket: "s3-bucket"
          compress: true # Create tar.gz instead of .tar (Compressed) Defaults to `false`.
          class: STANDARD # Optional. Defaults to empty which is usually STANDARD or based on policy.
          args: '--option 1' # Optional. Defaults to empty. Any optional argument that can be passed to aws s3 cp command.
        paths:
          - 'Pods/'
          - 'Rome/'
```

Or you can set this by Environment:

```
#!/bin/bash

export BUILDKITE_PLUGIN_CACHE_PR=false
```

### Auto deletion old caches

To keep caches and delete them in _for example_ 7 days, use tarball backend and use `max`. On S3 side, please use S3 Policy for this routine. Each uploaded file to S3 will be deleted according to your file deletion policy.

**For S3**, Due to expiration policy, we just re-upload the same tarball to refresh expiration date. As long as you use the same cache, S3 will not delete it. Otherwise, It will be deleted from S3-side not used in a manner time.

### Globs on paths

You can use glob pattern in paths (to be cached) after `v2.1.x`

## Roadmap

+ Google Cloud Cache Support.
+ S3 Storage Class Support

Original work by [@danthorpe](https://github.com/danthorpe/cache-buildkite-plugin)

Copyright (C) 2020 Gencer W. Genç.

Licensed as **MIT**.
