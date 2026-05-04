pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_REPO_FALLBACK = 'https://github.com/emanamjad61/inventory-app.git'
        TEST_REPO = 'https://github.com/emanamjad61/inventory-tests.git'
        TEST_BRANCH = 'main'
        COMPOSE_PROJECT_NAME = ''
        TEST_IMAGE = ''
    }

    stages {
        stage('Checkout App') {
            steps {
                checkout scm
            }
        }

        stage('Init') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".toLowerCase().replaceAll('[^a-z0-9_-]', '-')
                    env.COMPOSE_PROJECT_NAME = projectName.take(55)
                    env.TEST_IMAGE = "inventory-tests:${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Clone Tests') {
            steps {
                dir('inventory-tests') {
                    git url: "${TEST_REPO}", branch: "${TEST_BRANCH}"
                }
            }
        }

        stage('Prepare Reports') {
            steps {
                sh 'mkdir -p reports'
                writeFile file: 'inventory-tests/run_with_junit.py', text: '''
import os
import sys
import time
import unittest
from xml.etree import ElementTree as ET


class JUnitResult(unittest.TextTestResult):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.all_tests = []
        self.test_times = {}
        self._started_at = {}

    def startTest(self, test):
        self.all_tests.append(test)
        self._started_at[test.id()] = time.time()
        super().startTest(test)

    def stopTest(self, test):
        started_at = self._started_at.get(test.id(), time.time())
        self.test_times[test.id()] = time.time() - started_at
        super().stopTest(test)


def result_map(entries):
    return {test.id(): details for test, details in entries}


def write_junit(result, duration, report_dir):
    os.makedirs(report_dir, exist_ok=True)
    failures = result_map(result.failures)
    errors = result_map(result.errors)
    skipped = result_map(getattr(result, "skipped", []))

    suite = ET.Element(
        "testsuite",
        name="inventory-ui-tests",
        tests=str(result.testsRun),
        failures=str(len(result.failures)),
        errors=str(len(result.errors)),
        skipped=str(len(getattr(result, "skipped", []))),
        time=f"{duration:.3f}",
    )

    for test in result.all_tests:
        test_id = test.id()
        case = ET.SubElement(
            suite,
            "testcase",
            classname=f"{test.__class__.__module__}.{test.__class__.__name__}",
            name=getattr(test, "_testMethodName", test_id),
            time=f"{result.test_times.get(test_id, 0.0):.3f}",
        )
        if test_id in failures:
            failure = ET.SubElement(case, "failure", message="test failure")
            failure.text = failures[test_id]
        if test_id in errors:
            error = ET.SubElement(case, "error", message="test error")
            error.text = errors[test_id]
        if test_id in skipped:
            skipped_node = ET.SubElement(case, "skipped", message=str(skipped[test_id]))
            skipped_node.text = str(skipped[test_id])

    tree = ET.ElementTree(suite)
    tree.write(os.path.join(report_dir, "junit-inventory-tests.xml"), encoding="utf-8", xml_declaration=True)


def write_summary(result, duration, report_dir):
    status = "PASSED" if result.wasSuccessful() else "FAILED"
    lines = [
        f"Inventory UI tests: {status}",
        f"Total: {result.testsRun}",
        f"Failures: {len(result.failures)}",
        f"Errors: {len(result.errors)}",
        f"Skipped: {len(getattr(result, 'skipped', []))}",
        f"Duration: {duration:.2f}s",
    ]
    with open(os.path.join(report_dir, "summary.txt"), "w", encoding="utf-8") as handle:
        handle.write("\\n".join(lines) + "\\n")


def main():
    report_dir = os.environ.get("REPORT_DIR", "/reports")
    os.chdir("/app")
    suite = unittest.defaultTestLoader.discover(".", pattern="tests.py")

    start = time.time()
    runner = unittest.TextTestRunner(resultclass=JUnitResult, verbosity=2)
    result = runner.run(suite)
    duration = time.time() - start

    write_junit(result, duration, report_dir)
    write_summary(result, duration, report_dir)
    return 0 if result.wasSuccessful() else 1


if __name__ == "__main__":
    sys.exit(main())
'''
            }
        }

        stage('Compose Up App') {
            steps {
                sh 'docker compose -p "$COMPOSE_PROJECT_NAME" up -d --build app'
            }
        }

        stage('Wait For App') {
            steps {
                sh '''
                    cid="$(docker compose -p "$COMPOSE_PROJECT_NAME" ps -q app)"
                    if [ -z "$cid" ]; then
                        docker compose -p "$COMPOSE_PROJECT_NAME" ps
                        exit 1
                    fi

                    for i in $(seq 1 30); do
                        status="$(docker inspect -f '{{.State.Health.Status}}' "$cid" 2>/dev/null || true)"
                        if [ "$status" = "healthy" ]; then
                            exit 0
                        fi
                        sleep 2
                    done

                    docker compose -p "$COMPOSE_PROJECT_NAME" ps
                    docker compose -p "$COMPOSE_PROJECT_NAME" logs app
                    exit 1
                '''
            }
        }

        stage('Build Test Image') {
            steps {
                sh 'docker build -t "$TEST_IMAGE" inventory-tests'
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    docker run --rm \
                        --network "${COMPOSE_PROJECT_NAME}_default" \
                        -e APP_URL=http://app:5000 \
                        -e REPORT_DIR=/reports \
                        -v "$PWD/reports:/reports" \
                        "$TEST_IMAGE" \
                        python /app/run_with_junit.py
                '''
            }
        }
    }

    post {
        always {
            script {
                if (env.COMPOSE_PROJECT_NAME?.trim()) {
                    sh 'docker compose -p "$COMPOSE_PROJECT_NAME" logs --no-color app > reports/app.log 2>&1 || true'
                }

                junit allowEmptyResults: true, testResults: 'reports/junit-*.xml'
                archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true

                def appRepo = sh(script: 'git remote get-url origin 2>/dev/null || true', returnStdout: true).trim()
                def appCommit = sh(script: 'git rev-parse --short HEAD 2>/dev/null || true', returnStdout: true).trim()
                def committerEmail = sh(script: "git show -s --format='%ae' HEAD 2>/dev/null || true", returnStdout: true).trim()
                def summary = fileExists('reports/summary.txt')
                    ? readFile('reports/summary.txt').trim()
                    : 'Tests did not produce a summary. Check the Jenkins console log.'

                if (env.COMPOSE_PROJECT_NAME?.trim()) {
                    sh 'docker compose -p "$COMPOSE_PROJECT_NAME" down -v --remove-orphans || true'
                }
                if (env.TEST_IMAGE?.trim()) {
                    sh 'docker rmi "$TEST_IMAGE" || true'
                }

                if (committerEmail) {
                    try {
                        mail(
                            to: committerEmail,
                            subject: "Inventory CI #${env.BUILD_NUMBER}: ${currentBuild.currentResult}",
                            body: """Inventory pipeline finished.

Result: ${currentBuild.currentResult}
App repo: ${appRepo ?: env.APP_REPO_FALLBACK}
Tests repo: ${env.TEST_REPO}
App commit: ${appCommit}
Tests branch: ${env.TEST_BRANCH}
Build: ${env.BUILD_URL}
JUnit report: ${env.BUILD_URL}testReport/

${summary}
"""
                        )
                    } catch (err) {
                        echo "Email notification failed: ${err}"
                    }
                } else {
                    echo 'No committer email found; skipping email notification.'
                }
            }
        }
    }
}
