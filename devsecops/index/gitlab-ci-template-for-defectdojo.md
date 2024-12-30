---
description: Gitlab CI generic template to upload reports to DefectDojo using curl
---

# Gitlab CI template for DefectDojo

## Introduction

The template is generic, does not uses the python script and uses curl on which you can easily modify params.

The template will create a new engagement if not existing and add all the CI tests into the same engagement id.

It has a few variables to be defined in the project, group or instance gitlab CICD variables.

```
  DEFECTDOJO_URL: https://demo.defectdojo.org/api/v2
  DEFECTDOJO_USERNAME: admin
  DEFECTDOJO_PASSWORD: "1Defectdojo@demo#appsec"
```

## DefectDojo template

```yaml
spec:
  inputs:
    defectdojo_product_name:
      type: string
    defectdojo_scan_type:
      type: string
    defectdojo_file:
      type: string
    defectdojo_stage:
      type: string
      default: ".post"
    defectdojo_image:
      type: string
      default: alpine

---

variables:
  DEFECTDOJO_ENGAGEMENT_PERIOD: 7
  DEFECTDOJO_ENGAGEMENT_STATUS: "Not Started"
  DEFECTDOJO_ENGAGEMENT_BUILD_SERVER: "http://test.com"
  DEFECTDOJO_ENGAGEMENT_SOURCE_CODE_MANAGEMENT_SERVER: "http://test.com"
  DEFECTDOJO_ENGAGEMENT_ORCHESTRATION_ENGINE: "null"
  DEFECTDOJO_ENGAGEMENT_DEDUPLICATION_ON_ENGAGEMENT: "false"
  DEFECTDOJO_ENGAGEMENT_THREAT_MODEL: "true"
  DEFECTDOJO_ENGAGEMENT_API_TEST: "true"
  DEFECTDOJO_ENGAGEMENT_PEN_TEST: "true"
  DEFECTDOJO_ENGAGEMENT_CHECK_LIST: "true"
  DEFECTDOJO_NOT_ON_MASTER: "false"


defectdojo-$[[ inputs.defectdojo_scan_type ]]:
  stage: $[[ inputs.defectdojo_stage ]]
  image: $[[ inputs.defectdojo_image ]]
  when: always
  before_script:
    - apk add curl jq coreutils
    - export TODAY=$(date +%Y-%m-%d)
    - export TARGET_END=$(date -d "+$DEFECTDOJO_ENGAGEMENT_PERIOD days" +%Y-%m-%d)
    - >
      export DEFECTDOJO_API_KEY=$(curl -s -X POST -H 'content-type: application/json' $DEFECTDOJO_URL/api-token-auth/ -d '{"username": "'$DEFECTDOJO_USERNAME'", "password": "'$DEFECTDOJO_PASSWORD'"}' | jq -r '.token' )
    - >
      export DEFECTDOJO_PRODUCT_ID=$(curl $DEFECTDOJO_URL/products/ \
        -H "Authorization: Token $DEFECTDOJO_API_KEY" \
        -G --data-urlencode "name_exact=$[[ inputs.defectdojo_product_name ]]" | jq -r .results[].id)
    - >
      export DEFECTDOJO_ENGAGEMENT_ID=$(curl $DEFECTDOJO_URL/engagements/ \
        -H "Authorization: Token $DEFECTDOJO_API_KEY" \
        -G --data-urlencode "name=Pipeline #$CI_PIPELINE_ID" \
        --data-urlencode "product=$DEFECTDOJO_PRODUCT_ID" | jq -r .results[].id)
    - >
      if [ -z "$DEFECTDOJO_ENGAGEMENT_ID" ]; then
        export DEFECTDOJO_ENGAGEMENT_ID=$(curl -X POST $DEFECTDOJO_URL/engagements/ \
          -H "accept: application/json" \
          -H "Content-Type: multipart/form-data" \
          -H "Authorization: Token $DEFECTDOJO_API_KEY" \
          -F "tags=GITLAB-CI" \
          -F "name=Pipeline #$CI_PIPELINE_ID" \
          -F "description=$CI_COMMIT_DESCRIPTION" \
          -F "version=$CI_COMMIT_REF_NAME" \
          -F "first_contacted=$TODAY" \
          -F "target_start=$TODAY" \
          -F "target_end=$TARGET_END" \
          -F "reason=string" \
          -F "tracker=$CI_PROJECT_URL" \
          -F "threat_model=$DEFECTDOJO_ENGAGEMENT_THREAT_MODEL" \
          -F "api_test=$DEFECTDOJO_ENGAGEMENT_THREAT_MODEL" \
          -F "pen_test=$DEFECTDOJO_ENGAGEMENT_PEN_TEST" \
          -F "check_list=$DEFECTDOJO_ENGAGEMENT_CHECK_LIST" \
          -F "status=$DEFECTDOJO_ENGAGEMENT_STATUS" \
          -F "engagement_type=CI/CD" \
          -F "build_id=$CI_PIPELINE_ID" \
          -F "commit_hash=$CI_COMMIT_SHORT_SHA" \
          -F "branch_tag=$CI_COMMIT_REF_NAME" \
          -F "deduplication_on_engagement=$DEFECTDOJO_ENGAGEMENT_DEDUPLICATION_ON_ENGAGEMENT" \
          -F "product=$DEFECTDOJO_PRODUCT_ID" \
          -F "source_code_management_uri=$CI_PROJECT_URL" | jq -r .id)
      fi
  script:
    - >
      curl -X POST $DEFECTDOJO_URL/import-scan/ \
        -H  "accept: application/json" \
        -H "Content-Type: multipart/form-data" \
        -H "Authorization: Token $DEFECTDOJO_API_KEY" \
        -F "minimum_severity=Info" \
        -F "active=true" \
        -F "verified=true" \
        -F "scan_type=$[[ inputs.defectdojo_scan_type ]]" \
        -F "close_old_findings=false" \
        -F "push_to_jira=false" \
        -F "file=@$[[ inputs.defectdojo_file ]]" \
        -F "product_name=$[[ inputs.defectdojo_product_name ]]" \
        -F "scan_date=$(date +%Y-%m-%d)" \
        -F "engagement=$DEFECTDOJO_ENGAGEMENT_ID" \
        -F "engagement_name=Pipeline #$CI_PIPELINE_ID"
```

## Usage

Usage is simple, just import the template from local repository or a remote with the following inputs at the end of your project .gitlab-ci.yml

```yaml
sast-bandit:
  stage: test
  image: python:3.8-alpine
  before_script:
    - apk add curl jq
    - pip install -U bandit
  script:
    - bandit -r . -f json --output gl-sast-report.json
  artifacts:
    reports:
      sast: gl-sast-report.json
    paths: [gl-sast-report.json]
    when: always

include:
  - local: defectdojo.yml
    inputs:
      defectdojo_product_name: "Django Vulnerable Server"
      defectdojo_scan_type: "Bandit Scan"
      defectdojo_file: "gl-sast-report.json"
```

All these inputs are case sensitive,so make sure the product name exists in DefectDojo and the scan type is one of the supported formats.

For multiple scans upload include the template several times modifying the inputs. The template will include all the scans into the same engagement id

```yaml
include:
  - local: defectdojo.yml
    inputs:
      defectdojo_product_name: "Django Vulnerable Server"
      defectdojo_scan_type: "Bandit Scan"
      defectdojo_file: "gl-sast-report.json"
  - local: defectdojo.yml
    inputs:
      defectdojo_product_name: "Django Vulnerable Server"
      defectdojo_scan_type: "pip-audit Scan"
      defectdojo_file: "gl-dependency-scanning-report.json"
```
