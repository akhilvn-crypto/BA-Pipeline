# BA-Pipeline

Claude Code marketplace and plugin for Business Analysis requirements work.

| Plugin | Location |
|---|---|
| `ba-requirements` | [`ba-requirements-plugin/`](ba-requirements-plugin) |

## Install

```text
/plugin marketplace add akhilvn-crypto/BA-Pipeline
/plugin install ba-requirements@ba-plugin-marketplace
```

Then run `/ba-requirements:run` to start the pipeline. See the [plugin README](ba-requirements-plugin/README.md) for the commands and phases.

## Test locally

```text
/plugin marketplace add ./BA-Pipeline
```

To test the plugin alone without a marketplace, use `claude --plugin-dir ./BA-Pipeline/ba-requirements-plugin`.
