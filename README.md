# Open Inspection Registry

Open, versioned definitions for inspection findings used by the Open Inspection Observation Platform (OIO).

This repository is the shared vocabulary for computer-vision and Content Understanding inspections. It defines **what an inspection should observe** and how the result should be represented. It does not contain customer images, production credentials, or the private API implementation.

## What Is This?

An inspection system often returns free-form text such as:

> The material label is missing and the package is dirty.

That text is difficult to validate, compare, search, and use in automation. The registry turns the observation into stable fields:

```json
{
  "label_status": "missing",
  "cleanliness": "dirty"
}
```

The registry contains three related concepts:

- **Observation / finding field**: one thing to detect, such as `label_status` or `ppe_helmet`.
- **Observation Set**: a reusable group of fields for a domain, such as manufacturing quality or construction safety.
- **Skill**: the inspection intent and expected final finding, such as checking warehouse labels.

## Why Is It Needed?

- **Consistent results**: different clients use the same field IDs and allowed values.
- **Better automation**: APIs, MCP tools, dashboards, and downstream systems can consume structured data.
- **Clear review process**: new fields are proposed through a GitHub Pull Request instead of being silently added to a service.
- **Analyzer independence**: a field describes the inspection contract; Azure Content Understanding, a custom analyzer, or another vision provider can implement it.
- **Versioned behavior**: a result can record the registry revision used for an inspection.
- **Reusable skills**: an inspection skill can be shared across applications without copying prompt text and output definitions.

## Where Is It Used?

Typical scenarios include:

- Manufacturing quality: label status, surface damage, cleanliness, corrosion.
- Construction safety: helmet, gloves, harness, and other PPE observations.
- Warehouse operations: package labels, pallet condition, and material visibility.
- Retail: shelf fill level, price label status, and product count.
- Equipment inspection: nameplate presence, gauge reading, serial number, and leakage.

The first implementation targets image inspection. Document, audio, and video fields may be added later when the platform supports those modalities.

## Repository Layout

The target layout is:

```text
registry/
  observations/       # finding field definitions
  observation_sets/   # domain collections of fields
  skills/             # inspection intent and expected output
  examples/           # sample inputs and outputs
schemas/              # validation schemas
```

The initial repository contains the README only. The directories and validation workflow will be added with the first registry contribution.

## Finding Field Standard

Every new finding field must provide a stable `id`, a human-readable name, a precise description, an output type, allowed values where applicable, examples, and an expected output.

Recommended YAML format:

```yaml
id: label_status
name: Label Status
description: Check whether the material label is visible and readable.
type: classifier
values:
  - present
  - missing
  - damaged
  - unreadable
confidence_required: true
source_required: false
examples:
  positive:
    - images/label-status/present-001.jpg
    - images/label-status/present-002.jpg
    - images/label-status/present-003.jpg
  negative:
    - images/label-status/missing-001.jpg
    - images/label-status/damaged-001.jpg
    - images/label-status/unreadable-001.jpg
expected_output:
  label_status: missing
```

### Required fields

| Field | Requirement |
| --- | --- |
| `id` | Lowercase `snake_case`, stable, unique, and not tied to a vendor or model. |
| `name` | Short display name. |
| `description` | Observable, testable definition. Avoid vague terms such as “good” or “bad”. |
| `type` | One of `boolean`, `classifier`, `count`, or `extraction`. |
| `expected_output` | A JSON-shaped example matching the declared type. |
| `examples.positive` | At least 3 representative positive examples when the field is visual. |
| `examples.negative` | At least 3 representative negative or alternative examples when the field is visual. |

### Type guidance

- `boolean`: a yes/no observation, for example `ppe_helmet: true`.
- `classifier`: one value from a closed list, for example `label_status: missing`.
- `count`: a non-negative integer, for example `product_count: 12`.
- `extraction`: a value read from the image, for example `serial_number: ABC-123`.

### Quality rules

