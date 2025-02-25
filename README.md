# versiontagging

En un projecte que utilitza Trunk-Based Development (TBD) amb un repositori GitHub, l'etiquetatge de versions i la generació d'imatges Docker han de seguir un esquema que permeti identificar clarament cada versió de manera única i reproductible.

✅ Utilitzar SemVer per etiquetar versions.

✅ Generar imatges Docker amb etiquetes corresponents (latest, vX.Y.Z).

✅ Automatitzar amb GitHub Actions per garantir un procés eficient i sense errors humans.

## Estratègia d'etiquetatge de versions

1. Versions semàntiques

   Utilitzar Semantic Versioning (SemVer) és una bona pràctica. Això significa etiquetar les versions amb el format: `v<major>.<minor>.<patch>`

   Exemple: `v1.2.3`

   * Major: Canvis que trenquen compatibilitat.
   * Minor: Funcionalitats noves, però compatibles.
   * Patch: Correccions d'errors sense trencar res.

2. Etiquetatge automàtic

   Es pot automatitzar l'etiquetatge usant GitHub Actions o scripts per generar la versió basada en commits i etiquetar-la correctament.

3. Etiquetes per a cada release

   Quan es fa un merge a main, es pot afegir una etiqueta amb la nova versió:

   ```bash
   git tag -a v1.2.3 -m "Release v1.2.3"
   git push origin v1.2.3
   ```

## 🐳 Generació d’imatge Docker basada en la versió

Per garantir que les imatges Docker estan ben versionades, cal seguir aquestes pràctiques:

* Build i etiquetatge de la imatge Docker (Semver, latest i amb id del commit)
* Enviar imatge al repositori

```bash
# Generar una imatge amb diverses etiquetes (SemVer i latest)
docker build -t myapp:latest -t myapp:v1.2.3 .
# Etiquetat d'imatges per a versions de desenvolupament amb id. commit
docker build -t myapp:dev-$(git rev-parse --short HEAD) .

# Pujar la imatge al registre
docker push myapp:latest
docker push myapp:v1.2.3

```

## ⚡ Automatització amb GitHub Actions

Pots afegir un workflow de GitHub Actions per automatitzar tot el procés.

[Publishing and installing a package with GitHub Actions](https://docs.github.com/en/packages/managing-github-packages-using-github-actions-workflows/publishing-and-installing-a-package-with-github-actions#upgrading-a-workflow-that-accesses-ghcrio)

* Exemple de generació de imatge (SemVer i latest) i push de la imatge al repositori de Github
  * *docker/metadata-action*
    * Extreu la versió el tag de la **versió SemVer** i també per defecte el tag **latest**
  * *docker/build-push-action*
    * Requereix que el **GITHUB_TOKEN** disposi del permís `packages : write` per poder fer el push de la imatge
    * Utilitza les variables **tags** i **labels** obtingudes a l'step *docker/metadata-action*

```yaml
name: Build and Push Docker Image

on:
  workflow_dispatch:
  release:
    types: [created]
  push:
    tags:
      - 'v*.*.*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write
      attestations: write
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}

      - name: Log in to the Container registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker images
        id: push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./docker/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

## Bàsics

### Nginx per test

```bash
# build
docker build -f docker/Dockerfile -t ghcr.io/xsolsonac/versiontagging:1.0.0 .

# run
docker run --name versiontagging-nginx --rm -p 8080:80 ghcr.io/xsolsonac/versiontagging:1.0.0
```

### Git

```bash
git commit -am "Canvis versió...."

git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```
