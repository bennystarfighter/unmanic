# Unmanic Docker Image


### Building the image
Simply run this command from the root of the project:
```bash
docker build -f ./docker/Dockerfile -t josh5/unmanic:staging -t ghcr.io/unmanic/unmanic:staging .
```

The Docker build installs Python dependencies directly from `pyproject.toml`
and `uv.lock`, so it does not require a prebuilt wheel, sdist, or
`requirements.txt`.
