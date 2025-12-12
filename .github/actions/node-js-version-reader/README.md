# node-js-version

Composite action that reads `package.json` and emits its `version` field.

## Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `package-json-path` | Location of the manifest to parse. | No | `package.json` |

## Outputs
| Name | Description |
| --- | --- |
| `source_last_tag_version` | Raw version string extracted from the manifest. |

Use it before guards or tagging logic to keep workflows decoupled from the file-reading implementation.
