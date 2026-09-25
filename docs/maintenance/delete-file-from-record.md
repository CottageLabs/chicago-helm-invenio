# Deleting a single file from a published record

InvenioRDM deliberately has no UI, admin panel, or REST API path for deleting
a single file from an already-published record without creating a new
version — the supported route is always: edit → new version → remove file
from the draft → publish.

For the rare case where that isn't acceptable (e.g. a takedown of an
accidentally uploaded file where a new version isn't wanted), use
`site/chicago_invenio/scripts/delete_file_from_record.py` in the
`chicago-invenio` repo, run inside a temporary pod on the cluster. The script
ships baked into the application image already — no need to copy anything
onto the pod.

**Before you run it, read the script's own docstring** (top of the file) —
it documents exactly what does and doesn't happen (no audit trail beyond the
log line, no DOI metadata update, bucket lock/unlock handling, and the
metadata-only flip if it's the record's last file).

---

## 1. Set up the terminal pod

The cluster has a dedicated `invenio-terminal` deployment for exactly this
kind of one-off task — same image, same env and secrets as `invenio-web`,
normally scaled to 0 replicas. `scripts/console.sh` scales it up, attaches a
shell, and offers to scale it back down when you're done.

From `chicago-helm-invenio`:

```bash
export AWS_PROFILE=chicago-invenio   # needed for every kubectl call below

cd scripts
./console.sh --namespace invenio
```

This scales `invenio-terminal` to 1, waits for it to be ready, and drops you
into a `bash` shell inside the pod.

---

## 2. Run the script

Inside the pod shell:

```bash
cd /opt/invenio/src
```

**Dry run first** — no options, so no need for the double `--` (see below):

```bash
invenio shell site/chicago_invenio/scripts/delete_file_from_record.py <record_id> "<file_key>"
```

This resolves the record and file and prints what *would* be deleted,
without changing anything. Check the output — especially whether it reports
this as the record's last remaining file (in which case the record will be
flipped to metadata-only when you actually delete).

**Then actually delete**, once the dry run looks right:

```bash
invenio shell site/chicago_invenio/scripts/delete_file_from_record.py \
    -- -- <record_id> "<file_key>" --yes --reason "<why this file is being removed>"
```

Notes:

- **Quote the file key** (`"<file_key>"`). It's the literal filename as
  stored on the record, which can contain spaces or characters the shell
  would otherwise split on.
- **The double `--` is required whenever you pass any option** (`--yes`,
  `--reason`) — `invenio shell` runs this via IPython, and both Click
  (invenio shell's own option parser) and IPython each consume one `--` as
  their own "stop parsing options" marker before the rest reaches this
  script's argparse. The plain dry run above has no options, so it doesn't
  need this.
- **Always pass `--reason`** for the real deletion — Invenio keeps no other
  record of why the file disappeared.

---

## 3. Clean up

Exit the pod shell (`exit` or Ctrl-D). `console.sh` will then prompt:

```
🚨 Do you want to scale down the pod? (y/N):
```

Answer `y` to scale `invenio-terminal` back to 0 — there's no reason to
leave it running between uses.
