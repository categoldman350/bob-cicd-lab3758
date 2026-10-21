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


        // ── Lab 4: Linting ───────────────────────────────────────
        //    Add a lint stage + Bob lint analysis + parse and post
        //    Jenkins report as a PR comment.
        //    See labs/lab4/LINTING.md.
        stage('Run Linters') {
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                script {
                    echo '════════════════════════════════════════════════════════'
                    echo '  🔍 Running Linters'
                    echo "  Started: ${new Date()}"
                    echo '════════════════════════════════════════════════════════'

                    // Create lint-results directory
                    sh 'mkdir -p lint-results'

                    catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                        // Run Checkstyle in Maven container
                        echo '→ Running Checkstyle...'
                        sh '''
                            mvn -f order-service/pom.xml checkstyle:check || true
                            cp order-service/target/checkstyle-result.xml lint-results/checkstyle.xml || true
                        '''

                        // Run other linters in lint-tools container
                        container('lint-tools') {
                            // Run Hadolint on Dockerfile
                            echo '→ Running Hadolint...'
                            sh '''
                                hadolint order-service/Dockerfile > lint-results/hadolint.txt 2>&1 || true
                            '''

                            // Run Checkov on deployment manifests
                            echo '→ Running Checkov...'
                            sh '''
                                checkov -d order-service/deploy-flawed/ --compact --quiet > lint-results/checkov.txt 2>&1 || true
                            '''

                            // Run KubeLinter on deployment manifests
                            echo '→ Running KubeLinter...'
                            sh '''
                                kube-linter lint order-service/deploy-flawed/ > lint-results/kubelinter.txt 2>&1 || true
                            '''
                        }
                    }

                    echo ''
                    echo '✅ Linters completed'
                    echo "  Completed: ${new Date()}"
                    echo '════════════════════════════════════════════════════════'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'lint-results/*',
                                   allowEmptyArchive: true,
                                   fingerprint: true
                }
            }
        }

        stage('Bob Lint Analysis') {
            when {
                expression {
                    return fileExists('lint-results/checkstyle.xml') ||
                           fileExists('lint-results/hadolint.txt') ||
                           fileExists('lint-results/checkov.txt') ||
                           fileExists('lint-results/kubelinter.txt')
                }
            }
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                script {
                    echo '════════════════════════════════════════════════════════'
                    echo '  🤖 Bob Lint Analysis'
                    echo "  Started: ${new Date()}"
                    echo '════════════════════════════════════════════════════════'

                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        // Build prompt for Bob to read lint results and source files
                        def prompt = """
Analyze the linting results from multiple tools and provide a consolidated report.

Read and analyze the following files:
- lint-results/checkstyle.xml
- lint-results/hadolint.txt
- lint-results/checkov.txt
- lint-results/kubelinter.txt
- order-service/src/
- order-service/Dockerfile
- order-service/deploy-flawed/

Your analysis should:
1. Group findings by severity (Critical, High, Medium, Low)
2. Identify overlapping findings across different tools
3. Highlight the highest-priority issues that need immediate attention
4. Provide concrete remediation recommendations with code examples
5. Suggest a recommended fix order based on risk and impact

Output format: Use the sections defined in the pipeline-lint-analyzer mode:
- Executive Summary
- Highest Priority Findings
- Findings by Tool
- Cross-Tool Themes
- Recommended Fix Order
"""

                        // Run Bob analysis
                        echo '→ Running Bob lint analysis...'
                        def analysis = askBob(prompt, 'pipeline-lint-analyzer')

                        // Save full analysis report
                        writeFile file: 'bob-lint-report.md', text: analysis
                        echo '✅ Full lint report saved: bob-lint-report.md'

                        // Generate PR comment summary
                        echo ''
                        echo '→ Generating PR comment summary...'
                        def commentPrompt = """
Read bob-lint-report.md and create a concise PR comment summary (max 500 words).

The summary should:
- Start with an overall assessment (✅ Clean / ⚠️ Issues Found / 🔴 Critical Issues)
- List top 3-5 most critical findings with file locations
- Provide a quick action list for the developer
- End with a link reference to the full report

Keep it brief, actionable, and developer-friendly.
"""

                        def prComment = askBob(commentPrompt, 'pipeline-lint-analyzer')
                        writeFile file: 'bob-lint-pr-comment.md', text: prComment
                        echo '✅ PR comment saved: bob-lint-pr-comment.md'

                        // Print summary to console
                        echo ''
                        echo '════════════════════════════════════════════════════════'
                        echo '  📊 Lint Analysis Summary'
                        echo '════════════════════════════════════════════════════════'

                        // Extract key metrics from analysis (using Jenkins-safe string methods)
                        def criticalCount = analysis.toLowerCase().split('critical').length - 1
                        def highCount = analysis.toLowerCase().split('high').length - 1

                        echo "  Critical Issues: ${criticalCount > 0 ? '🔴 ' + criticalCount : '✅ 0'}"
                        echo "  High Priority: ${highCount > 0 ? '🟡 ' + highCount : '✅ 0'}"
                        echo ''
                        echo '  📄 Full Report: bob-lint-report.md (archived)'
                        echo '  💬 PR Comment: bob-lint-pr-comment.md (archived)'
                        echo ''

                        if (criticalCount > 0) {
                            echo '  ⚠️  CRITICAL ISSUES FOUND - Review Required'
                            currentBuild.result = 'UNSTABLE'
                        }
                    }

                    echo "  Completed: ${new Date()}"
                    echo '════════════════════════════════════════════════════════'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bob-lint-report.md,bob-lint-pr-comment.md',
                                   allowEmptyArchive: true,
                                   fingerprint: true
                }
            }
        }
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