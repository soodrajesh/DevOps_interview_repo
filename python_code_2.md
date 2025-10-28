# Universal Python & Bash Script Templates

> **Interview-ready skeletons** – paste, fill the docstring, add flags, replace the example logic, and you’re done.  
> Works with any problem that reads from files / `stdin` and writes to `stdout` / a file.

---

## 1. Python Template

```python
#!/usr/bin/env python3
"""
<Short one-line description of what the script does>

<Optional longer description / usage example>
"""

from __future__ import annotations

import argparse
import logging
import sys
from pathlib import Path
from typing import List, Optional

# ----------------------------------------------------------------------
# 1. CONFIGURATION & LOGGING
# ----------------------------------------------------------------------
LOGGER = logging.getLogger(Path(__file__).stem)

def setup_logging(verbosity: int = 0) -> None:
    """Configure root logger: 0=WARNING, 1=INFO, 2=DEBUG."""
    level = (logging.WARNING, logging.INFO, logging.DEBUG)[min(verbosity, 2)]
    fmt = "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s"
    logging.basicConfig(level=level, format=fmt, stream=sys.stderr)
    LOGGER.debug("Logging initialised (level=%s)", logging.getLevelName(level))


# ----------------------------------------------------------------------
# 2. ARGUMENT PARSING
# ----------------------------------------------------------------------
def parse_args(argv: Optional[List[str]] = None) -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description=__doc__.splitlines()[0],
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
    )
    parser.add_argument(
        "input",
        nargs="*",
        help="Input file(s) or data (leave empty for stdin)",
    )
    parser.add_argument(
        "-o", "--output",
        type=Path,
        help="Write result to FILE (default: stdout)",
    )
    parser.add_argument(
        "-v", "--verbose",
        action="count",
        default=0,
        help="Increase verbosity (-v=INFO, -vv=DEBUG)",
    )
    # ---- add problem-specific flags here ----
    # parser.add_argument("--foo", type=int, default=42, help="…")
    return parser.parse_args(argv)


# ----------------------------------------------------------------------
# 3. CORE LOGIC (the only part you change per interview question)
# ----------------------------------------------------------------------
def solve_problem(inputs: List[str], **options) -> str:
    """
    Implement the interview solution here.

    Parameters
    ----------
    inputs : List[str]
        Lines / tokens from the input source.
    **options
        Any extra flags you added in parse_args().

    Returns
    -------
    str
        The final answer (will be printed / written to file).
    """
    # ---- BEGIN INTERVIEW LOGIC ------------------------------------------------
    # Example: count the number of lines
    result = str(len(inputs))
    # ---- END INTERVIEW LOGIC --------------------------------------------------
    return result


# ----------------------------------------------------------------------
# 4. I/O HELPERS
# ----------------------------------------------------------------------
def read_inputs(paths: List[Path]) -> List[str]:
    """Read from files or stdin, return stripped lines."""
    if not paths:
        return [line.rstrip("\n") for line in sys.stdin]

    lines: List[str] = []
    for p in paths:
        if not p.exists():
            LOGGER.error("File not found: %s", p)
            sys.exit(1)
        lines.extend(p.read_text().splitlines())
    return lines


def write_output(text: str, dest: Optional[Path]) -> None:
    if dest:
        dest.write_text(text + "\n")
        LOGGER.info("Result written to %s", dest)
    else:
        sys.stdout.write(text + "\n")


# ----------------------------------------------------------------------
# 5. MAIN ENTRYPOINT
# ----------------------------------------------------------------------
def main(argv: Optional[List[str]] = None) -> int:
    args = parse_args(argv)
    setup_logging(args.verbose)

    input_paths = [Path(p) for p in args.input]
    data = read_inputs(input_paths)

    extra = {k: v for k, v in vars(args).items() if k not in {"input", "output", "verbose"}}
    answer = solve_problem(data, **extra)

    write_output(answer, args.output)
    return 0


if __name__ == "__main__":
    sys.exit(main())
