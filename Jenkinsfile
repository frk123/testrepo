pipeline {
    agent any
    
    options {
        // Make the job run quickly
        timeout(time: 2, unit: 'MINUTES')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }
    
    environment {
        // Branch being merged (source branch)
        SOURCE_BRANCH = "${env.CHANGE_BRANCH ?: env.GIT_BRANCH ?: 'unknown'}"
        // Target branch
        TARGET_BRANCH = "${env.CHANGE_TARGET ?: 'pre_prod'}"
        // Pull Request ID
        PR_ID = "${env.CHANGE_ID ?: 'N/A'}"
        // PR Author
        PR_AUTHOR = "${env.CHANGE_AUTHOR ?: 'unknown'}"
    }
    
    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "=========================================="
                    echo "Pre-Prod Branch Protection Check"
                    echo "=========================================="
                    echo "Source Branch: ${SOURCE_BRANCH}"
                    echo "Target Branch: ${TARGET_BRANCH}"
                    echo "PR ID: ${PR_ID}"
                    echo "PR Author: ${PR_AUTHOR}"
                    echo "=========================================="
                }
            }
        }
        
        stage('Verify Target Branch') {
            steps {
                script {
                    if (TARGET_BRANCH != 'pre_prod') {
                        echo "✅ Not targeting pre_prod branch. Check passed."
                        currentBuild.result = 'SUCCESS'
                        currentBuild.description = "Not targeting pre_prod - check skipped"
                        return
                    }
                }
            }
        }
        
        stage('Check Branch Exceptions') {
            steps {
                script {
                    def sourceBranchLower = SOURCE_BRANCH.toLowerCase()
                    
                    // Exception 1: Master branch
                    if (sourceBranchLower == 'master' || sourceBranchLower == 'main') {
                        echo "✅ EXCEPTION: Master branch can merge anytime"
                        currentBuild.result = 'SUCCESS'
                        currentBuild.description = "Master branch exception - approved"
                        env.EXCEPTION_GRANTED = 'true'
                        env.EXCEPTION_REASON = 'Master branch exception'
                        return
                    }
                    
                    // Exception 2: Branch name contains 'revert'
                    if (sourceBranchLower.contains('revert')) {
                        echo "✅ EXCEPTION: Revert branch can merge anytime"
                        currentBuild.result = 'SUCCESS'
                        currentBuild.description = "Revert branch exception - approved"
                        env.EXCEPTION_GRANTED = 'true'
                        env.EXCEPTION_REASON = 'Revert branch exception'
                        return
                    }
                    
                    env.EXCEPTION_GRANTED = 'false'
                }
            }
        }
        
        stage('Check Merge Time Window') {
            when {
                expression { env.EXCEPTION_GRANTED != 'true' }
            }
            steps {
                script {
                    // Get current time in IST
                    def istTime = sh(
                        script: 'TZ="Asia/Kolkata" date +"%Y-%m-%d %H:%M:%S %Z (Hour: %H)"',
                        returnStdout: true
                    ).trim()
                    
                    def currentHour = sh(
                        script: 'TZ="Asia/Kolkata" date +"%H"',
                        returnStdout: true
                    ).trim().toInteger()
                    
                    def currentDay = sh(
                        script: 'TZ="Asia/Kolkata" date +"%A"',
                        returnStdout: true
                    ).trim()
                    
                    echo "=========================================="
                    echo "Current IST Time: ${istTime}"
                    echo "Current Hour: ${currentHour}"
                    echo "Current Day: ${currentDay}"
                    echo "=========================================="
                    
                    // Check if within allowed merge window (8 AM to 6 PM IST)
                    if (currentHour >= 8 && currentHour < 18) {
                        echo "✅ APPROVED: Current time (${currentHour}:00 IST) is within merge window (08:00 - 18:00 IST)"
                        currentBuild.result = 'SUCCESS'
                        currentBuild.description = "Merge approved - within time window"
                        env.TIME_CHECK_PASSED = 'true'
                    } else {
                        def message = """
========================================
❌ MERGE BLOCKED - Outside Time Window
========================================

Current Time: ${istTime}
Allowed Window: 08:00 AM - 06:00 PM IST
Status: BLOCKED ⛔

Your merge to 'pre_prod' is blocked because it's outside the allowed time window.

NEXT STEPS:
-----------
1. Wait until 08:00 AM IST tomorrow to merge
2. If urgent, consider these options:
   • Merge from 'master' branch (allowed anytime)
   • Create a revert branch (allowed anytime)
   • Contact someone with force-push access
3. Re-run this check during allowed hours

EXCEPTIONS (Allowed Anytime):
-----------------------------
✓ Merges from 'master' branch
✓ Branches containing 'revert' in the name
✓ Users with force-push permissions

Branch Details:
--------------
Source: ${SOURCE_BRANCH}
Target: ${TARGET_BRANCH}
PR: #${PR_ID}

For questions, contact your DevOps team.
========================================
"""
                        echo message
                        
                        // Post comment to PR if available
                        if (env.PR_ID && env.PR_ID != 'N/A') {
                            postPRComment(message)
                        }
                        
                        currentBuild.result = 'FAILURE'
                        currentBuild.description = "Blocked - outside merge window (${currentHour}:00 IST)"
                        env.TIME_CHECK_PASSED = 'false'
                        error("Merge blocked: Outside allowed time window")
                    }
                }
            }
        }
        
        stage('Final Status') {
            steps {
                script {
                    if (env.EXCEPTION_GRANTED == 'true') {
                        def successMessage = """
========================================
✅ PRE-PROD MERGE CHECK PASSED
========================================

Status: APPROVED ✓
Reason: ${env.EXCEPTION_REASON}

Branch Details:
--------------
Source: ${SOURCE_BRANCH}
Target: ${TARGET_BRANCH}
PR: #${PR_ID}

You may proceed with the merge.
========================================
"""
                        echo successMessage
                        
                        if (env.PR_ID && env.PR_ID != 'N/A') {
                            postPRComment(successMessage)
                        }
                    } else if (env.TIME_CHECK_PASSED == 'true') {
                        def successMessage = """
========================================
✅ PRE-PROD MERGE CHECK PASSED
========================================

Status: APPROVED ✓
Current Time: Within allowed window (08:00-18:00 IST)

Branch Details:
--------------
Source: ${SOURCE_BRANCH}
Target: ${TARGET_BRANCH}
PR: #${PR_ID}

You may proceed with the merge.
========================================
"""
                        echo successMessage
                        
                        if (env.PR_ID && env.PR_ID != 'N/A') {
                            postPRComment(successMessage)
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "=========================================="
                echo "Build Result: ${currentBuild.result}"
                echo "=========================================="
            }
        }
        success {
            echo "✅ Pre-prod branch protection check PASSED"
        }
        failure {
            echo "❌ Pre-prod branch protection check FAILED"
        }
    }
}

// Helper function to post comments to PR
def postPRComment(String message) {
    // This is a placeholder - implement based on your Git platform
    // For GitHub: Use GitHub API or GitHub Branch Source Plugin
    // For GitLab: Use GitLab API
    // For Bitbucket: Use Bitbucket API
    
    echo "=== PR COMMENT ==="
    echo message
    echo "=================="
    
    // Example for GitHub (requires GitHub plugin and credentials):
    /*
    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
        sh """
            curl -X POST \
            -H "Authorization: token ${GITHUB_TOKEN}" \
            -H "Accept: application/vnd.github.v3+json" \
            https://api.github.com/repos/OWNER/REPO/issues/${PR_ID}/comments \
            -d '{"body":"${message.replaceAll('\n', '\\\\n').replaceAll('"', '\\\\"')}"}'
        """
    }
    */
}