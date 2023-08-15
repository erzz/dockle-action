# dockle-action
Github Action to run Dockle and report in workflows, pipeline and PR's


## Purpose

This action executes the excellent [Dockle](https://github.com/goodwithtech/dockle) linter for containers that will run numerous checks on an image for Best Practices and against CIS benchmarks.

The action will generate a report file of either JSON or SARIF format for upload to Github or downloading locally. It also produces the report in the stdout of the job's log.

You can set your own threshold for when to fail the job and decide whether a failure should "break the build."

## Available Inputs

| Input               | Default             | Details                                                                                           |
|---------------------|---------------------|---------------------------------------------------------------------------------------------------|
| `image`             | none - mandatory!   | The image you wish to scan. e.g. `alpine:latest` or `myownregistry.com/my-team/my-image:latest`   |
| `report-format`     | json                | The format to generate a report in. Either `json` or `sarif`                                      |
| `report-name`       | dockle-report       | The name of the report (without a file extension as thats added automatically by report-format)   |
| `failure-threshold` | warn                | Threshold for findings to trigger a failure of the job Options are `INFO`, `WARN`, or `FATAL`     |
| `exit-code`         | 1                   | Change to `0` if you don't wish to "break the build" on findings                                  |
| `dockle-version`    | latest              | Specify a version of Dockle to use in the format `1.2.3`, otherwise it uses the latest version    |
| `accept-keywords`   | ""                  | Comma seperated list of acceptable keywords for credential checks e.g. `GPG_KEY,KEYCLOAK_VERSION` |
| `accept-filenames`  | ""                  | Comma seperated list of acceptable file names for credential checks e.g. `id_rsa,id_dsa`          |
| `accept-extensions` | ""                  | Comma seperated list of acceptable file extensions for credential checks e.g. `pem,log`           |
| `timeout`           | "10m"               | Time allowed for the image to be pulled from the registry e.g) `5s`, `5m`...                      |
| `token`             | ${{ github.token }} | Token used to call the GitHub API and retrive the latest dockle version                           |


## Potential Artifacts

### dockle-report.json

The default output of this job is the JSON format of the report and it is often useful for further parsing and manipulation. To upload this report to your Worflow Summary you would add the following step after the scan:

```yaml
- name: Upload Report
  uses: actions/upload-artifact@v3
  if: always()
  with:
    name: Dockle Report
    path: dockle-report.json
```

### dockle-report.sarif

If you change `report-format` to `sarif` then it will be produced in a format compatible with [Github Advanced Security](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security) for integration with the UI. To upload findings to GHAS you would add the following step after the scan:

```yaml
- name: Upload SARIF file
  uses: github/codeql-action/upload-sarif@v2
  with:
    # Path to SARIF file relative to the root of the repository
    sarif_file: dockle-report.sarif
```

## Example usages

### Default settings against a public image

The simplest usage with all default settings would perform a scan, with the job failing with findings of severity WARN but would not break your build.

```yaml
jobs:
  dockle:
    name: Dockle Container Analysis
    runs-on: ubuntu-latest
    steps:
      - name: Run Dockle
        uses: erzz/dockle-action@v1
        with:
          image: alpine:latest
```

### Using a .dockleignore file

In this example you also want to add a [.dockleignore](https://github.com/goodwithtech/dockle#ignore-the-specified-checkpoints) file to whitelist certain findings whilst breaking the build for any finding of severity `FATAL`

The important thing to note here is you must check out your code in a preceding step in order for your `.dockleignore` file to be available.

```yaml
jobs:
  dockle:
    name: Dockle Container Analysis
    runs-on: ubuntu-latest
    steps:
       # Makes sure your .dockleignore file is available to the next step
      - name: Checkout
        uses: actions/checkout@v3

      - name: Run Dockle
        uses: erzz/dockle-action@v1
        with:
          image: alpine:latest
          exit-code: 1
          failure-threshold: fatal
```

### Advanced usage and private registries

In this more complex example we try to use all inputs and upload the resulting SARIF file to GHAS. It includes a rule to ignore .pem files in the credentials checks and renaming the report.

The example is also using an image from a private registry so we will add a preceding step to also authenticate with the registry before trying to pull it in the scan. The step you use will depend on your registry, in this case, its GCR.

```yaml
jobs:
  dockle:
    name: Dockle Container Analysis
    runs-on: ubuntu-latest
    steps:
       # Makes sure your .dockleignore file is available to the next step
      - name: Checkout
        uses: actions/checkout@v3

      # Here you might use a docker login action, or something else
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v0
        with:
          credentials_json: ${{ secrets.SA_JSON_KEY }}

      - name: Run Dockle
        uses: erzz/dockle-action@v1
        with:
          image: eu.gcr.io/my-project/my-image:v1.2.3
          report-format: sarif
          report-name: dockle-results
          failure-threshold: fatal
          exit-code: 1
          dockle-version: 0.4.11
          accept-extensions: pem

      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: super-report.sarif
```
