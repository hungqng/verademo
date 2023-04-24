pipeline {
    agent any

    environment {
        VERACODE_APP_NAME = 'Verademo'      // App Name in the Veracode Platform
        VERACODE_SANDBOX_NAME = 'Jenkins'   // Sandbox Name
    }

    // this is optional on Linux, if jenkins does not have access to your locally installed docker
    //tools {
        // these match up with 'Manage Jenkins -> Global Tool Config'
        //'org.jenkinsci.plugins.docker.commons.tools.DockerTool' 'docker-latest' 
    //}

    options {
        // only keep the last x build logs and artifacts (for space saving)
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
    }

    stages{
        stage ('environment verify') {
            steps {
                script {
                    if (isUnix() == true) {
                        sh 'pwd'
                        sh 'ls -la'
                        sh 'echo $PATH'
                    }
                    else {
                        bat 'dir'
                        bat 'echo %PATH%'
                    }
                }
            }
        }

        stage ('build') {
            steps {
                withMaven(maven:'maven-3') {
                    script {
                        if(isUnix() == true) {
                            sh 'mvn clean package -f app/pom.xml'
                        }
                        else {
                            bat 'mvn clean package -f app/pom.xml'
                        }
                    }
                }
            }
        }

        stage ('Veracode Upload and Scan') {
            steps {
                echo 'Veracode Upload scanning'
                script {
                    if(isUnix() == true) {
                        env.HOST_OS = 'Unix'
                    }
                    else {
                        env.HOST_OS = 'Windows'
                    }
                }
                withCredentials([ usernamePassword ( 
                    credentialsId: 'veracode_login', usernameVariable: 'VERACODE_API_ID', passwordVariable: 'VERACODE_API_KEY') ]) {
                        veracode applicationName: "${VERACODE_APP_NAME}", criticality: 'VeryHigh', deleteIncompleteScan: 2, debug: true, scanName: "${BUILD_TAG}-${env.HOST_OS}", uploadIncludesPattern: 'app/target/verademo.war', vid: "${VERACODE_API_ID}", vkey: "${VERACODE_API_KEY}"
                    }
            }
        }

        stage ('Veracode Pipeline Scanner') {
            steps {
                echo 'Veracode Pipeline scanning'
                withCredentials([ usernamePassword ( 
                    credentialsId: 'veracode_login', usernameVariable: 'VERACODE_API_ID', passwordVariable: 'VERACODE_API_KEY') ]) {
                        script {

                            // this try-catch block will show the flaws in the Jenkins log, and yet not
                            // fail the build due to any flaws reported in the pipeline scan
                            // alternately, you could add --fail_on_severity '', but that would not show the
                            // flaws in the Jenkins log

                            // issue_details true: add flaw details to the results.json file
                            try {
                                if(isUnix() == true) {
                                    sh """
                                        curl -sO https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip
                                        unzip pipeline-scan-LATEST.zip pipeline-scan.jar
                                        java -jar pipeline-scan.jar \
                                            --veracode_api_id '${VERACODE_API_ID}' \
                                            --veracode_api_key '${VERACODE_API_KEY}' \
                                            --file app/target/verademo.war \
                                            --issue_details true \
                                            --json_output true \
                                            --json_output_file results.json
                                        """
                                }
                                else {
                                    powershell """
                                            curl  https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o pipeline-scan.zip
                                            Expand-Archive -Path pipeline-scan.zip -DestinationPath veracode_scanner
                                            java -jar veracode_scanner\\pipeline-scan.jar \
                                                --veracode_api_id '${VERACODE_API_ID}' \
                                                --veracode_api_key '${VERACODE_API_KEY}' \
                                                --file app/target/verademo.war \
                                                --issue_details true \
                                                --json_output true \
                                                --json_output_file results.json
                                            """
                                }
                            } catch (err) {
                                echo 'Pipeline err: ' + err
                            }
                        }    
                    } 
            }
        }
        stage ('Veracode Software Compositition Analysis') {
            steps {
                echo 'Veracode SCA scanning'
                withCredentials([ string(credentialsId: 'SCA_Token', variable: 'SRCCLR_API_TOKEN')]) {
                    withMaven(maven:'maven-3') {
                        script {
                            if(isUnix() == true) {
                                sh "curl -sSL https://download.sourceclear.com/ci.sh | sh -s -- scan ./app"

                                // debug, no upload
                                //sh "curl -sSL https://download.sourceclear.com/ci.sh | DEBUG=1 sh -s -- scan --no-upload"
                            }
                            else {
                                powershell '''
                                            Set-ExecutionPolicy AllSigned -Scope Process -Force
                                            $ProgressPreference = "silentlyContinue"
                                            iex ((New-Object System.Net.WebClient).DownloadString('https://download.srcclr.com/ci.ps1'))
                                            srcclr scan ./app
                                            '''
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: "results.json", fingerprint: true, allowEmptyArchive: true
            deleteDir()
        }
    }
}
