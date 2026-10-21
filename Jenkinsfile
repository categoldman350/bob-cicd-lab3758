// ═══════════════════════════════════════════════════════════════════
// Base Pipeline (Jenkinsfile)
//
// Starting point for the workshop. Provisions a Kubernetes
// agent pod with three containers — build-tools (Maven + JDK17),
// oc-tools (OpenShift CLI), and bob (IBM Bob CLI) — sharing a
// workspace-volume emptyDir mounted at /workspace on all three,
// with HOME set to /workspace so Bob can read .bob/custom_modes.yaml
// from the checkout.
//
// This file has a single Checkout stage. Each lab adds one more
// stage beneath it; see labs/LAB<N>_*.md for instructions.
//
// Before running: if your OpenShift project is NOT named `jenkins`,
// update the `bob` container image URL below.
// ═══════════════════════════════════════════════════════════════════
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: build-tools
    image: maven:3.9-eclipse-temurin-17
    command: ['sleep', 'infinity']
    workingDir: /workspace
    volumeMounts:
    - name: workspace-volume
      mountPath: /workspace
    env:
    - name: HOME
      value: /workspace
  - name: oc-tools
    image: quay.io/openshift/origin-cli:latest
    command: ['sleep', 'infinity']
    workingDir: /workspace
    volumeMounts:
    - name: workspace-volume
      mountPath: /workspace
    env:
    - name: HOME
      value: /workspace
  - name: lint-tools
    image: image-registry.openshift-image-registry.svc:5000/jenkins/lint-tools:latest
    command: ['sleep', 'infinity']
    workingDir: /workspace
    volumeMounts:
    - name: workspace-volume
      mountPath: /workspace
    env:
    - name: HOME
      value: /workspace
  - name: bob
    image: image-registry.openshift-image-registry.svc:5000/jenkins/bob-cli:latest
    command: ['sleep', 'infinity']
    workingDir: /workspace
    volumeMounts:
    - name: workspace-volume
      mountPath: /workspace
    env:
    - name: BOBSHELL_API_KEY
      valueFrom:
        secretKeyRef:
          name: bob-cli-credentials
          key: BOBSHELL_API_KEY
    - name: BOB_ACCEPT_LICENSE
      value: "true"
    - name: HOME
      value: /workspace
  volumes:
  - name: workspace-volume
    emptyDir: {}
