# Configuración del Workflow SLSA Go Release Generator

## Descripción
Este documento explica cómo configurar el workflow de **SLSA Go Release** en GitHub Actions para generar artefactos Go firmados y provenance verificable utilizando el framework SLSA.

El generador oficial de SLSA para Go permite:
- Compilar binarios Go de manera segura
- Generar attestations/provenance
- Firmar metadata automáticamente
- Mejorar la seguridad del pipeline CI/CD

## ¿Qué es SLSA?
**SLSA (Supply-chain Levels for Software Artifacts)** es un framework de seguridad diseñado para proteger la cadena de suministro de software.

Permite:
- Verificar cómo fue construido un artefacto
- Garantizar integridad
- Detectar modificaciones maliciosas
- Generar provenance criptográficamente verificable

### Referencias
- https://slsa.dev
- https://github.com/slsa-framework/slsa-github-generator


# Arquitectura del Workflow
El workflow utiliza el builder oficial de Go de SLSA.

El proceso:
1. Checkout del código
2. Setup de Go
3. Compilación del binario
4. Generación automática de provenance
5. Firma OIDC de attestations
6. Publicación de assets del release

## Requisitos
Antes de comenzar:
- Repositorio GitHub
- Proyecto Go funcional
- GitHub Actions habilitado
- Archivo `go.mod`
- Releases mediante tags Git
- Permisos OIDC habilitados

## Estructura Recomendada

```text
.
├── cmd/
│   └── app/
│       └── main.go
├── go.mod
└── .github/
    └── workflows/
        └── release.yml
```

# Configuración del Workflow
- Nos vamos al apartado Actions del repositorio y le damos a New Workflow
<img width="668" height="186" alt="image" src="https://github.com/user-attachments/assets/f37baa7d-2196-4008-80c6-7a4af0af81e6" /> <br>

- Buscamos _SLSA Go releaser_ y le damos a Configurar, esto nos creará un YAML de ejemplo
<img width="1043" height="446" alt="image" src="https://github.com/user-attachments/assets/503cfa34-18eb-4e1a-9f37-367f4aa3dc2b" />
<br>

```yaml
# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by separate terms of service, privacy policy, and support documentation.

# This workflow lets you compile your Go project using a SLSA3 compliant builder.
# This workflow will generate a so-called "provenance" file describing the steps that were performed to generate the final binary.
# The project is an initiative of the OpenSSF (openssf.org) and is developed at https://github.com/slsa-framework/slsa-github-generator.
# The provenance file can be verified using https://github.com/slsa-framework/slsa-verifier. For more information about SLSA and how it improves the supply-chain, visit slsa.dev.

name: SLSA Go releaser
on:
  workflow_dispatch:
  release:
    types: [created]

permissions: read-all

jobs:
  # ========================================================================================================================================
  #     Prerequesite: Create a .slsa-goreleaser.yml in the root directory of your project.
  #       See format in https://github.com/slsa-framework/slsa-github-generator/blob/main/internal/builders/go/README.md#configuration-file
  #=========================================================================================================================================
  build:
    permissions:
      id-token: write # To sign.
      contents: write # To upload release assets.
      actions: read   # To read workflow path.
    uses: slsa-framework/slsa-github-generator/.github/workflows/builder_go_slsa3.yml@v2.1.0
    with:
      go-version: 1.17

```

### Explicación de Permisos

| Permiso | Uso |
|---|---|
| `contents: write` | Subir artefactos al release |
| `id-token: write` | Firma OIDC |
| `actions: read` | Metadata del workflow |

- Este Workflow buscara en la raiz del repositorio el fichero _.slsa-goreleaser.yml_ así que lo creamos con el siguiente contenido <br>

```yaml
version: 1

# (Optional) List of env variables used during compilation.
env:
  - GO111MODULE=on
  - CGO_ENABLED=0

# (Optional) Flags for the compiler.
flags:
  - -trimpath
  - -tags=netgo

# The OS to compile for. `GOOS` env variable will be set to this value.
goos: linux

# The architecture to compile for. `GOARCH` env variable will be set to this value.
goarch: amd64

# Entrypoint to compile.
main: ./cmd/modern-go-application   # Set the path to the folder containing main.go

# Binary output name.
# {{ .Os }} will be replaced by goos field in the config file.
# {{ .Arch }} will be replaced by goarch field in the config file.
binary: binary-{{ .Os }}-{{ .Arch }}
```

## Resultado Esperado

Al ejecutar el workflow:
- Se compila el binario Go
- Se genera provenance SLSA
- La provenance queda firmada
- Se publica automáticamente en GitHub Releases

<img width="1444" height="218" alt="image" src="https://github.com/user-attachments/assets/44f91818-5825-4621-97e7-6eab9ce1bb33" /><br>

Archivos generados:

```text
myapp
provenance.intoto.jsonl
```
### Nota:
* Cuando se lance de forma manual, creará los artefactos pero no los subirá ya que no hay release a dónde subirlo
* Usar tags versionados y siempre referenciar versiones fijas (Usar @v1.9.0 en lugar de @main)
* Proteger Branches y Releases, recomendado:
  - Branch protection rules
  - Signed commits
  - Required reviews
  - Protected environments
* Mantener Dependencias Actualizadas, actualizar módulos Go:
  ```bash
  go mod tidy
  go get -u ./...
  ```
* Si en lugar de lanzar en _Release_ lo configuramos de esta forma
  ```yaml
  on:
    push:
      tags:
        - 'v*'
  ```
  Entonces crear un tag para disparar el workflow:
  ```bash
  git tag v1.0.0
  git push origin v1.0.0
  ```

## Verificación de Provenance

1. Instalar el verificador oficial:

```bash
go install github.com/slsa-framework/slsa-verifier/v2/cli/slsa-verifier@latest
```

2. Verificar un Binario

```bash
slsa-verifier verify-artifact \
  --provenance-path provenance.intoto.jsonl \
  --source-uri github.com/mi-org/mi-repo \
  myapp
```

# Referencias Oficiales
- https://slsa.dev
- https://github.com/slsa-framework/slsa-github-generator
- https://github.com/slsa-framework/slsa-github-generator/tree/main/internal/builders/go
- https://github.com/slsa-framework/slsa-verifier
