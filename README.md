#!/bin/bash
# sd-rg: A wrapper around ripgrep to mimic sd (find-and-replace)
# Usage: sd-rg [OPTIONS] PATTERN REPLACEMENT [PATH...]

set -euo pipefail

show_help() {
    cat <<'EOF'
sd-rg - Find and replace using ripgrep (sd alternative)

USAGE:
    sd-rg [OPTIONS] PATTERN REPLACEMENT [PATH...]
    sd-rg [OPTIONS] PATTERN -- REPLACEMENT [PATH...]
    command | sd-rg PATTERN REPLACEMENT

OPTIONS:
    -p, --preview       Preview changes without modifying files
    -s, --string-mode   Treat pattern as literal string, not regex
    -f, --flags FLAGS   Regex flags (e.g., 'i' for case-insensitive)
    -F, --fixed-strings Alias for --string-mode
    -h, --help          Show this help message

QUICK GUIDE:

  1. String-literal mode (-F or -s). Disable regex:

      echo 'lots((([]))) of special chars' | sd-rg -F '((([])))' ''
      lots of special chars

  2. Basic regex use - trim trailing whitespace:

      echo 'lorem ipsum 23   ' | sd-rg '\s+$' ''
      lorem ipsum 23

  3. Capture groups

     Indexed capture groups:

      echo 'cargo +nightly watch' | sd-rg '(\w+)\s+\+(\w+)\s+(\w+)' 'cmd: $1, channel: $2, subcmd: $3'
      cmd: cargo, channel: nightly, subcmd: watch

     Named capture groups:

      echo "123.45" | sd-rg '(?P<dollars>\d+)\.(?P<cents>\d+)' '$dollars dollars and $cents cents'
      123 dollars and 45 cents

     Resolve ambiguities with ${var} instead of $var:

      echo '123.45' | sd-rg '(?P<dollars>\d+)\.(?P<cents>\d+)' '${dollars}_dollars and ${cents}_cents'
      123_dollars and 45_cents

  4. Find & replace in a file (modified in-place):

      sd-rg 'window.fetch' 'fetch' http.js

     To preview changes:

      sd-rg -p 'window.fetch' 'fetch' http.js

  5. Find & replace across project (using fd):

      fd --type file --exec sd-rg 'from "react"' 'from "preact"'

EDGE CASES:

  Arguments starting with - need -- separator:

      echo "./hello foo" | sd-rg "foo" -- "-w"
      ./hello -w

      echo "./hello --foo" | sd-rg -- "--foo" "-w"
      ./hello -w

COMPARISON TO SED:

  Replace newlines with commas:
      sd-rg '\n' ','
      sed ':a;N;$!ba;s/\n/,/g'

  Extract path from string:
      echo "sample with /path/" | sd-rg '.*(/.*/)' '$1'
      echo "sample with /path/" | sed -E 's/.*(\/.*\/)/\1/g'

NOTES:
    Replacement supports capture groups: $1, $2, etc.
    Named groups: $name or ${name}
EOF
}

preview=false
string_mode=false
flags=""
end_of_flags=false
positional=()

while [[ $# -gt 0 ]]; do
    if $end_of_flags; then
        positional+=("$1")
        shift
        continue
    fi

    case "$1" in
        -p|--preview)
            preview=true
            shift
            ;;
        -s|--string-mode|-F|--fixed-strings)
            string_mode=true
            shift
            ;;
        -f|--flags)
            flags="$2"
            shift 2
            ;;
        -h|--help)
            show_help
            exit 0
            ;;
        --)
            end_of_flags=true
            shift
            ;;
        -*)
            echo "Error: Unknown option: $1" >&2
            echo "Use -- to separate flags from arguments starting with -" >&2
            exit 1
            ;;
        *)
            positional+=("$1")
            shift
            ;;
    esac
done

if [[ ${#positional[@]} -lt 2 ]]; then
    echo "Error: PATTERN and REPLACEMENT required" >&2
    show_help >&2
    exit 1
fi

pattern="${positional[0]}"
replacement="${positional[1]}"
paths=("${positional[@]:2}")

# Build rg arguments
rg_args=()

$string_mode && rg_args+=(--fixed-strings)
[[ "$flags" == *i* ]] && rg_args+=(--ignore-case)

# Check if reading from stdin
if [[ ${#paths[@]} -eq 0 && ! -t 0 ]]; then
    rg "${rg_args[@]}" --passthru --replace="$replacement" -- "$pattern"
    exit 0
fi

# File mode: default to current directory
[[ ${#paths[@]} -eq 0 ]] && paths=(.)

rg_args+=(--no-heading --with-filename)

if $preview; then
    echo "Preview of changes (showing replacements):"
    echo "==========================================="
    rg "${rg_args[@]}" --color=always --replace="$replacement" -- "$pattern" "${paths[@]}" | head -100
    echo ""
    echo "Files that would be modified:"
    rg "${rg_args[@]}" --files-with-matches -- "$pattern" "${paths[@]}"
else
    mapfile -t files < <(rg "${rg_args[@]}" --files-with-matches -- "$pattern" "${paths[@]}" 2>/dev/null || true)

    if [[ ${#files[@]} -eq 0 ]]; then
        echo "No matches found"
        exit 0
    fi

    for file in "${files[@]}"; do
        rg "${rg_args[@]}" --passthru --no-line-number --no-filename \
           --replace="$replacement" -- "$pattern" "$file" > "${file}.tmp"
        mv "${file}.tmp" "$file"
        echo "Modified: $file"
    done
    echo "${#files[@]} file(s) modified"
fi
