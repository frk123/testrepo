pipeline {
    agent any
    
    options {
        // Keep the build fast
        timeout(time: 2, unit: 'MINUTES')
    }
    
    stages {
        stage('Check Merge Time Restrictions') {
            steps {
                script {
                    // Get branch name
                    def branchName = env.BRANCH_NAME ?: env.GIT_BRANCH?.replaceFirst('origin/', '')
                    echo "Checking branch: ${branchName}"
                    
                    // Check if this is an exception branch (master or contains 'revert')
                    def isExceptionBranch = branchName == 'master' || branchName.toLowerCase().contains('revert')
                    
                    if (isExceptionBranch) {
                        echo "🟢 ALLOWED: Branch '${branchName}' is exempt from time restrictions"
                        currentBuild.description = "✅ MERGE ALLOWED: Exception branch detected"
                        return
                    }
                    
                    // Get current time in IST
                    def currentTimeIST = sh(script: '''
                        TZ="Asia/Kolkata" date +"%H%M"
                    ''', returnStdout: true).trim() as Integer
                    
                    def startTime = 800  // 8:00 AM
                    def endTime = 1835   // 6:00 PM
                    
                    echo "Current time (IST): ${currentTimeIST}"
                    
                    if (currentTimeIST >= startTime && currentTimeIST < endTime) {
                        echo "🟢 ALLOWED: Current time is within the allowed merge window (8:00 AM - 6:00 PM IST)"
                        currentBuild.description = "✅ MERGE ALLOWED: Within permitted time window"
                    } else {
                        echo "❌ BLOCKED: Current time is outside the allowed merge window"
                        echo "Allowed merge window is 8:00 AM - 6:00 PM IST"
                        echo "You can:"
                        echo "1. Wait until the next merge window opens"
                        echo "2. If urgent, rename your branch to include 'revert'"
                        echo "3. Contact an administrator with force push access for assistance"
                        
                        currentBuild.description = "❌ MERGE BLOCKED: Outside permitted time window"

                        // Create PR comment if applicable
                        if (env.CHANGE_ID) {
                            def comment = """## ❌ Merge to pre_prod branch blocked
**Reason**: Attempted merge outside permitted time window (8:00 AM - 6:00 PM IST)

**Current time (IST)**: ${new Date().format('HH:mm', TimeZone.getTimeZone('Asia/Kolkata'))}

### Options:
1. Wait until the merge window opens at 8:00 AM IST tomorrow
2. For urgent changes, create a new branch with 'revert' in the name
3. Contact an administrator with force push access for assistance

*This is an automated message from the branch protection system.*
"""
                            // Try to comment using GitHub CLI
                            try {
                                sh """
                                    if command -v gh >/dev/null 2>&1; then
                                        gh pr comment ${env.CHANGE_ID} --body "${comment.replace('"', '\\"')}"
                                    else
                                        echo "⚠️ GitHub CLI not installed. Skipping PR comment."
                                    fi
                                """
                            } catch (err) {
                                echo "⚠️ Failed to post PR comment: ${err}"
                            }
                        }
                        
                        error "Merge rejected: Outside of permitted merge window (8:00 AM - 6:00 PM IST)"
                    }
                }
            }
        }
    }
    
    post {
        success {
            script {
                if (env.CHANGE_ID) {
                    def message = """## ✅ pre_prod Branch Protection Check Passed
                    
This pull request has been approved for merging to the pre_prod branch.

*This is an automated message from the branch protection system.*
"""
                    try {
                        sh """
                            if command -v gh >/dev/null 2>&1; then
                                gh pr comment ${env.CHANGE_ID} --body "${message.replace('"', '\\"')}"
                            else
                                echo "⚠️ GitHub CLI not installed. Skipping PR comment."
                            fi
                        """
                    } catch (err) {
                        echo "⚠️ Failed to post success comment: ${err}"
                    }
                }
            }
        }
        failure {
            echo "Check failed. See above messages for details."
        }
    }
}
