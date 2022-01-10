def AGENT_LABEL = env.AGENT_LABEL ?: 'ubuntu'
def JDK_NAME = env.JDK_NAME ?: 'jdk_1.8_latest'
def MVN_NAME = env.MVN_NAME ?: 'maven_3_latest'

pipeline
    {
        agent
             {
                label AGENT_LABEL
            }
        parameters
            {
                //string(name: 'MULTI_MODULE', defaultValue: 'staging', description: '')
                choice(name: 'MULTI_MODULE', choices: ['staging', 'site'], description: 'Run as multi or single module ')
                // no default
                choice(name: 'FULCRUM_COMPONENT', choices: ['','cache', 'crypto', 'factory', 'intake', 'json', 'localization', 'parser', 'pool', 'quartz', 'security', 'site', 'testcontainer', 'upload', 'yaafi', 'yaafi-crypto'], description: 'Select fulcrum component')
                // default master
                choice(name: 'SUB_MODULE_HEAD', choices: ['master', 'main','trunk'], description: 'The master/main/trunk branch of the Fulcrum component')
                booleanParam(name: 'TEST_MODE', defaultValue: true, description: 'Run as Test Build or Deploy site for Component ')
            }
        tools
            {
                maven MVN_NAME
                jdk JDK_NAME
            }
        environment
            {
                DEPLOY_BRANCH = 'asf-site'
                STAGING_DIR = "target/${params.MULTI_MODULE}"
                // LANG = 'C.UTF-8'
                // -B, --batch-mode Run in non-interactive (batch) mode
                // -e, --error Produce execution error messages
                // -fae, --fail-at-end Only fail the build afterwards; allow all non-impacted builds to continue
                // -U,--update-snapshots                  Forces a check for missing
                // -V, --show-version Display version information WITHOUT stopping build
                // -ntp, --no-transfer-progress Do not display transfer progress when downloading or uploading
                // surefire.useFile  Option to generate a file test report or just output the test report to the console. Default true
                MAVEN_CLI_OPTS = "-B -V -U -e -fae -ntp -Dsurefire.useFile=false"
                MAVEN_GOALS = "${params.MULTI_MODULE == 'site' ? 'clean site' : 'clean site site:stage'}"
            }
        stages
            {
            stage('Prepare')
                {
                    steps
                        {
                            git 'https://gitbox.apache.org/repos/asf/turbine-fulcrum-build.git'
                            // not --update as clone ise done without --recurse-submodules
                            sh "git submodule update --init ${params.FULCRUM_COMPONENT}"
                            // branch will be detached head, need to checkout explicitely
                            dir("${params.FULCRUM_COMPONENT}")
                                {
                                    script
                                        {
                                            sh "pwd"
                                            sh "git branch"
                                            echo "${params.FULCRUM_COMPONENT}: Checking out ${params.SUB_MODULE_HEAD}"
                                            sh "git checkout ${params.SUB_MODULE_HEAD}"
                                            env.CURRENT_BRANCH = sh(script: "git status --branch --porcelain | grep '##' | cut -c 4-", returnStdout: true).trim()
                                            echo "CURRENT_BRANCH: ${env.CURRENT_BRANCH}"
                                            // Capture last commit hash for final commit message
                                            env.LAST_SHA = sh(script: 'git log -n 1 --pretty=format:%H', returnStdout: true).trim()
                                            echo "LAST_SHA: ${env.LAST_SHA}"
                                        }
                                }
                        }
                }
            stage('Build')
                {
                    when
                        {
                        expression { params.MULTI_MODULE == 'site' }
                    }
                    steps
                        {
                            dir("${params.FULCRUM_COMPONENT}")
                                {
                                    sh "pwd"
                                    // builds into target/site folder, this folder is expected to be preserved as it is used in next step
                                    sh "mvn $MAVEN_CLI_OPTS $MAVEN_GOALS"
                                    stash includes: "${STAGING_DIR}/**/*", name: "${params.FULCRUM_COMPONENT}-site"
                                }
                        }
                }
            stage('BuildWithStage')
                {
                    when
                        {
                            expression
                                {
                                params.MULTI_MODULE == 'staging'
                            }
                        }
                    steps
                        {
                            dir("${params.FULCRUM_COMPONENT}")
                                {
                                    sh "pwd"
                                    // builds into target/staging folder, this folder is expected to be preserved as it is used in next step
                                    sh "mvn $MAVEN_CLI_OPTS $MAVEN_GOALS"
                                    stash includes: "${STAGING_DIR}/**/*", name: "${params.FULCRUM_COMPONENT}-site"
                                }
                        }
                }
            stage('Deploy Site')
                {
                    when
                        {
                           anyOf
                            {
                                expression
                                    {
                                        env.CURRENT_BRANCH ==~ /(?i)^(master|trunk|main).*?/
                                    }
                            }
                    }
                    agent {
                        node {
                            label 'git-websites'
                        }
                    }
                    steps
                        {
                            sh "git submodule update --init ${params.FULCRUM_COMPONENT}"
                            dir("${params.FULCRUM_COMPONENT}")
                                {
                                    script
                                        {
                                            sh "pwd"
                                            echo "Deploying ${params.FULCRUM_COMPONENT} Site"
                                            unstash "${params.FULCRUM_COMPONENT}-site"
                                            // Checkout branch with current site content, target folder should be ignored!
                                            sh "git checkout ${DEPLOY_BRANCH}"
                                            // fetch only shallow
                                            // sh "git pull --depth=2 origin ${DEPLOY_BRANCH}"

                                            def exists = fileExists '.gitignore'
                                            if (exists)
                                            {
                                                echo "Fulcrum component ${params.FULCRUM_COMPONENT}: .gitignore exists in branch ${DEPLOY_BRANCH}."
                                            } else {
                                                echo "Fulcrum component ${params.FULCRUM_COMPONENT}: creating default .gitignore in branch ${DEPLOY_BRANCH}."
                                                sh "echo 'target/' > .gitignore"
                                                sh "git add .gitignore"
                                                sh "git commit -m \"Added .gitignore\""
                                            }
                                            // Remove the content (files) of the root folder and subdirectories and replace it with the content of the STAGING_DIR folder
                                            sh """
git ls-files | grep -v "^\\." | xargs  rm -f
"""
                                            sh "cp -rf ./${STAGING_DIR}/* ."
                                            // Commit the changes to the target branch BRANCH_NAME, groovy allows to omit env. prefix, available in multibranch pipeline.
                                            env.COMMIT_MESSAGE = "${params.FULCRUM_COMPONENT}: Updated site in ${DEPLOY_BRANCH} from ${env.CURRENT_BRANCH} (${env.LAST_SHA}) from ${params.MULTI_MODULE} from ${BUILD_URL}"
                                            echo "${env.COMMIT_MESSAGE} ....break...."
                                            
                                            if ( params.TEST_MODE == false) {
                                                echo 'committing as test mode false'
                                                //sh "git add -A"
                                                //sh "git commit -m "${env.COMMIT_MESSAGE}" | true"
                                                // Push the generated content for deployment
                                                //sh "git push -u origin ${DEPLOY_BRANCH}"
                                            } else {
                                                echo 'Skipping as test mode'
                                            }
                                            
                                        }
                                }
                        }
                }
        }
        post
            {
                always
                    {
                        deleteDir() /* clean up our workspace */
                    }
            }
    }
