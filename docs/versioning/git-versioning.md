# Using tags for Versioning in Git

## Steps to release a version following [SemVer](https://semver.org/)

Assuming a forking workflow:

1. Pull latest `main` branch from `upstream` repo (and update our fork while we are at it)

```
git pull upstream main
git push origin main
```

2. Create branch to prepare new release

```
git switch -c prepare-release
```

3. Update **p**yproject.toml, **C**HANGELOG, **R**EADME.

- pyproject.toml

```
[tool.poetry]
name = "project-name"
version = "MAJOR.MINOR.PATCH"
```

- CHANGELOG.md

```
## [MAJOR.MINOR.PATCH] - YYYY-MM-DD

### Added

- Items

[MAJOR.MINOR.PATCH]:
  https://github.com/organization/project/releases/tag/MAJOR.MINOR.PATCH
```

- README.md

```
Install the application using `pip`:

pip install git+https://github.com/organization/project@MAJOR.MINOR.PATCH

```

4. Commit using this sample commit message

```
Prepare codebase for MAJOR.MINOR.PATCH release

Prepares the codebase for the MAJOR.MINOR.PATCH release.
```

5. Create PR with a required reviewer

```
gh pr create --fill -a @me -r reviewer'sgithubaccount
```

6. After PR is approved, update `main` branch (and update our fork while we are at it)

```
git pull upstream main
git push origin main
```

7. Create the current version's tag

```
git tag -am MAJOR.MINOR.PATCH "MAJOR.MINOR.PATCH"
```

8. Push tags upstream

```
git push upstream --tags
```
