FROM public.ecr.aws/d3j8x8q7/swe-bench-202605:kh72qraccnjwdet6ynagsccr4x82y65c-v1.1
WORKDIR /app
RUN <<'DEEPSWE_TIME_TRAVEL'
set -eux
git checkout -B benchmark-base 88e3d965c0b1628642a30a841745b410d6835052
git remote | while read -r remote; do git remote remove "$remote"; done
git for-each-ref --format='%(refname)' refs/remotes/ | while read -r ref; do [ -z "$ref" ] || git update-ref -d "$ref"; done
git for-each-ref --format='%(refname:short)' refs/heads/ | while read -r name; do [ "$name" = "benchmark-base" ] || git branch -D "$name"; done
git tag | while read -r tag; do git merge-base --is-ancestor "$tag" HEAD || git tag -d "$tag"; done
git reflog expire --expire=now --all
git gc --prune=now
test "$(git rev-parse HEAD)" = "88e3d965c0b1628642a30a841745b410d6835052"
test -z "$(git status --porcelain --untracked-files=no)"
DEEPSWE_TIME_TRAVEL
