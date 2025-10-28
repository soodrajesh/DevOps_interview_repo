#!/usr/bin/env bash
#===============================================================================
# Script Name: <name_here>.sh
# Description: <brief purpose>
# Author: Raj
#===============================================================================

set -euo pipefail
IFS=$'\\n\\t'

LOG_FILE="/tmp/script.log"
DEBUG=false

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

usage() {
    echo "Usage: $0 [-i input] [-o output] [-d]"
    exit 1
}

cleanup() {
    log "Cleaning up..."
}
trap cleanup EXIT

while getopts ":i:o:d" opt; do
    case $opt in
        i) INPUT="$OPTARG" ;;
        o) OUTPUT="$OPTARG" ;;
        d) DEBUG=true ;;
        *) usage ;;
    esac
done

log "Starting script..."

if [[ "${DEBUG}" == true ]]; then
    set -x
fi

# ADD YOUR LOGIC HERE
if [[ -n "${INPUT:-}" && -f "$INPUT" ]]; then
    COUNT=$(wc -l < "$INPUT")
    log "File $INPUT has $COUNT lines."
else
    log "No valid input file provided."
fi

log "Script completed successfully."
