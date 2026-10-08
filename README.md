# SignalForge

An Industrial Equipment Monitoring & Maintenance Platform built through guided engineering work. Current stage: environment setup and investigation of the MetroPT-3 compressor dataset. Application features and PostgreSQL have not been implemented.

## Python environment

Prerequisites: Git, uv, and Python 3.12. The initial setup uses the server's Python 3.12.3 and uv 0.11.7.

```bash
uv sync --locked
uv run python --version
```

`pyproject.toml` declares the project and dependencies; `uv.lock` records their resolved versions. `.python-version` selects Python 3.12. `uv sync` creates the local `.venv`, and `uv run` runs a command in it without requiring shell activation. There are no third-party dependencies yet.

## Source data

MetroPT-3 is provided by Davari, Veloso, Ribeiro, and Gama (2021), through the [UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/791/metropt%2B3%2Bdataset), DOI [10.24432/C5VW3R](https://doi.org/10.24432/C5VW3R), under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

Download the original archive on a fresh checkout:

```bash
mkdir -p data/raw/metropt-3
curl --fail --location --retry 3 --connect-timeout 20 --max-time 300 \
  --output data/raw/metropt-3/metropt-3.zip.part \
  'https://archive.ics.uci.edu/static/public/791/metropt%2B3%2Bdataset.zip'
mv data/raw/metropt-3/metropt-3.zip.part data/raw/metropt-3/metropt-3.zip
```

Verify the archive against the recorded download before extraction:

```bash
uv run python - <<'PY'
import hashlib
import json
from pathlib import Path
from zipfile import ZipFile

manifest = json.loads(Path('docs/data-manifest.json').read_text())
root = Path('data/raw/metropt-3')
archive = root / 'metropt-3.zip'
expected = next(item for item in manifest['files'] if item['path'] == str(archive))
with archive.open('rb') as stream:
    actual = hashlib.file_digest(stream, 'sha256').hexdigest()
if actual != expected['sha256']:
    raise RuntimeError('Download differs from recorded source; investigate before proceeding.')
with ZipFile(archive) as source:
    if source.testzip() is not None:
        raise RuntimeError('ZIP integrity check failed')
    for item in source.infolist():
        target = (root / item.filename).resolve()
        if not target.is_relative_to(root.resolve()):
            raise RuntimeError('Unsafe archive path')
        if target.exists():
            raise RuntimeError(f'Refusing to overwrite: {target}')
    source.extractall(root)
for item in manifest['files']:
    with Path(item['path']).open('rb') as stream:
        actual = hashlib.file_digest(stream, 'sha256').hexdigest()
    if actual != item['sha256']:
        raise RuntimeError(f"Checksum mismatch: {item['path']}")
print('Archive and extracted files match the recorded source.')
PY
```

Raw data is ignored by Git. Preserve these files unchanged; future transformations should write to separate paths. The recorded download includes a roughly 218 MB CSV and its source-description PDF. Checksums and source metadata are in [docs/data-manifest.json](docs/data-manifest.json).

Next: inspect timestamps and sensor definitions together, then compare measurements around a reported air-leak interval. SQL and PostgreSQL follow the initial data investigation.
