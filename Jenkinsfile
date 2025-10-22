pipeline {
    agent any
    
    parameters {
        choice(
            name: 'TARGET_BRANCH', 
            choices: ['pre_prod', 'main', 'develop'], 
            description: 'Target branch to merge into'
        )
        string(
            name: 'SOURCE_BRANCH', 
            defaultValue: 'feature/test-branch', 
            description: 'Source branch name'
        )
        string(
            name: 'PR_NUMBER', 
            defaultValue: '123', 
            description: 'Pull Request number'
        )
        booleanParam(
            name: 'SIMULATE_OFF_HOURS',
            defaultValue: false,
            description: 'Simulate merge attempt outside 8 AM - 6 PM IST'
        )
    }
    
    options {
        timeout(time: 2, unit: 'MINUTES')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }
    
    environment {
        SOURCE_BRANCH = "${params.SOURCE_BRANCH}"
        TARGET_BRANCH = "${params.TARGET_BRANCH}"
        PR_ID = "${params.PR_NUMBER}"
        PR_AUTHOR = "test-user"
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
                    echo "Simulate Off Hours: ${params.SIMULATE_OFF_HOURS}"
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
                        env.SKIP_REMAINING = 'true'
                    } else {
                        env.SKIP_REMAINING = 'false'
                    }
                }
            }
        }
        
        stage('Check Branch Exceptions') {
            when {
                expression { env.SKIP_REMAINING != 'true' }
            }
            steps {
                script {
                    def sourceBranchLower = SOURCE_BRANCH.toLowerCase()
                    
                    echo "Checking branch: ${sourceBranchLower}"
                    
                    // Exception 1: Master branch
                    if (sourceBranchLower == 'master' || sourceBranchLower == 'main' || sourceBranchLower == 'origin/master' || sourceBranchLower == 'origin/main') {
                        echo "✅ EXCEPTION: Master/Main branch can merge anytime"
                        currentBuild.result = 'SUCCESS'
                        currentBuild.description = "Master branch exception - approved"
                        env.EXCEPTION_GRANTED = 'true'
                        env.EXCEPTION_REASON = 'Master/Main branch exception'
                        env.SKIP_TIME_CHECK = 'true'
                        return
                    }
                    
                    // Exception 2: Branch name contains 'revert'
                    if (sourceBranchLower.contains('revert')) {
                        echo "✅ EXCEPTION: Revert branch can merge anytime"
                        currentBuild.result = 'SUCCESS'
                        currentBuild.description = "Revert branch exception - approved"
                        env.EXCEPTION_GRANTED = 'true'
                        env.EXCEPTION_REASON = 'Revert branch exception'
                        env.SKIP_TIME_CHECK = 'true'
                        return
                    }
                    
                    echo "No exceptions apply. Proceeding to time window check..."
                    env.EXCEPTION_GRANTED = 'false'
                    env.SKIP_TIME_CHECK = 'false'
                }
            }
        }
        
        stage('Check Merge Time Window') {
            when {
                expression { 
                    env.SKIP_REMAINING != 'true' && env.SKIP_TIME_CHECK != 'true'
                }
            }
            steps {
                script {
                    // Get current time in IST
                    def istTime = sh(
                        script: 'TZ="Asia/Kolkata" date +"%Y-%m-%d %H:%M:%S %Z"',
                        returnStdout: true
                    ).trim()
                    
                    def currentHour = sh(
                        script: 'TZ="Asia/Kolkata" date +"%H"',
                        returnStdout: true
                    ).trim().toInteger()
                    
                    def currentMinute = sh(
                        script: 'TZ="Asia/Kolkata" date +"%M"',
                        returnStdout: true
                    ).trim()
                    
                    def currentDay = sh(
                        script: 'TZ="Asia/Kolkata" date +"%A"',
                        returnStdout: true
                    ).trim()
                    
                    echo "=========================================="
                    echo "TIME CHECK"
                    echo "=========================================="
                    echo "Current IST Time: ${istTime}"
                    echo "Current Hour: ${currentHour}"
                    echo "Current Minute: ${currentMinute}"
                    echo "Current Day: ${currentDay}"
                    echo "Allowed Window: 08:00 - 18:00 IST"
                    echo "=========================================="
                    
                    // Simulate off-hours if parameter is set
                    def hourToCheck = currentHour
                    if (params.SIMULATE_OFF_HOURS) {
                        hourToCheck = 19  // 7 PM - outside window
                        echo "⚠️  SIMULATION MODE: Pretending current hour is ${hourToCheck}:00"
                    }
                    
                    // Check if within allowed merge window (8 AM to 6 PM IST)
                    if (hourToCheck >= 8 && hourToCheck < 18) {
                        echo "✅ APPROVED: Current time (${hourToCheck}:00 IST) is within merge window (08:00 - 18:00 IST)"
                        currentBuild.result = 'SUCCESS'
                        currentBuild.description = "Merge approved - within time window (${hourToCheck}:00 IST)"
                        env.TIME_CHECK_PASSED = 'true'
                    } else {
                        def message = """
========================================
❌ MERGE BLOCKED - Outside Time Window
========================================

Current Time: ${istTime}
Current Hour: ${hourToCheck}:00 IST
Allowed Window: 08:00 AM - 06:00 PM IST
Status: BLOCKED ⛔

Your merge to 'pre_prod' is blocked because it's outside the allowed time window.

NEXT STEPS:
-----------
1. ⏰ Wait until 08:00 AM IST to merge
2. If urgent, consider these options:
   • Merge from 'master' or 'main' branch (allowed anytime)
   • Create a branch with 'revert' in the name (allowed anytime)
   • Contact someone with force-push access
3. Re-run this check during allowed hours (8 AM - 6 PM IST)

EXCEPTIONS (Allowed Anytime):
-----------------------------
✓ Merges from 'master' or 'main' branch
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
                        
                        currentBuild.result = 'FAILURE'
                        currentBuild.description = "❌ Blocked - outside merge window (${hourToCheck}:00 IST)"
                        env.TIME_CHECK_PASSED = 'false'
                        error("Merge blocked: Outside allowed time window (${hourToCheck}:00 IST)")
                    }
                }
            }
        }
        
        stage('Final Status') {
            when {
                expression { env.SKIP_REMAINING != 'true' }
            }
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
                    } else if (env.TIME_CHECK_PASSED == 'true') {
                        def successMessage = """
========================================
✅ PRE-PROD MERGE CHECK PASSED
========================================

Status: APPROVED ✓
Reason: Within allowed time window (08:00-18:00 IST)

Branch Details:
--------------
Source: ${SOURCE_BRANCH}
Target: ${TARGET_BRANCH}
PR: #${PR_ID}

You may proceed with the merge.
========================================
"""
                        echo successMessage
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "=========================================="
                echo "BUILD COMPLETE"
                echo "=========================================="
                echo "Build Result: ${currentBuild.result}"
                echo "Build Number: ${BUILD_NUMBER}"
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