"""
            defaultContainer 'build-tools'
        }
    }

    stages {
        stage('Checkout') {
            steps {
                echo "=== Checking out repository ==="
                checkout scm

                echo "=== Verifying workspace contents ==="
                sh '''
                    echo "Current directory: $(pwd)"
                    echo "Workspace contents:"
                    ls -la
                '''
            }
        }

        // ── Lab 1: PR / Git Diff Review ──────────────────────────
        
        // ── Lab 2: Unit Testing ──────────────────────────────────

        // ── Lab 3: Security Scanning ─────────────────────────────
        //    Add a scanner stage + Bob CVE/vuln analysis.
        //    See labs/lab3/SECURITY_ANALYSIS.md.
        stage('Security Analysis') {
            options {
                timeout(time: 15, unit: 'MINUTES')
            }
            environment {
                SONAR_TOKEN    = credentials('sonarqube-token')
                SONAR_HOST_URL = 'https://sonarqube-sonarqube.apps.itz-y6rg5f.infra01-lb.wdc07.techzone.ibm.com'
                PROJECT_KEY    = "order-service-${env.USER ?: 'jenkins'}-${env.BUILD_NUMBER}"
            }
            steps {
                script {
                    // Serializable results only: String / Integer / Boolean values
                    def securityResults = [
                        sonarqube: [
                            taskStatus: 'NOT_RUN',
                            bugs: 'N/A', vulnerabilities: 'N/A', code_smells: 'N/A',
                            security_hotspots: 'N/A', security_rating: 'N/A',
                            reliability_rating: 'N/A', security_review_rating: 'N/A'
                        ],
                        trivy: [critical: 0, high: 0, medium: 0, scanned: false],
                        bob:   [recommendation: 'NOT_RUN', analyzed: false],
                        risk:  [score: 0, level: 'LOW'],
                        decision: [action: 'PROCEED', message: '']
                    ]

                    echo '════════════════════════════════════════════════════════'
                    echo '  🔒 Security Analysis'
                    echo "  Project key: ${env.PROJECT_KEY}"
                    echo '════════════════════════════════════════════════════════'

                    // ── Phase 1: SonarQube ───────────────────────────────
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        echo '🔍 Phase 1: SonarQube security scan'
                        sh '''
                            mvn -B -f order-service/pom.xml test-compile \
                                org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                                -Dsonar.host.url="$SONAR_HOST_URL" \
                                -Dsonar.token="$SONAR_TOKEN" \
                                -Dsonar.projectKey="$PROJECT_KEY" \
                                -Dsonar.projectName="$PROJECT_KEY"
                        '''

                        // Phase 1a: wait for the Compute Engine task (12 x 10s)
                        echo '⏳ Phase 1a: waiting for SonarQube task completion'
                        def taskStatus = 'UNKNOWN'
                        for (int i = 1; i <= 12; i++) {
                            try {
                                taskStatus = sh(
                                    script: '''curl -sk -u "$SONAR_TOKEN:" "${SONAR_HOST_URL}/api/ce/component?component=${PROJECT_KEY}" | grep -o '"status":"[^"]*"' | head -1 | cut -d'"' -f4 || true''',
                                    returnStdout: true
                                ).trim()
                            } catch (Exception e) {
                                echo "⚠️  Task status check failed (attempt ${i}/12): ${e.message}"
                                taskStatus = 'UNKNOWN'
                            }
                            echo "   Attempt ${i}/12 — task status: ${taskStatus ?: 'n/a'}"
                            if (taskStatus == 'SUCCESS' || taskStatus == 'FAILED') {
                                break
                            }
                            if (i < 12) {
                                sleep time: 10, unit: 'SECONDS'
                            }
                        }
                        securityResults.sonarqube.taskStatus = taskStatus ?: 'UNKNOWN'

                        if (taskStatus == 'FAILED') {
                            error('SonarQube analysis task FAILED')
                        }
                        if (taskStatus != 'SUCCESS') {
                            error("SonarQube task did not complete in time (last status: ${taskStatus ?: 'n/a'})")
                        }
                        echo '✅ SonarQube task completed'

                        // Phase 1b: fetch metrics only after the task is done (6 x 5s)
                        echo '📊 Phase 1b: fetching SonarQube metrics'
                        def metricKeys = 'bugs,vulnerabilities,code_smells,security_hotspots,security_rating,reliability_rating,security_review_rating'
                        for (int i = 1; i <= 6; i++) {
                            def json = ''
                            try {
                                json = sh(
                                    script: "curl -sk -u \"\$SONAR_TOKEN:\" \"\${SONAR_HOST_URL}/api/measures/component?component=\${PROJECT_KEY}&metricKeys=${metricKeys}\" || true",
                                    returnStdout: true
                                ).trim()
                            } catch (Exception e) {
                                echo "⚠️  Metrics fetch failed (attempt ${i}/6): ${e.message}"
                            }

                            def fetched = false
                            if (json) {
                                try {
                                    def sonarData = new groovy.json.JsonSlurper().parseText(json)
                                    if (sonarData.component?.measures && sonarData.component.measures.size() > 0) {
                                        for (m in sonarData.component.measures) {
                                            securityResults.sonarqube[m.metric.toString()] = m.value.toString()
                                        }
                                        fetched = true
                                    }
                                    sonarData = null // avoid non-serializable LazyMap in CPS state
                                } catch (Exception e) {
                                    echo "⚠️  Could not parse metrics response (attempt ${i}/6): ${e.message}"
                                }
                            }
                            if (fetched) {
                                break
                            }
                            echo "   Attempt ${i}/6 — metrics not available yet"
                            if (i < 6) {
                                sleep time: 5, unit: 'SECONDS'
                            }
                        }

                        echo '   SonarQube results:'
                        securityResults.sonarqube.each { k, v -> echo "     ${k}: ${v}" }
                    }

                    // ── Phase 2: Grype ───────────────────────────────────
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        container('build-tools') {
                            echo '🛡️  Phase 2: Grype vulnerability scan'
                            sh 'curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin'
                            sh 'cd order-service && grype dir:. --scope all-layers -o json > grype-report.json'

                            def count = { String sev ->
                                sh(
                                    script: "grep -o '\"severity\": *\"${sev}\"' order-service/grype-report.json 2>/dev/null | wc -l",
                                    returnStdout: true
                                ).trim().toInteger()
                            }
                            securityResults.trivy.critical = count('Critical')
                            securityResults.trivy.high     = count('High')
                            securityResults.trivy.medium   = count('Medium')
                            securityResults.trivy.scanned  = true
                            echo "   Grype — Critical: ${securityResults.trivy.critical}, High: ${securityResults.trivy.high}, Medium: ${securityResults.trivy.medium}"
                        }
                    }

                    // ── Phase 3: CVE analysis with Bob (conditional) ─────
                    def toInt = { String s ->
                        try { return Integer.parseInt(s) } catch (Exception e) { return 0 }
                    }
                    def sonarVulns = toInt(securityResults.sonarqube.vulnerabilities)
                    def totalVulns = sonarVulns + securityResults.trivy.critical + securityResults.trivy.high + securityResults.trivy.medium

                    if (totalVulns > 0) {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            echo "🤖 Phase 3: Bob CVE analysis (${totalVulns} vulnerabilities)"
                            def prompt = """
