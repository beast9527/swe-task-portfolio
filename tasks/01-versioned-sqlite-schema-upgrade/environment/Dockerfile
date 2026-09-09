FROM docker.io/swerebenchv2/senth-minecraft-mod-manager:82-2782e2e
WORKDIR /minecraft-mod-manager
RUN <<'DEEPSWE_TIME_TRAVEL'
set -eux
git checkout -B benchmark-base 2782e2ed818d8586be2bc01474acecc10cee4e74
git remote | while read -r remote; do git remote remove "$remote"; done
git for-each-ref --format='%(refname)' refs/remotes/ | while read -r ref; do [ -z "$ref" ] || git update-ref -d "$ref"; done
git for-each-ref --format='%(refname:short)' refs/heads/ | while read -r name; do [ "$name" = "benchmark-base" ] || git branch -D "$name"; done
git tag | while read -r tag; do git merge-base --is-ancestor "$tag" HEAD || git tag -d "$tag"; done
git reflog expire --expire=now --all
git gc --prune=now
test "$(git rev-parse HEAD)" = "2782e2ed818d8586be2bc01474acecc10cee4e74"
test -z "$(git status --porcelain --untracked-files=no)"
DEEPSWE_TIME_TRAVEL
