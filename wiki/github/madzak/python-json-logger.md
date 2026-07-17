# madzak/python-json-logger

> A formatter that makes Python's stdlib `logging` emit one JSON object per log line — the original, now retired in favor of a maintained fork.

[GitHub repo](https://github.com/madzak/python-json-logger) ·
[PyPI: python-json-logger](https://pypi.org/project/python-json-logger/) ·
[License: BSD-2-Clause](https://github.com/madzak/python-json-logger/blob/master/LICENSE)

## Overview

`python-json-logger` is a `logging.Formatter` subclass that serializes each log record to a single-line JSON object instead of a plain text string. The premise is narrow and durable: structured logs are readable by machines, so you stop writing regex parsers for freeform syslog-style output and let your log shipper (Fluentd, Vector, Logstash, CloudWatch, Loki) consume fields directly. It plugs into the standard library — no new logging API, just a formatter you attach to an existing handler.

This repository (`madzak/python-json-logger`) is the **original** project, created in 2011. As of this writing it is **archived and retired**: the maintainer has publicly recommended transitioning to the community fork `nhairs/python-json-logger`, which continues development under the same PyPI name[^1]. The archived repo still matters because an enormous amount of deployed code imports `from pythonjsonlogger import jsonlogger` against a version resolved from this lineage, and the PyPI package `python-json-logger` remains one of the most-downloaded logging libraries in the Python ecosystem[^2].

The defining tension is scope versus staleness. The library does exactly one small thing and does it well enough that it was rarely touched — which is also why it sat unmaintained for long stretches, accumulated an import-path change that broke code (0.1.0), and eventually needed a fork to modernize Python-version support and packaging.

## Getting Started

```bash
pip install python-json-logger
```

```python
import logging
from pythonjsonlogger import jsonlogger

logger = logging.getLogger()
handler = logging.StreamHandler()
handler.setFormatter(jsonlogger.JsonFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Extra keys land at the JSON root, alongside the standard record fields.
logger.info("user login", extra={"user_id": 42, "ip": "10.0.0.1"})
# {"message": "user login", "user_id": 42, "ip": "10.0.0.1"}
```

Choose which record attributes appear by passing a format string; the field names are the same `%(...)s` LogRecord attributes the stdlib uses:

```python
formatter = jsonlogger.JsonFormatter("%(asctime)s %(name)s %(levelname)s %(message)s")
```

## Architecture / How It Works

The whole library is essentially one class, `JsonFormatter`, and the surprise is how much of Python's `logging` internals leak through it:

- **`parse()`** reads the format string and returns the list of field names to include. Override it to source required fields from something other than a `%(...)s` string (the README shows splitting on `;`).
- **`add_fields(log_record, record, message_dict)`** is the extension seam. It is called for every event and is where you inject or normalize fields — a stable timestamp, an uppercased level, a `severity` key for GCP, a service name. Subclass, call `super()`, then mutate `log_record` in place.
- **Message merging.** Three sources are flattened into one flat object: the standard LogRecord attributes named in the format string, any dict passed as the log message itself, and the `extra={}` kwarg. Later sources override earlier ones, which means an `extra` key named `message` or `levelname` will silently clobber the built-in — a footgun worth knowing.
- **Serialization** goes through `json.dumps`. Non-serializable values are handled by `json_default` (a translator callable) or a custom `json_encoder` class. There is no schema and no field-type coercion; whatever survives `json.dumps` is what ships.

Because it is a formatter and not a handler, it composes with the rest of `logging` unchanged: rotating file handlers, `dictConfig`/`fileConfig`, `QueueHandler`, third-party handlers all work by pointing their `formatter` at `pythonjsonlogger.jsonlogger.JsonFormatter`.

## Production Notes

- **Import path history.** Version 0.1.0 changed the import structure; code written against very old releases breaks. The canonical import for this lineage is `from pythonjsonlogger import jsonlogger`. The maintained `nhairs` fork later reorganized modules again (`pythonjsonlogger.json`), so pinning your version and matching the import is mandatory when you upgrade across the fork boundary.
- **Fork migration is the real decision.** New projects should install the `nhairs` fork, which shares the `python-json-logger` PyPI name and adds maintained Python 3.8+ support, orjson/msgspec encoder options, and a documented field API[^1]. Existing projects can often upgrade in place because the base import compatibility was preserved, but read the changelog — module reorganization is real.
- **Field-name collisions.** Since `extra` and message-dict keys merge at the root and override built-ins, a careless `extra={"name": ...}` overwrites the logger name. Namespacing custom keys (or using `rename_fields`) avoids silent data loss.
- **Timestamps.** There is no default timestamp field unless you request `%(asctime)s`, and `asctime` formatting follows stdlib rules (local time, comma-milliseconds) that log pipelines often dislike. Most production setups override `add_fields` to emit an ISO-8601 UTC `timestamp`, as the README itself demonstrates.
- **Performance.** Serialization cost is dominated by `json.dumps` on every record on the calling thread. High-throughput services should route logs through a `QueueHandler`/`QueueListener` so formatting happens off the hot path, and consider the fork's faster encoders.
- **Reserved-key clashes with shippers.** Cloud log agents expect specific keys (`severity`, `message`, `logging.googleapis.com/trace`). This library gives you the flattened object but does not map to any vendor schema — that mapping is your `add_fields` subclass to write.

## When to Use / When Not

**Use when:**
- You already use stdlib `logging` and want JSON output with a one-line formatter swap.
- You ship logs to a system that parses JSON (ELK, Loki, CloudWatch, Datadog, Fluent Bit).
- You want minimal dependencies and a tiny, auditable surface.

**Avoid when:**
- You are starting fresh and want active maintenance — install the `nhairs` fork instead, or adopt `structlog`.
- You need contextual/bound loggers, processor pipelines, or async-native structured logging — `structlog` is the better model.
- You want opinionated observability (trace correlation, sampling, OTel export) out of the box — this is only a formatter.

## Alternatives

- nhairs/python-json-logger — the maintained continuation under the same PyPI name; use this instead of the archived madzak repo for any new install.
- hynek/structlog — use when you want bound loggers, processor pipelines, and structured context, not just JSON serialization at the formatter layer.
- Delgan/loguru — use when you want a batteries-included logging library that replaces stdlib ergonomics and can serialize to JSON with `serialize=True`.
- jteppinette/python-json-logging or stdlib `logging` + a hand-rolled `Formatter` — use when your needs are trivial enough that a dependency is not worth it.
- open-telemetry/opentelemetry-python — use when logging is one leg of a traces/metrics/logs observability stack and you want OTLP export.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | ~2015 | Import structure changed to `pythonjsonlogger.jsonlogger`. |
| 2.0.0 | 2021-06 | Dropped Python 2; modernized packaging and defaults. |
| 2.0.7 | 2023-01 | Last widely-used release from this repo. |
| — | 2024-12 | Repository archived; maintainer directs users to the `nhairs` fork[^1]. |

Version dates are approximate; confirm against the PyPI release history before relying on a specific one.

## References

[^1]: Retirement notice and fork recommendation in the project README. https://github.com/madzak/python-json-logger
[^2]: PyPI project page and download statistics. https://pypi.org/project/python-json-logger/

## Tags

python, logging, json, structured-logging, stdlib-logging, formatter, observability, log-shipping, archived, library