You are reviewing the security scan results of the order-service Java application.

SonarQube results:
- Task status: ${securityResults.sonarqube.taskStatus}
- Bugs: ${securityResults.sonarqube.bugs}
- Vulnerabilities: ${securityResults.sonarqube.vulnerabilities}
- Code smells: ${securityResults.sonarqube.code_smells}
- Security hotspots: ${securityResults.sonarqube.security_hotspots}
- Security rating: ${securityResults.sonarqube.security_rating}
- Reliability rating: ${securityResults.sonarqube.reliability_rating}
- Security review rating: ${securityResults.sonarqube.security_review_rating}

Grype results (dependency CVEs):
- Critical: ${securityResults.trivy.critical}
- High: ${securityResults.trivy.high}
- Medium: ${securityResults.trivy.medium}

The full Grype report is in order-service/grype-report.json (read it for CVE details).

Provide, as plain text:
1. Risk assessment (overall, with justification)
2. Top 3 vulnerabilities (CVE id, affected package, why it matters)
3. Remediation steps (specific version upgrades or code changes)
4. Deployment recommendation

End your answer with exactly one final line in the form:
DEPLOYMENT RECOMMENDATION: BLOCK
or
DEPLOYMENT RECOMMENDATION: WARN
or
DEPLOYMENT RECOMMENDATION: PROCEED
"""
                            // askBob runs inside container('bob')
                            def analysis = askBob(prompt, 'code')
                            writeFile file: 'bob-cve-analysis.txt', text: analysis
                            securityResults.bob.analyzed = true

                            def recommendation = 'UNKNOWN'
                            for (String line : analysis.readLines().reverse()) {
                                if (line.toUpperCase().contains('DEPLOYMENT RECOMMENDATION')) {
                                    def up = line.toUpperCase()
                                    if (up.contains('BLOCK')) { recommendation = 'BLOCK' }
                                    else if (up.contains('WARN')) { recommendation = 'WARN' }
                                    else if (up.contains('PROCEED')) { recommendation = 'PROCEED' }
                                    break
                                }
                            }
                            securityResults.bob.recommendation = recommendation

                            echo '════════════════════════════════════════════════════════'
                            echo '  Bob — CVE Analysis'
                            echo '════════════════════════════════════════════════════════'
                            echo analysis
                            echo '════════════════════════════════════════════════════════'
                            echo "   Bob recommendation: ${recommendation}"
                        }
                    } else {
                        echo '✅ Phase 3: no vulnerabilities found — skipping Bob CVE analysis'
                    }

                    // ── Phase 4: Risk level ──────────────────────────────
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        echo '📈 Phase 4: calculating risk level'
                        Integer score = (toInt(securityResults.sonarqube.vulnerabilities) * 10) +
                                        (toInt(securityResults.sonarqube.security_hotspots) * 5) +
                                        (toInt(securityResults.sonarqube.bugs) * 2) +
                                        (securityResults.trivy.critical * 20) +
                                        (securityResults.trivy.high * 10)
                        String level = score >= 50 ? 'CRITICAL' : score >= 30 ? 'HIGH' : score >= 10 ? 'MEDIUM' : 'LOW'
                        securityResults.risk.score = score
                        securityResults.risk.level = level
                        echo "   Risk score: ${score} → ${level}"
                    }

                    // ── Phase 5: Deployment decision ─────────────────────
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        echo '🚦 Phase 5: deployment decision'
                        def level = securityResults.risk.level
                        if (level == 'CRITICAL') {
                            securityResults.decision.action = 'BLOCK'
                            securityResults.decision.message = 'Deployment BLOCKED: critical security risk.'
                            echo "🛑 ${securityResults.decision.message}"
                            currentBuild.result = 'UNSTABLE'
                        } else if (level == 'HIGH') {
                            securityResults.decision.action = 'WARN'
                            securityResults.decision.message = 'Deployment WARNING: high security risk — manual review recommended.'
                            echo "⚠️  ${securityResults.decision.message}"
                        } else {
                            securityResults.decision.action = 'PROCEED'
                            securityResults.decision.message = "Deployment may PROCEED with approval (${level} risk)."
                            echo "✅ ${securityResults.decision.message}"
                        }
                    }

                    // ── Phase 6: Consolidated report ─────────────────────
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        echo '📝 Phase 6: generating consolidated report'
                        def sq = securityResults.sonarqube
                        def tr = securityResults.trivy
                        def cveSection = fileExists('bob-cve-analysis.txt') ?
                            readFile('bob-cve-analysis.txt') :
                            'No CVE analysis performed (no vulnerabilities detected or analysis unavailable).'

                        def report = """# Security Analysis Report