- Use one field for one observable concept.
- Prefer explicit allowed values over free-form text.
- Define what counts as `missing`, `unknown`, or `not_visible` when those states are possible.
- Do not encode a final business decision in the field ID. Use a Skill for interpretation such as “warehouse label check”.
- Avoid personally identifiable information and confidential customer images in this public repository.
- Keep positive and negative examples representative of real lighting, angles, occlusion, and image quality.
- Do not claim that a field is supported by a specific Azure analyzer until it has been tested.

## How to Submit a New Finding Field

1. Create a new file under `registry/observations/<field-id>.yaml`.
2. Follow the standard above and include at least three positive and three negative image references when applicable.
3. Add or update an Observation Set only when the field belongs to a reusable domain collection.
4. Add a Skill only when the field participates in a user-facing inspection intent.
5. Explain the use case and expected output in the Pull Request description.
6. Run the repository validation checks when they become available.
7. Submit the Pull Request for review.

### Pull Request checklist

- [ ] The field ID is unique and uses lowercase `snake_case`.
- [ ] The description is observable and testable.
- [ ] The output type and allowed values are complete.
- [ ] The expected output is valid JSON and matches the type.
- [ ] Positive and negative examples are included or the PR explains why examples are not applicable.
- [ ] No secrets, private URLs, personal data, or customer images are included.
- [ ] Backward compatibility and any renamed/replaced field are documented.

## API Specification

The registry is not itself a running API. The OIO service will expose these concepts through a REST API. This is the initial contract and may be expanded after implementation in the private service repository.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/v1/observations` | List available finding fields. |
| `GET` | `/api/v1/observations/{id}` | Get one field definition and revision. |
| `GET` | `/api/v1/observation-sets` | List reusable field collections. |
| `GET` | `/api/v1/skills` | List inspection skills. |
| `POST` | `/api/v1/inspections/analyze` | Analyze one image using an Observation Set and optional Skill. |

Example analysis request:

```json
{
  "image_url": "https://example.com/inspection-image.jpg",
  "observation_set": "manufacturing_quality_basic",
  "skill": "warehouse_label_check"
}
```

Example response shape:

```json
{
  "inspection_result": {
    "finding": "Material label is missing"
  },
  "observations": {
    "label_status": "missing",
    "cleanliness": "dirty"
  },
  "provenance": {
    "registry_revision": "main@<commit>",
    "analyzer_id": "prebuilt-imageSearch",
    "api_version": "2025-11-01"
  },
  "telemetry": {
    "request_id": "uuid",
    "trace_id": "uuid",
    "processing_time_ms": 1200,
    "usage": {}
  },
  "warnings": []
}
```

The API must preserve structured observations separately from the final natural-language finding. It must also preserve analyzer provenance, registry revision, warnings, and available usage data.

## MCP Tools

The OIO service will expose the same domain operation through MCP. REST and MCP must use the same application service so they cannot drift into different result formats.

Initial tool:

### `analyze_inspection`

Input:

```json
{
  "image_url": "https://example.com/inspection-image.jpg",
  "observation_set": "manufacturing_quality_basic",
  "skill": "warehouse_label_check"
}
```

Output: the same object as the REST analysis response, including `inspection_result`, `observations`, `provenance`, `telemetry`, and `warnings`.

The MCP implementation must not expose arbitrary Azure operations, unrestricted file access, secrets, or raw internal exceptions. Protocol version and transport details are implementation concerns of the private service and should be updated here when verified against the supported MCP SDK.

## Relationship to the Private Service Repository

- **This public repository**: schemas, registry records, examples, contribution rules, and reviewed domain vocabulary.
- **Private service repository**: Azure Content Understanding adapter, REST API, MCP server, Admin UI, authentication, telemetry, deployment, and production configuration.
- **Synchronization**: the private service can consume a tagged registry revision. Once the API and MCP implementation are stable, their verified contract and examples should be synchronized back into this README.

This separation lets users contribute a new standard finding field without needing access to the private Azure service implementation.

## Status

The repository is at the initial documentation stage. The next contribution should add the validation schema, the first sample Observation, Observation Set, Skill, and GitHub Pull Request templates.

## License

To be decided before accepting external contributions.
