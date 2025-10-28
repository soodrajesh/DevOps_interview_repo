from pathlib import Path

template_content = """
# Universal Script Templates (Python)
Author: Raj

This document provides **reusable, production-grade templates** for both Python scripting — ideal for DevOps, AWS automation, or interviews.

---

## Python Script Template

```python
#!/usr/bin/env python3
\"\"\"
Script Name: <name_here>.py
Description: <1-line what this does>
Author: Raj
\"\"\"

import os
import sys
import argparse
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

def parse_args():
    parser = argparse.ArgumentParser(description="Describe what the script does")
    parser.add_argument("-i", "--input", help="Input file or resource", required=False)
    parser.add_argument("-o", "--output", help="Output path", required=False)
    parser.add_argument("-v", "--verbose", action="store_true", help="Enable debug logging")
    return parser.parse_args()

def setup_env():
    logging.debug("Setting up environment...")

def main():
    args = parse_args()
    if args.verbose:
        logging.getLogger().setLevel(logging.DEBUG)
    setup_env()

    try:
        logging.info("Starting script logic...")
        # ADD YOUR LOGIC HERE
        if args.input:
            with open(args.input) as f:
                count = sum(1 for _ in f)
            logging.info(f"File {args.input} has {count} lines.")
        logging.info("Completed successfully.")
    except Exception as e:
        logging.exception(f"Script failed: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
