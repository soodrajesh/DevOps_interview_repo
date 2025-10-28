#!/usr/bin/env bash
# shellcheck disable=SC2155
#
# <Short one-line description of what the script does>
#
# Usage: ./script.sh [-h] [-v] [-o OUTFILE] [INPUT...]
#   INPUT   files or stdin if omitted
#   -o      write result to OUTFILE (default: stdout)
#   -v      increase verbosity (repeatable)

set -euo pipefail
IFS=$'\n\t'

# ----------------------------------------------------------------------
# 1. LOGGING
# ----------------------------------------------------------------------
declare -i VERBOSITY=0
log() { (( VERBOSITY >= "$1" )) && printf '%s\n' "${@:2}"; }
die() { printf 'ERROR: %s\n' "$1" >&2; exit 1; }

# ----------------------------------------------------------------------
# 2. ARGUMENT PARSING
# ----------------------------------------------------------------------
OUTFILE=""
while (( $# )); do
    case "$1" in
        -h|--help)    echo "Usage: $0 [-h] [-v] [-o OUTFILE] [INPUT...]"; exit 0 ;;
        -v|--verbose) ((VERBOSITY++)) ; shift ;;
        -o|--output)  OUTFILE="$2"; shift 2 ;;
        --)           shift; break ;;
        -* )          die "Unknown option: $1" ;;
        *)            break ;;
    esac
done

INPUT_FILES=("$@")

# ----------------------------------------------------------------------
# 3. CORE LOGIC
# ----------------------------------------------------------------------
solve_problem() {
    mapfile -t lines
    # ---- BEGIN INTERVIEW LOGIC -----------------------------------------
    echo "${#lines[@]}"
    # ---- END INTERVIEW LOGIC -------------------------------------------
}

# ----------------------------------------------------------------------
# 4. I/O HELPERS
# ----------------------------------------------------------------------
if (( ${#INPUT_FILES[@]} == 0 )); then
    tmp=$(mktemp)
    cat > "$tmp"
    INPUT_FILES=("$tmp")
fi

result=$(cat "${INPUT_FILES[@]}" | solve_problem)
[[ -f "${INPUT_FILES[0]}" && "${INPUT_FILES[0]}" == *tmp* ]] && rm -f "${INPUT_FILES[0]}"

# ----------------------------------------------------------------------
# 5. OUTPUT
# ----------------------------------------------------------------------
if [[ -n "$OUTFILE" ]]; then
    printf '%s\n' "$result" > "$OUTFILE"
    log 1 "Result written to $OUTFILE"
else
    printf '%s\n' "$result"
fi

exit 0
