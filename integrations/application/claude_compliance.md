# Claude Compliance

:::{csv-table}
:align: left
:width: 45%
:widths: 15, 25
**Integration Details**
    Ingester, Claude Compliance Hosted Ingester
:::

## Claude Compliance Configuration

Anthropic provides a **Compliance API** for Claude Enterprise and Claude Console customers. See [Compliance API](https://platform.claude.com/docs/en/manage-claude/compliance-api). The Claude Compliance ingester polls this API and ingests each JSON record as a Gravwell entry.

To configure the ingester you will need the following from Anthropic:

* **API key**: Either a **Compliance Access Key** (created in claude.ai, reaches every endpoint) or an **Admin API key** (reaches the Activity Feed only). The ingester reads the key from a file.
* **Scope label**: A non-secret label you choose to identify the key's scope boundary (for example, the organization the key belongs to). It is used for rate limiting and saved state.

What each key can read determines which datasets you can ingest:

* **Activity Feed** (`activities` dataset): admin/system and resource events (workspace changes, API key creation, file operations). Available with either key type.
* **Everything else** (directory, organization settings, chat/file/project content, and session transcripts): Claude Enterprise only, and requires a Compliance Access Key.

```{attention}
The chat, project, and session datasets contain conversation content. Ingest only the datasets you need, and restrict access to the resulting well with [CBAC](/cbac/cbac).
```

Use a dedicated key for Gravwell. The ingester is read-only: it does not download binary files, delete vendor content, or acknowledge vendor records.

A separate, narrower **manual export** also exists: Organization Owners and Primary Owners can export audit logs via **claude.ai > Organization settings > Data and Privacy**, capped at a 180-day lookback and delivered as a downloadable file via an emailed link. This excludes chat and project content and title text and cannot be automated, so it is not a feed for continuous ingestion.

There is no webhook or push mechanism for the Compliance API; everything is pull-based, which is why the ingester polls.

## Gravwell Configuration

### Gravwell Storage Well Configuration

Set up the well configuration in your Gravwell indexers.

#### Sample Well Config
Create or edit: `/opt/gravwell/etc/gravwell.conf.d/claude_compliance-well.conf`
```ini
[Storage-Well "claude_compliance"]
    Location=/opt/gravwell/storage/claude_compliance
    Tags=claude-compliance*
```

### Gravwell Ingester Configuration: Claude Compliance Hosted Ingester

The ingester runs as a plugin inside the [Gravwell Hosted Runner](hosted_runner_configuration). If the Hosted Runner is not installed, install it first. The `[Global]` and `[State]` blocks common to all Hosted Runner plugins are described in [Hosted Runner Configuration](hosted_runner_configuration).

```{note}
The Claude Compliance plugin is still in development (see [gravwell/gravwell#2783](https://github.com/gravwell/gravwell/pull/2783)) and is not yet available in a released Hosted Runner. The options below come from that development version and may change before release.
```

Save the API key on a single line in its own file, readable only by the Hosted Runner user (mode `0600` is recommended). The file must be a regular file of 16 KB or less, and the key must not contain line breaks.

Edit: `/opt/gravwell/etc/hosted_runner.conf`

#### Sample Claude Compliance Config: Activity Feed

```ini
[ClaudeCompliance "activities"]
    Ingester-UUID="99f00000-0000-0000-0000-000000000000"
    Dataset="activities"
    Credential-File="/opt/gravwell/secrets/claude-compliance-key"
    Scope-Identity="example-org"
```

#### Sample Claude Compliance Config: Parameterized Dataset
Datasets that address a specific resource need a `Parameter` for each placeholder in their API path, written as `name:id`. For example, the users of a single organization:

```ini
[ClaudeCompliance "org-users"]
    Ingester-UUID="99f10000-0000-0000-0000-000000000000"
    Dataset="organization-users"
    Parameter="organization_id:your-organization-id"
    Credential-File="/opt/gravwell/secrets/claude-compliance-key"
    Scope-Identity="example-org"
```

#### Datasets

Set `Dataset` to one of the values below. By default the plugin stores each family under its own tag, with the prefix `claude-compliance-` (for example, `claude-compliance-activities`). Set `Tag-Name` to override it.

| Family | Datasets that need no `Parameter` | Datasets that need a `Parameter` (placeholder) |
| --- | --- | --- |
| Activity Feed | `activities` | |
| Directory | `organizations`, `groups` | `organization-users`, `organization-roles`, `organization-settings` (`organization_id`); `organization-role` (`organization_id`, `role_id`); `role-permissions` (`organization_id`, `role_id`); `group` (`group_id`); `group-members` (`group_id`) |
| Conversations | `chats` | `chat-messages` (`chat_id`); `file-metadata` (`file_id`); `generated-file-metadata` (`file_id`) |
| Projects | `projects` | `project` (`project_id`); `project-attachments` (`project_id`); `project-collaborators` (`project_id`); `project-document` (`document_id`); `project-document-metadata` (`document_id`) |
| Artifacts | | `artifact-metadata` (`artifact_version_id`) |
| Sessions | `local-sessions`, `remote-sessions` | `local-session` (`session_id`); `local-session-messages` (`session_id`); `remote-session-messages` (`session_id`) |

With `Follow-Children` at its default of `enabled`, the plugin discovers the child records of `organizations`, `groups`, `chats`, `projects` and the session roots on its own, so you usually only need stanzas for the roots. Only configure datasets that your key is authorized to read.

```{note}
Give every `[ClaudeCompliance]` stanza its own `Ingester-UUID`, and keep it unchanged once data has been collected. Each stanza sets one `Dataset`, so use one stanza for each dataset you want.
```

```{note}
Remember to restart the service to apply the new config:
`sudo systemctl restart gravwell_hosted_runner.service`
```

## Additional Resources
* [Compliance API](https://platform.claude.com/docs/en/manage-claude/compliance-api)
* [Admin API](https://platform.claude.com/docs/en/manage-claude/admin-api)
* [Usage & Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)
* [Access audit logs (manual export)](https://support.claude.com/en/articles/9970975-access-audit-logs)
