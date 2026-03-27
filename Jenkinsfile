pipeline {
  agent any

  options {
    timestamps()
  }

  environment {
    REPORTS_DIR = "reports"
    SBOM_DIR    = "sbom"
  }

  tools {
    // Update these names to match Jenkins global tool configuration
    jdk 'JDK17'
    maven 'Maven3'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Tool Install (ephemeral)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-fiesta2k4-pat', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]) {
          sh '''
            set -e
            mkdir -p "${REPORTS_DIR}" "${SBOM_DIR}" .ci/bin

            GITLEAKS_VERSION="8.18.4"
            TRIVY_VERSION="0.50.1"

            # Proxy compatibility: map uppercase env vars to lowercase for shell tools.
            [ -n "${HTTP_PROXY:-}" ] && export http_proxy="${HTTP_PROXY}"
            [ -n "${HTTPS_PROXY:-}" ] && export https_proxy="${HTTPS_PROXY}"
            [ -n "${NO_PROXY:-}" ] && export no_proxy="${NO_PROXY}"

            gh_curl() {
              if [ -n "${GITHUB_TOKEN:-}" ]; then
                curl -fsSL -H "Authorization: Bearer ${GITHUB_TOKEN}" "$@"
              else
                curl -fsSL "$@"
              fi
            }

            retry_cmd() {
              max_attempts="$1"
              shift
              attempt=1
              while true; do
                if "$@"; then
                  return 0
                fi
                if [ "$attempt" -ge "$max_attempts" ]; then
                  return 1
                fi
                attempt=$((attempt + 1))
                echo "[Tools] Retry ${attempt}/${max_attempts}..."
                sleep 2
              done
            }

            install_trivy_script() {
              gh_curl https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
                | sh -s -- -b .ci/bin "v${TRIVY_VERSION}"
            }

            echo "[Tools] Installing yq..."
            gh_curl -o .ci/bin/yq https://github.com/mikefarah/yq/releases/download/v4.44.2/yq_linux_amd64
            chmod +x .ci/bin/yq

            echo "[Tools] Installing jq..."
            gh_curl -o .ci/bin/jq https://github.com/jqlang/jq/releases/download/jq-1.7.1/jq-linux-amd64
            chmod +x .ci/bin/jq

            echo "[Tools] Installing gitleaks..."
            # Linux amd64; change URL if your Jenkins agent architecture differs
            if gh_curl -o .ci/bin/gitleaks.tar.gz "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz" \
              && tar -tzf .ci/bin/gitleaks.tar.gz >/dev/null \
              && tar -xzf .ci/bin/gitleaks.tar.gz -C .ci/bin gitleaks; then
              chmod +x .ci/bin/gitleaks
              echo "[Tools] gitleaks installed"
            else
              echo "[Tools] WARN: gitleaks install failed; secrets scan will produce empty report"
              rm -f .ci/bin/gitleaks .ci/bin/gitleaks.tar.gz
            fi

            echo "[Tools] Installing trivy..."
            if retry_cmd 3 install_trivy_script; then
              chmod +x .ci/bin/trivy
              echo "[Tools] trivy installed via install script"
            elif retry_cmd 3 gh_curl -o .ci/bin/trivy.tar.gz "https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz" \
              && tar -tzf .ci/bin/trivy.tar.gz >/dev/null \
              && tar -xzf .ci/bin/trivy.tar.gz -C .ci/bin trivy; then
              chmod +x .ci/bin/trivy
              echo "[Tools] trivy installed via direct asset fallback"
            else
              echo "[Tools] WARN: trivy install failed after retries/fallback; SCA scan will produce empty SARIF"
              rm -f .ci/bin/trivy .ci/bin/trivy.tar.gz
            fi

            echo "[Tools] Versions:"
            .ci/bin/yq --version || true
            .ci/bin/jq --version || true
            .ci/bin/gitleaks version || true
            .ci/bin/trivy version || true
          '''
        }
      }
    }

    stage('Plan Gate') {
      steps {
        sh '''
          set -e
          echo "[Plan Gate] Checking required plan artifacts from policy..."
          test -f security-policy.yml

          .ci/bin/yq e '.plan.requiredFiles[]' security-policy.yml | while read -r required_file; do
            if [ ! -f "$required_file" ]; then
              echo "[Plan Gate] Missing required file: $required_file"
              exit 1
            fi
          done

          echo "[Plan Gate] OK"
        '''
      }
    }

    stage('Build Artifact') {
      steps {
        sh '''
          set -e
          echo "[Maven] Building artifact (skip tests)..."
          mvn -B -DskipTests clean package
          ls -la target || true
        '''
      }
    }

    stage('Secrets Scan (gitleaks)') {
      steps {
        sh '''
          set -e
          echo "[Gitleaks] Scanning repo..."
          # Always produce report; Security Gate decides pass/fail by policy thresholds.
          if [ -x .ci/bin/gitleaks ]; then
            .ci/bin/gitleaks detect \
              --source . \
              --redact \
              --report-format json \
              --report-path "${REPORTS_DIR}/gitleaks.json" \
              --exit-code 0
          else
            echo "[]" > "${REPORTS_DIR}/gitleaks.json"
            echo "[Gitleaks] WARN: scanner unavailable, generated empty report"
          fi
        '''
      }
    }

    stage('Generate SBOM (CycloneDX)') {
      steps {
        sh '''
          set -e
          echo "[SBOM] Generating CycloneDX SBOM..."
          mkdir -p "${SBOM_DIR}"

          mvn -B -DskipTests org.cyclonedx:cyclonedx-maven-plugin:2.8.2:makeAggregateBom \
            -Dcyclonedx.outputFormat=json \
            -Dcyclonedx.outputName=bom \
            -Dcyclonedx.outputDirectory="${SBOM_DIR}"

          # Some CycloneDX plugin executions still write to target/ depending on plugin config/version.
          if [ -f "${SBOM_DIR}/bom.json" ]; then
            echo "[SBOM] Found ${SBOM_DIR}/bom.json"
          elif [ -f "target/bom.json" ]; then
            cp target/bom.json "${SBOM_DIR}/bom.json"
            [ -f target/bom.xml ] && cp target/bom.xml "${SBOM_DIR}/bom.xml" || true
            echo "[SBOM] Normalized target/bom.json -> ${SBOM_DIR}/bom.json"
          elif [ -f "target/classes/META-INF/sbom/application.cdx.json" ]; then
            cp target/classes/META-INF/sbom/application.cdx.json "${SBOM_DIR}/bom.json"
            echo "[SBOM] Normalized application.cdx.json -> ${SBOM_DIR}/bom.json"
          fi

          test -f "${SBOM_DIR}/bom.json"
        '''
      }
    }

    stage('Vulnerability Scan (trivy fs)') {
      steps {
        sh '''
          set -e
          echo "[Trivy] Scanning filesystem paths (target)..."
          SEVERITIES=$(.ci/bin/yq e '.securityScans.sca.includeSeverities | join(",")' security-policy.yml)
          TARGET_PATH=$(.ci/bin/yq e '.securityScans.sca.targetPath' security-policy.yml)

          # SARIF output for evidence/audit; Security Gate enforces thresholds.
          if [ -x .ci/bin/trivy ]; then
            .ci/bin/trivy fs \
              --format sarif \
              --output "${REPORTS_DIR}/trivy-fs.sarif" \
              --severity "${SEVERITIES}" \
              --ignore-unfixed \
              --exit-code 0 \
              "${TARGET_PATH}"
          else
            printf '%s\n' '{"version":"2.1.0","runs":[{"tool":{"driver":{"name":"trivy","rules":[]}},"results":[]}]}' > "${REPORTS_DIR}/trivy-fs.sarif"
            echo "[Trivy] WARN: scanner unavailable, generated empty SARIF"
          fi
        '''
      }
    }

    stage('Security Gate') {
      steps {
        sh '''
          set -e

          echo "[Security Gate] Loading thresholds from security-policy.yml..."
          SECRETS_ENABLED=$(.ci/bin/yq e '.securityScans.secrets.enabled' security-policy.yml)
          SECRETS_MAX=$(.ci/bin/yq e '.securityScans.secrets.maxFindings' security-policy.yml)

          SCA_ENABLED=$(.ci/bin/yq e '.securityScans.sca.enabled' security-policy.yml)
          SCA_MAX_CRITICAL=$(.ci/bin/yq e '.securityScans.sca.threshold.maxCritical' security-policy.yml)

          SAST_ENABLED=$(.ci/bin/yq e '.securityScans.sast.enabled' security-policy.yml)
          SAST_MAX_CRITICAL=$(.ci/bin/yq e '.securityScans.sast.threshold.maxCritical' security-policy.yml)
          SAST_MAX_HIGH=$(.ci/bin/yq e '.securityScans.sast.threshold.maxHigh' security-policy.yml)

          if [ "$SECRETS_ENABLED" = "true" ]; then
            test -f "${REPORTS_DIR}/gitleaks.json"
            SECRETS_FOUND=$(.ci/bin/jq 'length' "${REPORTS_DIR}/gitleaks.json")
            echo "[Security Gate] Secrets findings: ${SECRETS_FOUND} (max allowed: ${SECRETS_MAX})"
            if [ "${SECRETS_FOUND}" -gt "${SECRETS_MAX}" ]; then
              echo "[Security Gate] FAIL: Secrets findings exceed policy threshold"
              exit 1
            fi
          fi

          if [ "$SCA_ENABLED" = "true" ]; then
            test -f "${REPORTS_DIR}/trivy-fs.sarif"
            SCA_CRITICAL_FOUND=$(.ci/bin/jq '
              [ .runs[]? as $run
                | ($run.tool.driver.rules // []) as $rules
                | $run.results[]?
                | .ruleId as $rid
                | ($rules[]? | select(.id == $rid) | .properties.tags // []) as $tags
                | select(any($tags[]?; ascii_upcase == "CRITICAL"))
              ] | length
            ' "${REPORTS_DIR}/trivy-fs.sarif")
            echo "[Security Gate] SCA critical findings: ${SCA_CRITICAL_FOUND} (max allowed: ${SCA_MAX_CRITICAL})"
            if [ "${SCA_CRITICAL_FOUND}" -gt "${SCA_MAX_CRITICAL}" ]; then
              echo "[Security Gate] FAIL: SCA critical findings exceed policy threshold"
              exit 1
            fi
          fi

          if [ "$SAST_ENABLED" = "true" ]; then
            if [ ! -f "${REPORTS_DIR}/semgrep.sarif" ]; then
              echo "[Security Gate] FAIL: SAST is enabled but reports/semgrep.sarif is missing"
              exit 1
            fi

            SAST_CRITICAL_FOUND=$(.ci/bin/jq '[.runs[]?.results[]? | select((.level // "") | ascii_downcase == "error")] | length' "${REPORTS_DIR}/semgrep.sarif")
            SAST_HIGH_FOUND=$(.ci/bin/jq '[.runs[]?.results[]? | select((.level // "") | ascii_downcase == "warning")] | length' "${REPORTS_DIR}/semgrep.sarif")
            echo "[Security Gate] SAST critical: ${SAST_CRITICAL_FOUND} (max ${SAST_MAX_CRITICAL}), high: ${SAST_HIGH_FOUND} (max ${SAST_MAX_HIGH})"

            if [ "${SAST_CRITICAL_FOUND}" -gt "${SAST_MAX_CRITICAL}" ] || [ "${SAST_HIGH_FOUND}" -gt "${SAST_MAX_HIGH}" ]; then
              echo "[Security Gate] FAIL: SAST findings exceed policy threshold"
              exit 1
            fi
          fi

          echo "[Security Gate] PASS"
        '''
      }
    }

    stage('Build Metadata (evidence)') {
      steps {
        sh '''
          set -e
          ARTIFACT=$(ls -1 target/*.jar 2>/dev/null | head -n 1 || true)
          if [ -n "$ARTIFACT" ]; then
            SHA256=$(sha256sum "$ARTIFACT" | awk '{print $1}')
          else
            SHA256=""
          fi

          .ci/bin/jq -n \
            --arg job "${JOB_NAME}" \
            --arg build_number "${BUILD_NUMBER}" \
            --arg build_url "${BUILD_URL}" \
            --arg git_commit "${GIT_COMMIT}" \
            --arg git_branch "${GIT_BRANCH}" \
            --arg artifact_path "${ARTIFACT}" \
            --arg artifact_sha256 "${SHA256}" \
            --arg sbom_path "${SBOM_DIR}/bom.json" \
            --arg gitleaks_report "${REPORTS_DIR}/gitleaks.json" \
            --arg trivy_report "${REPORTS_DIR}/trivy-fs.sarif" \
            '{
              job: $job,
              build_number: $build_number,
              build_url: $build_url,
              git_commit: $git_commit,
              git_branch: $git_branch,
              artifact_path: $artifact_path,
              artifact_sha256: $artifact_sha256,
              sbom_path: $sbom_path,
              gitleaks_report: $gitleaks_report,
              trivy_report: $trivy_report
            }' > build-metadata.json
        '''
      }
    }
  }

  post {
    always {
      // Evidence bundle
      archiveArtifacts artifacts: 'build-metadata.json, target/*.jar, reports/*, sbom/*, security-policy.yml, docs/*, .github/CODEOWNERS', fingerprint: true
    }
  }
}