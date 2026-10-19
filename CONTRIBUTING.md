# Contributing to Switch Trust SDK (Python)

Thank you for your interest in the Switch Trust SDK.

## Contribution policy

At this time, **we do not accept external code contributions**. Pull requests from outside the team will not be merged.

This is not a reflection on the quality of any contribution — it allows us to maintain tight control over the codebase, security posture, and release process.

## How you can help

- **Bug reports:** [open an issue](https://github.com/sandbox-quantum/switch-trust-sdk-py/issues/new)
  with a minimal reproducible example and the version of `switch-trust-sdk-py`
  you are using.
- **Feature requests:** open an issue describing the use case.
- **Questions:** open an issue.

## Security issues

Do not open a public issue for a vulnerability. Follow [SECURITY.md](SECURITY.md)
to report it privately.

## Building locally

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
pytest              # Run test suite
ruff check .        # Lint
ruff format .       # Format
mypy src/           # Type-check
```

## License

This SDK is licensed under the terms in [LICENSE](LICENSE): Apache 2.0, as
limited by a modified Commons Clause license condition.
