pipeline {
    agent {
        label 'mettleci:datastage11.5'
    }
    parameters {
        string(name: 'domainName', defaultValue: 'test1-svcs.datamigrators.io:59445', description: 'DataStage Service Tier')
        string(name: 'serverName', defaultValue: 'TEST1-ENGN.DATAMIGRATORS.IO', description: 'DataStage Engine Tier')
        string(name: 'projectName', defaultValue: 'jenkins', description: 'Logical Project Name')
        string(name: 'environmentId', defaultValue: 'ci', description: 'Environment Identifer')
    }
    environment {
        DATASTAGE_PROJECT = "${params.projectName}_${params.environmentId}"
        ENVIRONMENT_ID = "${params.environmentId}"
    }
    stages {
        stage("Deploy") {
            agent { label "mettleci:datastage11.5" }

            steps {
                withCredentials([usernamePassword(credentialsId: 'mci-user', passwordVariable: 'datastagePassword', usernameVariable: 'datastageUsername')]) {

                    bat label: 'Create DataStage Project', script: "${env.METTLE_SHELL} datastage create-project -domain ${params.domainName} -server ${params.serverName} -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword}"
                    
                    bat label: 'Substitute parameteres in DataStage config', script: "${env.METTLE_SHELL} properties config -baseDir datastage -filePattern \"*.sh\" -filePattern \"DSParams\" -filePattern \"Parameter Sets/*/*\" -properties var.${params.environmentId} -outDir config"
                    bat label: 'Transfer DataStage config and filesystem assets', script: "${env.METTLE_SHELL} remote upload -host ${params.serverName} -username ${datastageUsername} -password ${datastagePassword} -transferPattern \"filesystem/**/*,config/*\" -destination \"${env.BUILD_TAG}\""
                    bat label: 'Deploy DataStage config and file system assets', script: "${env.METTLE_SHELL} remote execute -host ${params.serverName} -username ${datastageUsername} -password ${datastagePassword} -script \"config\\deploy.sh\""
                    
                    bat label: 'Deploy DataStage project', script: "${env.METTLE_SHELL} datastage deploy -domain ${params.domainName} -server ${params.serverName} -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword} -assets datastage -parameter-sets \"config\\Parameter Sets\" -threads 8 -project-cache \"C:\\dm\\mci\\cache\\${params.serverName}\\${env.DATASTAGE_PROJECT}\""

                }
            }
            post {
                always {
                    junit testResults: 'log/**/mettleci_compilation.xml', allowEmptyResults: true
                    withCredentials([usernamePassword(credentialsId: 'mci-user', passwordVariable: 'datastagePassword', usernameVariable: 'datastageUsername')]) {
                        bat label: 'Cleanup temporary files', script: "${env.METTLE_SHELL} remote execute -host ${params.serverName} -username ${datastageUsername} -password ${datastagePassword} -script \"config\\cleanup.sh\""
                    }
                    deleteDir()
                }
            }
        }
        stage("Test") {
            parallel {
                stage('Static Analysis') {
                    agent { label "mettleci:datastage11.5" }
                    
                    steps {
                        bat label: 'Perform static analysis', script: "${env.METTLE_SHELL} compliance test -assets datastage -report \"compliance_report.xml\" -junit -rules compliance -project-cache \"C:\\dm\\mci\\cache\\${params.serverName}\\${env.DATASTAGE_PROJECT}\""
                    }
                    post {
                        always {
                            junit testResults: 'compliance_report.xml', allowEmptyResults: true
                            deleteDir()
                        }
                    }
                }
                stage('Unit Tests') {
                    agent { label "mettleci:datastage11.5" }
                    
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'mci-user', passwordVariable: 'datastagePassword', usernameVariable: 'datastageUsername')]) {
                            bat label: 'Upload unit test specs', script: "${env.METTLE_SHELL} remote upload -host ${params.serverName} -username ${datastageUsername} -password ${datastagePassword} -source \"unittest\" -transferPattern \"**/*\" -destination \"/opt/dm/mci/specs/${env.DATASTAGE_PROJECT}\""
                            bat label: 'Execute unit tests for changed DataStage jobs', script: "${env.METTLE_SHELL} unittest test -domain ${params.domainName} -server ${params.serverName} -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword} -specs unittest -reports test-reports -project-cache \"C:\\dm\\mci\\cache\\${params.serverName}\\${env.DATASTAGE_PROJECT}\""
                            bat label: 'Retrieve unit test results', script: "${env.METTLE_SHELL} remote download -host ${params.serverName} -username ${datastageUsername} -password ${datastagePassword} -source \"/opt/dm/mci/reports\" -transferPattern \"${env.DATASTAGE_PROJECT}/**/*.xml\" -destination \"test-reports\""
                        }
                    }
                    post {
                        always {
                            junit testResults: 'test-reports/**/*.xml', allowEmptyResults: true
                            deleteDir()
                        }
                    }
                }
            }
        }
        stage("Promote") {
            parallel {
                stage('1.Testing') {
                    agent { label "mettleci:datastage11.5" }
                    
                    input {
                        message "Should we promote to Testing?"
                        ok "Yes"
                        parameters {
                            string(name: 'domainName', defaultValue: 'test1-svcs.datamigrators.io:59445', description: 'DataStage Service Tier')
                            string(name: 'serverName', defaultValue: 'TEST1-ENGN.DATAMIGRATORS.IO', description: 'DataStage Engine Tier')
                            string(name: 'projectName', defaultValue: 'jenkins', description: 'Logical Project Name')
                            string(name: 'environmentId', defaultValue: 'test', description: 'Environment Identifer')
                        }
                    }
                    environment {
                        DATASTAGE_PROJECT = "${projectName}_${environmentId}"
                        ENVIRONMENT_ID = "${environmentId}"
                    }
                    steps {

                        withCredentials([usernamePassword(credentialsId: 'mci-user', passwordVariable: 'datastagePassword', usernameVariable: 'datastageUsername')]) {

                            bat label: 'Create DataStage Project', script: "${env.METTLE_SHELL} datastage create-project -domain ${domainName} -server ${serverName} -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword}"

                            bat label: 'Substitute parameteres in DataStage config', script: "${env.METTLE_SHELL} properties config -baseDir datastage -filePattern \"*.sh\" -filePattern \"DSParams\" -filePattern \"Parameter Sets/*/*\" -properties var.${environmentId} -outDir config"
                            bat label: 'Transfer DataStage config and filesystem assets', script: "${env.METTLE_SHELL} remote upload -host ${serverName} -username ${datastageUsername} -password ${datastagePassword} -transferPattern \"filesystem/**/*,config/*\" -destination \"${env.BUILD_TAG}\""
                            bat label: 'Deploy DataStage config and file system assets', script: "${env.METTLE_SHELL} remote execute -host ${serverName} -username ${datastageUsername} -password ${datastagePassword} -script \"config\\deploy.sh\""

                            bat label: 'Deploy DataStage project', script: "${env.METTLE_SHELL} datastage deploy -domain ${domainName} -server ${serverName} -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword} -assets datastage -parameter-sets \"config\\Parameter Sets\" -threads 8 -project-cache \"C:\\dm\\mci\\cache\\${params.serverName}\\${env.DATASTAGE_PROJECT}\""

                        }
                    }
                    post {
                        always {
                            withCredentials([usernamePassword(credentialsId: 'mci-user', passwordVariable: 'datastagePassword', usernameVariable: 'datastageUsername')]) {
                                bat label: 'Cleanup temporary files', script: "${env.METTLE_SHELL} remote execute -host ${serverName} -username ${datastageUsername} -password ${datastagePassword} -script \"config\\cleanup.sh\""
                            }
                            deleteDir()
                        }
                    }
                }
                stage('2.Quality Assurance') {
                    agent { label "mettleci:datastage11.5" }
                    
                    input {
                        message "Should we promote to Quality Assurance?"
                        ok "Yes"
                        parameters {
                            string(name: 'domainName', defaultValue: 'test1-svcs.datamigrators.io:59445', description: 'DataStage Service Tier')
                            string(name: 'serverName', defaultValue: 'TEST1-ENGN.DATAMIGRATORS.IO', description: 'DataStage Engine Tier')
                            string(name: 'projectName', defaultValue: 'jenkins', description: 'Logical Project Name')
                            string(name: 'environmentId', defaultValue: 'qa', description: 'Environment Identifer')
                        }
                    }
                    environment {
                        DATASTAGE_PROJECT = "${projectName}_${environmentId}"
                        ENVIRONMENT_ID = "${environmentId}"
                    }
                    steps {

                        withCredentials([usernamePassword(credentialsId: 'mci-user', passwordVariable: 'datastagePassword', usernameVariable: 'datastageUsername')]) {

                            bat label: 'Create DataStage Project', script: "${env.METTLE_SHELL} datastage create-project -domain ${domainName} -server ${serverName} -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword}"

                            bat label: 'Substitute parameteres in DataStage config', script: "${env.METTLE_SHELL} properties config -baseDir datastage -filePattern \"*.sh\" -filePattern \"DSParams\" -filePattern \"Parameter Sets/*/*\" -properties var.${environmentId} -outDir config"
                            bat label: 'Transfer DataStage config and filesystem assets', script: "${env.METTLE_SHELL} remote upload -host ${serverName} -username ${datastageUsername} -password ${datastagePassword} -transferPattern \"filesystem/**/*,config/*\" -destination \"${env.BUILD_TAG}\""
                            bat label: 'Deploy DataStage config and file system assets', script: "${env.METTLE_SHELL} remote execute -host ${serverName} -username ${datastageUsername} -password ${datastagePassword} -script \"config\\deploy.sh\""

                            bat label: 'Deploy DataStage project', script: "${env.METTLE_SHELL} datastage deploy -domain ${domainName} -server ${serverName} -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword} -assets datastage -parameter-sets \"config\\Parameter Sets\" -threads 8 -project-cache \"C:\\dm\\mci\\cache\\${params.serverName}\\${env.DATASTAGE_PROJECT}\""

                        }
                    }
                    post {
                        always {
                            withCredentials([usernamePassword(credentialsId: 'mci-user', passwordVariable: 'datastagePassword', usernameVariable: 'datastageUsername')]) {
                                bat label: 'Cleanup temporary files', script: "${env.METTLE_SHELL} remote execute -host ${serverName} -username ${datastageUsername} -password ${datastagePassword} -script \"config\\cleanup.sh\""
                            }
                            deleteDir()
                        }
                    }
               }
               stage('3.Production') {
                    agent { label "mettleci:datastage11.5" }
                    
                    input {
                        message "Should we promote to Production?"
                        ok "Yes"
                        parameters {
                            string(name: 'domainName', defaultValue: 'test1-svcs.datamigrators.io:59445', description: 'DataStage Service Tier')
                            string(name: 'serverName', defaultValue: 'TEST1-ENGN.DATAMIGRATORS.IO', description: 'DataStage Engine Tier')
                            string(name: 'projectName', defaultValue: 'jenkins', description: 'Logical Project Name')
                            string(name: 'environmentId', defaultValue: 'prod', description: 'Environment Identifer')
                        }
                    }
                    environment {
                        DATASTAGE_PROJECT = "${projectName}_${environmentId}"
                        ENVIRONMENT_ID = "${environmentId}"
                    }
                    steps {

                        withCredentials([usernamePassword(credentialsId: 'mci-user', passwordVariable: 'datastagePassword', usernameVariable: 'datastageUsername')]) {

                            bat label: 'Create DataStage Project', script: "${env.METTLE_SHELL} datastage create-project -domain ${domainName} -server ${serverName} -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword}"

                            bat label: 'Substitute parameteres in DataStage config', script: "${env.METTLE_SHELL} properties config -baseDir datastage -filePattern \"*.sh\" -filePattern \"DSParams\" -filePattern \"Parameter Sets/*/*\" -properties var.${environmentId} -outDir config"
                            bat label: 'Transfer DataStage config and filesystem assets', script: "${env.METTLE_SHELL} remote upload -host ${serverName} -username ${datastageUsername} -password ${datastagePassword} -transferPattern \"filesystem/**/*,config/*\" -destination \"${env.BUILD_TAG}\""
                            bat label: 'Deploy DataStage config and file system assets', script: "${env.METTLE_SHELL} remote execute -host ${serverName} -username ${datastageUsername} -password ${datastagePassword} -script \"config\\deploy.sh\""

                            bat label: 'Deploy DataStage project', script: "${env.METTLE_SHELL} datastage deploy -domain ${domainName} -server ${serverName} -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword} -assets datastage -parameter-sets \"config\\Parameter Sets\" -threads 8 -project-cache \"C:\\dm\\mci\\cache\\${params.serverName}\\${env.DATASTAGE_PROJECT}\""

                        }
                    }
                    post {
                        always {
                            withCredentials([usernamePassword(credentialsId: 'mci-user', passwordVariable: 'datastagePassword', usernameVariable: 'datastageUsername')]) {
                                bat label: 'Cleanup temporary files', script: "${env.METTLE_SHELL} remote execute -host ${serverName} -username ${datastageUsername} -password ${datastagePassword} -script \"config\\cleanup.sh\""
                            }
                            deleteDir()
                        }
                    }
                }
            }
        }
    }
}
