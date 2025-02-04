#!groovy

/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
 
def AGENT_LABEL = env.AGENT_LABEL ?: 'ubuntu'
def JDK_NAME = env.JDK_NAME ?: 'jdk_1.8_latest'
def MVN_NAME = env.MVN_NAME ?: 'maven_3_latest'

// =================================================================
// https://cwiki.apache.org/confluence/display/INFRA/Jenkins
// https://cwiki.apache.org/confluence/display/INFRA/Multibranch+Pipeline+recipes
// =================================================================

// general pipeline documentation: https://jenkins.io/doc/book/pipeline/syntax/
pipeline {

    // https://jenkins.io/doc/book/pipeline/syntax/#agent
    agent {
        node {
            label AGENT_LABEL
        }
    }

    tools {
        maven MVN_NAME
        jdk JDK_NAME
    }

    // https://jenkins.io/doc/book/pipeline/syntax/#options
    options {
        // support ANSI colors in stdout/stderr
        ansiColor 'xterm'
        // only keep the latest 10 builds
        buildDiscarder logRotator(numToKeepStr: '10')
        // cancel build if not complete within two hours of scheduling
        timeout time: 3, unit: 'HOURS'
        disableConcurrentBuilds()
    }
    
    environment {
        LANG = 'C.UTF-8'
        // --batch-mode Run in non-interactive (batch) mode
        // --error Produce execution error messages
        // --fail-at-end Only fail the build afterwards; allow all non-impacted builds to continue
        // --show-version Display version information WITHOUT stopping build
        // --no-transfer-progress Do not display transfer progress when downloading or uploading
        MAVEN_CLI_OPTS = "--batch-mode --errors --fail-at-end --show-version --no-transfer-progress"
        // https://cwiki.apache.org/confluence/display/INFRA/SonarQube+Analysis
        // https://sonarcloud.io/documentation/analysis/scan/sonarscanner-for-maven/
        SONARCLOUD_PARAMS = "-Dsonar.host.url=https://sonarcloud.io -Dsonar.organization=apache -Dsonar.projectKey=apache_gora"
    }

    stages {
        stage('Build') {
            steps {
                sh "mvn $MAVEN_CLI_OPTS -DskipTests clean install"
            }

            post {
                success {
                    archiveArtifacts '**/target/*.jar'
                }
            }
        }
        
        stage('Test') {
            steps {
                warnError(message: "Failure during tests execution") {
                    sh "mvn $MAVEN_CLI_OPTS verify"
                }
            }

            post {
                always {
                    junit testResults: '**/target/surefire-reports/TEST-*.xml', testDataPublishers: [[$class: 'StabilityTestDataPublisher']]
                }
            }
        }

        stage('Deploy snapshots') {
            when {
                branch 'master'
            }
            steps {
                // Use release profile defined in project pom.xml
                sh "mvn $MAVEN_CLI_OPTS --activate-profiles release -DskipTests deploy"
            }
        }

        stage('Code Quality') {
            // https://sonarcloud.io/documentation/appendices/move-analysis-java-11/
            // "continue building in Java 8 but will use Java 11 to scan the code"
            // https://cwiki.apache.org/confluence/display/INFRA/JDK+Installation+Matrix
            tools {
                jdk 'jdk_11_latest'
            }
            steps {
                echo 'Checking Code Quality on SonarCloud'
                // 'drazzib-sonarcloud-token' needs to be defined for this job and contains the user token
                withCredentials([string(credentialsId: 'drazzib-sonarcloud-token', variable: 'SONAR_TOKEN')]) {
                    sh "mvn sonar:sonar ${SONARCLOUD_PARAMS} -Dsonar.branch.name=${BRANCH_NAME}"
                }
            }
        }
    }
}
