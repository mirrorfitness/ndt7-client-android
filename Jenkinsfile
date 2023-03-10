def mainRegex = 'mirror_main'
def releaseBranchRegex = '^v-[0-9]+\\.[0-9]+'
def releaseTagRegex = '^r-[0-9]+\\.[0-9]+\\.[0-9]+$'

def statusToColorMap = [
        'SUCCESS': 'good',
        'FAILURE': 'danger',
        'UNSTABLE': 'warning',
        'ABORTED': 'warning'
]

def projectNameParts = JOB_NAME.tokenize('/') as String[]
def projectName = projectNameParts[0]
def revName = env.BRANCH_NAME
if (revName == null) {
    revName = env.CHANGED_ID
}

def currentTag = ""
try {
    currentTag = TAG_NAME
} catch (MissingPropertyException e) {
    // Unless a build was triggered by a tag being pushed, the TAG_NAME variable is not set
    // https://issues.jenkins.io/browse/JENKINS-58751?focusedCommentId=372585&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-372585
    echo "No TAG_NAME present - build not triggered because of tag"
}

slackSend(channel: "android-notifications",
        attachments: [
                [
                        title: projectName,
                        text: "Started run\nVariant `$revName`\nBuild: <$env.RUN_DISPLAY_URL|#$env.BUILD_NUMBER>\n"
                ]
        ])

pipeline {
    agent {
        node {
            label 'android_app'
        }
    }

    stages {
        stage('Build') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws_jenkins_userpass',
                        usernameVariable: 'AWS_ACCESS_KEY',
                        passwordVariable: 'AWS_SECRET_KEY')]) {
                    sh './gradlew clean assemble'
                }
            }

            post {
                success {
                    s3Upload consoleLogLevel: 'INFO',
                            dontWaitForConcurrentBuildCompletion: false,
                            dontSetBuildResultOnFailure: false,
                            entries: [
                                    [bucket: 'mirror-artifacts/jenkins',
                                     excludedFile: '',
                                     flatten: false,
                                     gzipFiles: false,
                                     keepForever: true,
                                     managedArtifacts: true,
                                     noUploadOnFailure: true,
                                     selectedRegion: 'us-east-1',
                                     showDirectlyInBrowser: false,
                                     sourceFile: 'libndt7/build/outputs/aar/**/*.aar',
                                     storageClass: 'STANDARD',
                                     uploadFromSlave: false,
                                     useServerSideEncryption: false]
                            ],
                            pluginFailureResultConstraint: 'FAILURE',
                            profileName: 'jenkins',
                            userMetadata: []

                }
            }
        }

        stage('Test') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws_jenkins_userpass',
                        usernameVariable: 'AWS_ACCESS_KEY',
                        passwordVariable: 'AWS_SECRET_KEY')]) {
                    sh './gradlew -i lint test'
                }
            }

            post {
                always {
                    s3Upload consoleLogLevel: 'INFO',
                            dontWaitForConcurrentBuildCompletion: false,
                            dontSetBuildResultOnFailure: false,
                            entries: [
                                    [bucket: 'mirror-artifacts/jenkins',
                                     excludedFile: '',
                                     flatten: false,
                                     gzipFiles: false,
                                     keepForever: true,
                                     managedArtifacts: true,
                                     noUploadOnFailure: false,
                                     selectedRegion: 'us-east-1',
                                     showDirectlyInBrowser: false,
                                     sourceFile: 'libndt7/build/reports/**',
                                     storageClass: 'STANDARD',
                                     uploadFromSlave: false,
                                     useServerSideEncryption: false]
                            ],
                            pluginFailureResultConstraint: 'FAILURE',
                            profileName: 'jenkins',
                            userMetadata: []
                }
            }
        }

        stage('Publish Release Build') {
            when {
                expression { currentTag ==~ releaseTagRegex }
            }

            steps {
                withCredentials([usernamePassword(credentialsId: 'aws_jenkins_userpass',
                        usernameVariable: 'AWS_ACCESS_KEY',
                        passwordVariable: 'AWS_SECRET_KEY')]) {
                    sh '''
                    ./gradlew publishAllPublicationsToMavenRepository
                    '''
                }
            }
        }
    }

    post {
        always {
            slackSend(channel: "android-notifications",
                    attachments: [
                            [
                                    color: statusToColorMap[currentBuild.currentResult],
                                    title: projectName,
                                    text: "Variant: `$revName`\nBuild:  <$env.RUN_DISPLAY_URL|#$env.BUILD_NUMBER>\nResult: ${currentBuild.currentResult}"
                            ]
                    ])
        }
    }
}
