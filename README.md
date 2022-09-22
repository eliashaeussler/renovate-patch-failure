# Renovate failure due to missing `patch` binary

## Additional configuration in `config.js`

The following configuration must be added to the global `config.js`:

```js
module.exports = {
    allowPlugins: true,
};
```

:bulb: We use `allowPlugins` because we need some other Compoer plugins during
the `composer update` process triggered by Renovate. Otherwise, the resulting
`composer.lock` file results in unwanted changes.

## Expected behavior

The patches defined in [`composer.patches.json`](composer.patches.json) should
be applied when Renovate runs the `composer update` process. This is an expected
behavior of the underlying [`cweagans/composer-patches`](https://github.com/cweagans/composer-patches)
Composer plugin.

## Current behavior

Instead of correctly running the `composer update` process, this process fails
with the following error message:

```
Gathering patches from patch file.
Gathering patches for dependencies. This might take a minute.
  - Applying patches for ralouphie/getallheaders
    patches/getallheaders.patch (Test patch)
    Could not apply patch! Skipping. The error was: Cannot apply patch patches/getallheaders.patch
```

## Futher investigation

I was able to track this issue down to a missing `patch` library in the Docker container.

### Reproduction

```
$ docker run --rm -it --user=0:0 -v $(pwd):/app --workdir /app renovate/php:7.4.30 \
    bash -c "install-tool composer 2.4.2 && composer update"

installing v2 tool composer v2.4.2
linking tool composer v2.4.2
Composer version 2.4.2 2022-09-14 16:11:15
Installed v2 /usr/local/buildpack/tools/v2/composer.sh in 2 seconds
skip cleanup, not a docker build: 2bd48ed7c2f2
Loading composer repositories with package information
Info from https://repo.packagist.org: #StandWithUkraine
Updating dependencies
Lock file operations: 0 installs, 1 update, 0 removals
  - Upgrading ralouphie/getallheaders (3.0.0 => 3.0.3)
Writing lock file
Installing dependencies from lock file (including require-dev)
Package operations: 8 installs, 0 updates, 0 removals
  - Downloading cweagans/composer-patches (1.7.2)
  - Downloading localheinz/diff (1.1.1)
  - Downloading justinrainbow/json-schema (5.2.12)
  - Downloading ergebnis/json-printer (3.2.0)
  - Downloading ergebnis/json-schema-validator (2.0.0)
  - Downloading ergebnis/json-normalizer (2.1.0)
  - Downloading ergebnis/composer-normalize (2.28.3)
  - Downloading ralouphie/getallheaders (3.0.3)
  - Installing cweagans/composer-patches (1.7.2): Extracting archive
Gathering patches from patch file.
Gathering patches for dependencies. This might take a minute.
  - Installing localheinz/diff (1.1.1): Extracting archive
  - Installing justinrainbow/json-schema (5.2.12): Extracting archive
  - Installing ergebnis/json-printer (3.2.0): Extracting archive
  - Installing ergebnis/json-schema-validator (2.0.0): Extracting archive
  - Installing ergebnis/json-normalizer (2.1.0): Extracting archive
  - Installing ergebnis/composer-normalize (2.28.3): Extracting archive
  - Installing ralouphie/getallheaders (3.0.3): Extracting archive
  - Applying patches for ralouphie/getallheaders
    patches/getallheaders.patch (Test patch)
   Could not apply patch! Skipping. The error was: Cannot apply patch patches/getallheaders.patch

In Patches.php line 326:

  Cannot apply patch Test patch (patches/getallheaders.patch)!
```

### Possible solution

Now, in a second run, we install the `patch` binary:

```
$ docker run --rm -it --user=0:0 -v $(pwd):/app --workdir /app renovate/php:7.4.30 \
    bash -c "install-tool composer 2.4.2 && apt-get update -qq && apt-get install -yqq patch && composer update"

installing v2 tool composer v2.4.2
linking tool composer v2.4.2
Composer version 2.4.2 2022-09-14 16:11:15
Installed v2 /usr/local/buildpack/tools/v2/composer.sh in 2 seconds
skip cleanup, not a docker build: b5c07dbd3451
Unpacking patch (2.7.6-6) ...
Setting up patch (2.7.6-6) ...
Loading composer repositories with package information
Info from https://repo.packagist.org: #StandWithUkraine
Updating dependencies
Lock file operations: 0 installs, 1 update, 0 removals
  - Upgrading ralouphie/getallheaders (3.0.0 => 3.0.3)
Writing lock file
Installing dependencies from lock file (including require-dev)
Package operations: 8 installs, 0 updates, 0 removals
  - Downloading cweagans/composer-patches (1.7.2)
  - Downloading localheinz/diff (1.1.1)
  - Downloading justinrainbow/json-schema (5.2.12)
  - Downloading ergebnis/json-printer (3.2.0)
  - Downloading ergebnis/json-schema-validator (2.0.0)
  - Downloading ergebnis/json-normalizer (2.1.0)
  - Downloading ergebnis/composer-normalize (2.28.3)
  - Downloading ralouphie/getallheaders (3.0.3)
  - Installing cweagans/composer-patches (1.7.2): Extracting archive
Gathering patches from patch file.
Gathering patches for dependencies. This might take a minute.
  - Installing localheinz/diff (1.1.1): Extracting archive
  - Installing justinrainbow/json-schema (5.2.12): Extracting archive
  - Installing ergebnis/json-printer (3.2.0): Extracting archive
  - Installing ergebnis/json-schema-validator (2.0.0): Extracting archive
  - Installing ergebnis/json-normalizer (2.1.0): Extracting archive
  - Installing ergebnis/composer-normalize (2.28.3): Extracting archive
  - Installing ralouphie/getallheaders (3.0.3): Extracting archive
  - Applying patches for ralouphie/getallheaders
    patches/getallheaders.patch (Test patch)

Generating autoload files
4 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
No security vulnerability advisories found
```