**Build:** #${env.BUILD_NUMBER}
**Project key:** ${env.PROJECT_KEY}
**Generated:** ${new Date()}

## Executive Summary

- **Risk level:** ${securityResults.risk.level} (score ${securityResults.risk.score})
- **Deployment decision:** ${securityResults.decision.action}
- **Grype findings:** ${tr.critical} critical, ${tr.high} high, ${tr.medium} medium
- **SonarQube vulnerabilities:** ${sq.vulnerabilities}

## SonarQube Results

| Metric | Value |
|--------|-------|
| Task status | ${sq.taskStatus} |
| Bugs | ${sq.bugs} |
| Vulnerabilities | ${sq.vulnerabilities} |
| Code smells | ${sq.code_smells} |
| Security hotspots | ${sq.security_hotspots} |
| Security rating | ${sq.security_rating} |
| Reliability rating | ${sq.reliability_rating} |
| Security review rating | ${sq.security_review_rating} |

## Grype Results

| Severity | Count |
|----------|-------|
| Critical | ${tr.critical} |
| High | ${tr.high} |
| Medium | ${tr.medium} |

Scan completed: ${tr.scanned}

## CVE Analysis (Bob)

${cveSection}

## Deployment Recommendation

- **Pipeline decision:** ${securityResults.decision.action} — ${securityResults.decision.message}
- **Bob recommendation:** ${securityResults.bob.recommendation}
"""
                        writeFile file: 'security-analysis-report.md', text: report
                        echo '   Report: security-analysis-report.md'
                    }

                    echo '════════════════════════════════════════════════════════'
                    echo "  Security Analysis complete — ${securityResults.risk.level} risk, decision: ${securityResults.decision.action}"
                    echo '════════════════════════════════════════════════════════'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'security-analysis-report.md,bob-cve-analysis.txt,order-service/grype-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        // ── Lab 4: Linting ───────────────────────────────────────
        //    Add a lint stage + Bob lint analysis + parse and post
        //    Jenkins report as a PR comment.
        //    See labs/lab4/LINTING.md.

    }

    post {
        always {
            echo "=== Pipeline Complete ==="
            echo "Result: ${currentBuild.result ?: 'SUCCESS'}"
        }
    }
}
// ?? Helper: ask Bob, optionally with a specific custom mode ???????????????????
// Writes the prompt to a tempfile in the shared workspace and runs `bob` in
// the bob container.
// Returns the analysis as a string. Using a tempfile (instead of inlining the
// prompt on the command line) avoids shell-escaping issues when the prompt
// contains quotes, backticks, or newlines ? common with diffs.
def askBob(String prompt, String mode = null) {
    container('bob') {
        def promptFile = ".bob-prompt-${System.currentTimeMillis()}.txt"
        writeFile file: promptFile, text: prompt

        def modeFlag = mode ? "--mode ${mode}" : ""
        def analysis = sh(
            script: """cat ${promptFile} | bob run ${modeFlag} --accept-license""",
            returnStdout: true
        ).trim()

        sh "rm -f ${promptFile}"
        return analysis
    }
}