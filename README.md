#!/bin/bash

INPUT_FILE="repos.yml"
OUTPUT_FILE="updated_repos.yml"
TEMP_DIR="./repos_temp"
USER_X="userX"
GIT_BASE_SSH="git@gitstash.com"

mkdir -p "$TEMP_DIR"
cp "$INPUT_FILE" "$OUTPUT_FILE"

# Check for yq
if ! command -v yq &> /dev/null; then
    echo "❌ 'yq' not found. Install from https://github.com/mikefarah/yq"
    exit 1
fi

keys=$(yq e 'keys | .[]' "$INPUT_FILE")

for key in $keys; do
    repo_name=$(echo "$key" | sed -E 's/_version$//' | tr '_' '-')
    repo_dir="$TEMP_DIR/$repo_name"

    echo "🔍 Processing $repo_name"

    if [ ! -d "$repo_dir" ]; then
        git clone --quiet "$GIT_BASE_SSH:$repo_name.git" "$repo_dir" || {
            echo "⚠️ Failed to clone $repo_name"
            continue
        }
    fi

    cd "$repo_dir" || continue
    git fetch --all --tags --quiet

    # Get latest semver tag (must match release format X.Y.Z)
    latest_tag=$(git tag --sort=-creatordate | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | tail -n 1)

    if [ -z "$latest_tag" ]; then
        echo "⚠️ No release tags found — skipping"
        cd - > /dev/null
        continue
    fi

    echo "🕑 Latest release tag: $latest_tag"

    # Check if repo was updated since tag
    committers=$(git log "$latest_tag"..origin/master --pretty="%an" | sort -u)

    updated_by_other=false
    while read -r committer; do
        if [[ "$committer" != "$USER_X" && -n "$committer" ]]; then
            updated_by_other=true
            break
        fi
    done <<< "$committers"

    if $updated_by_other; then
        echo "✅ Updated by someone other than $USER_X → snapshot needed"
        IFS='.' read -r major minor patch <<< "$latest_tag"
        minor=$((minor + 1))
        new_version="${major}.${minor}.0-SNAPSHOT"
    else
        echo "🛑 No updates by others → using latest release tag"
        new_version="$latest_tag"
    fi

    cd - > /dev/null

    # Write result, preserving order and formatting
    yq e ".\"$key\" = \"$new_version\"" -i "$OUTPUT_FILE"
done

echo "✅ Finished. Output written to $OUTPUT_FILE"
