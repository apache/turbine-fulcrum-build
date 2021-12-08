#!groovy

/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

// https://ci-builds.apache.org/pipeline-syntax/globals
// https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/
// https://www.jenkins.io/doc/pipeline/examples/#maven-and-jdk-specific-version
// https://cwiki.apache.org/confluence/display/INFRA/Jenkins+node+labels
// https://cwiki.apache.org/confluence/display/INFRA/ASF+Cloudbees+Operations+Center
 
// started with jena-site and gora

// git-websites:
// "Nodes that are reserved for ANY project that wants to build their website docs and
// publish directly live (requires asf-site and pypubsub"
// no git-websites label her using ubuntu

// Jenkins build server used: https://builds.apache.org/ / ci-builds.apache.org

// Fulcurm-Build has submodules, which are expcetd for this Jenkinsfile NOT to be fetched if cloning!
// This is the default (do NOT provide git clone --recurse-submodules flag) and this is because
// only the current submodule is initialized downstrema with git submodule update --init

def AGENT_LABEL = env.AGENT_LABEL ?: 'ubuntu'

def JDK_NAME = env.JDK_NAME ?: 'jdk_1.8_latest'
def MVN_NAME = env.MVN_NAME ?: 'maven_3_latest'

pipeline
    {
        agent
            {
                node {
                    label AGENT_LABEL 
                }
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
                STAGING_DIR = "target/${params.MULTI_MODULE}/"
                FULCRUM_COMPONENT = "${params.FULCRUM_COMPONENT}"
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
                            // not --update as clone ise done without --recurse-submodules
                            sh "git submodule update --init $FULCRUM_COMPONENT"
                            // branch will be detached head, need to checkout explicitely
                            dir("$FULCRUM_COMPONENT")
                                {
                                    script
                                        {
                                            sh "pwd"
                                            sh "git branch"
                                            echo "$FULCRUM_COMPONENT: Checking out ${params.SUB_MODULE_HEAD}"
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
                            dir("$FULCRUM_COMPONENT")
                                {
                                    sh "pwd"
                                    // builds into target/site folder, this folder is expected to be preserved as it is used in next step
                                    sh "mvn $MAVEN_CLI_OPTS $MAVEN_GOALS"
                                }
                        }
                }
            stage('Build2')
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
                            dir("$FULCRUM_COMPONENT")
                                {
                                    sh "pwd"
                                    // builds into target/staging folder, this folder is expected to be preserved as it is used in next step
                                    sh "mvn $MAVEN_CLI_OPTS $MAVEN_GOALS"
                                }
                        }
                }
            stage('Deploy')
                {
                    when
                        {
                        allOf {
                            not {
                                expression 
                                    { 
                                        params.TEST_MODE 
                                    }
                            }
                           anyOf
                            {
                                expression
                                    {
                                        env.CURRENT_BRANCH ==~ /(?i)^(master|trunk|main).*?/
                                    }
                            }
                        }
                    }
                    steps
                        {
                            dir("$FULCRUM_COMPONENT")
                                {
                                    script
                                        {
                                            sh "pwd"
                                            // Checkout branch with current site content, target folder should be ignored!
                                            sh "git checkout ${DEPLOY_BRANCH}"
                                            // fetch only shallow
                                            sh "git pull --depth=2 origin ${DEPLOY_BRANCH}"

                                            def exists = fileExists '.gitignore'
                                            if (exists)
                                            {
                                                echo "Fulcrum component $FULCRUM_COMPONENT: .gitignore exists in branch ${DEPLOY_BRANCH}."
                                            } else {
                                                echo "Fulcrum component $FULCRUM_COMPONENT: creating default .gitignore in branch ${DEPLOY_BRANCH}."
                                                sh "echo 'target/' > .gitignore"
                                                sh "git add .gitignore"
                                                sh "git commit -m \"Added .gitignore\""
                                            }
                                            // Remove the content (files) of the root folder and subdirectories and replace it with the content of the STAGING_DIR folder
                                            sh """
git ls-files | grep -v "^\\." | xargs  rm -f
"""
                                            sh "cp -rf ./${STAGING_DIR}* ."
                                            // Commit the changes to the target branch BRANCH_NAME, groovy allows to omit env. prefix, available in multibranch pipeline.
                                            env.COMMIT_MESSAGE = "$FULCRUM_COMPONENT: Updated site in ${DEPLOY_BRANCH} from ${env.CURRENT_BRANCH} (${env.LAST_SHA}) from ${params.MULTI_MODULE} from ${BUILD_URL}"
                                            sh "git add -A"
                                            sh "git commit -m "${env.COMMIT_MESSAGE}" | true"
                                            // Push the generated content for deployment
                                            sh "git push -u origin ${DEPLOY_BRANCH}"
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
