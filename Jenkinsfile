pipeline {
    agent any
    environment {

        ARACHNI = ''
    }
    stages {
        stage('Checkout & Build & Sonar Analysis') {
            steps {
                git credentialsId: 'gitserverPrivateKey', url: 'git@github.com:urand0m/Sonarqube-Jenkins.git'
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn clean package -DskipTests -Dsurefire.useFile=false sonar:sonar'
                    sh 'mvn dependency:tree -DoutputType=dot --file="pom.xml"'
                    stash includes: 'target/JavaVulnerableLab.war', name: 'warfile'
                }
            }
        }

        stage("Sonar Quality Gate") {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('SpotBug Static Code Analysis') {
            steps {

                sh "mvn -batch-mode -V -U -e checkstyle:checkstyle findbugs:findbugs com.github.spotbugs:spotbugs-maven-plugin:3.1.7:spotbugs"
                //recordIssues enabledForFailure: true, tools: [mavenConsole(), java(), javaDoc()]
                //recordIssues enabledForFailure: true, tool: checkStyle()
                recordIssues enabledForFailure: true, tool: spotBugs()
                //recordIssues enabledForFailure: true, tool: cpd(pattern: '**/target/cpd.xml')
                //recordIssues enabledForFailure: true, tool: pmd(pattern: '**/target/pmd.xml')

            }
        }

        stage('Owasp Dependency Check') {
            steps {
                dir('WarExtract-SCA') {
                    unstash 'warfile'
                }
                //War file extracted at: $(pwd)/WarExtract-SCA/jspwiki-war/target/JSPWiki.war
                dependencyCheckAnalyzer datadir: '', hintsFile: '', includeCsvReports: true, includeHtmlReports: true, includeJsonReports: true, includeVulnReports: true, isAutoupdateDisabled: false, outdir: '', scanpath: '', skipOnScmChange: false, skipOnUpstreamChange: false, suppressionFile: '', zipExtensions: ''
                dependencyCheckPublisher canRunOnFailed: true, defaultEncoding: '', healthy: '', pattern: '', shouldDetectModules: true, unHealthy: ''
                dependencyTrackPublisher artifact: 'dependency-check-report.xml', artifactType: 'scanResult', projectId: '639cccde-564f-499b-ae14-45473d72eb30', synchronous: true
                archiveArtifacts artifacts: 'dependency-check-report.html,dependency-check-report.xml', onlyIfSuccessful: true
                //Add command to extract report from SCA scan
                stash includes: 'dependency-check-report.html,dependency-check-report.xml', name: 'DependencyCheckReport'


            }
        }
        stage('Snyk.io Dependency Check without Jenkins Plugin') {
            steps {
                script {
                    try {
                        sh 'snyk test --json | snyk-to-html -o snyk_report.html || true'
                        archiveArtifacts artifacts: 'snyk_report.html', onlyIfSuccessful: true
                        stash includes: 'snyk_report.html', name: 'SnykReport'
                        publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: false, reportDir: '', reportFiles: 'snyk_report.html', reportName: 'Snyk Report', reportTitles: 'Snyk Report'])
                    }
                    catch (err) {
                        echo err
                    }
                }
            }
        }
        stage('Deploy to Tomcat') {
            steps {

                sh 'scp target/JavaVulnerableLab.war urandom@dockerd:/home/urandom/tomcat/'
                echo '[*] Waiting for Tomcat to explode package'


            }
        }

        //stage('Wait Deployment To Complete'){
        //    steps{
        //        script {
        //        def response = sh(script: 'curl http://dockerd:8080/JavaVulnerableLab/', returnStdout: true)
        //
        //        String curltest = "200"
        //
        //        if (response == curltest) {
        //            echo ok
        //        }else{
        //            assert condition : "Seems Deployment went wrong..."
        //            }
        //        }
        //    }
        //}

        stage('Dynamic Analysis - Aachni') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    sh 'export today="$(date -d "today" +"%Y%m%d%H%M")"'
                    sh '/opt/arachni/bin/arachni --checks=*,-code_injection_php_input_wrapper,-ldap_injection,-no_sql*,-backup_files,-backup_directories,-captcha,-cvs_svn_users,-credit_card,-ssn,-localstart_asp,-webdav --timeout=00:20:00 --http-user-agent=\'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.103 Safari/537.36\' --report-save-path=scan_report_$today.afr http://dockerd:8080/JavaVulnerableLab/'
                    sh '/opt/arachni/bin/arachni_reporter scan_report_$today.afr --reporter=html:outfile=arachni_scan_report_$today.zip ; '

                }
                //script {

                //    ARACHNI=sh label: 'arachni', returnStdout: true, script: 'ls scan_report_* | awk -F _ \'{print "arachni_scan_report_"$3""}\''
                //}
                sh 'rm -rf arachni_scan'
                sh 'unzip arachni_scan_report_.zip -d arachni_scan'
                archiveArtifacts artifacts: 'arachni_scan_*.zip', onlyIfSuccessful: false
                stash includes: 'arachni_scan_*.zip', name: 'ArachniReport'
                publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: false, reportDir: 'arachni_report', reportFiles: 'index.html', reportName: 'ArachniReport', reportTitles: 'Arachni Report'])
            }
        }

        stage('Dynamic Analysis Security Testing- AppScan') {
            steps {
                node('Appscan') {
                    step([$class: 'AppScanStandardBuilder', additionalCommands: '/report_file C:\\Jenkins\\workspace\\JavaVulnerable\\javavuln.html /report_type Html', authScanPw: '', authScanRadio: true, authScanUser: '', includeURLS: '', installation: 'Appscan', pathRecordedLoginSequence: '', policyFile: 'C:\\Program Files (x86)\\IBM\\AppScan Standard\\Policies\\Application-Only.policy', reportName: 'javavuln.html', startingURL: 'http://dockerd:8080/JavaVulnerableLab/', verbose: true])
                    stash includes: 'javavuln.html', name: 'AppscanReport'
                    archiveArtifacts artifacts: 'javavuln.html', onlyIfSuccessful: false
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: false, reportDir: '', reportFiles: 'javavuln.html', reportName: 'Appscan Report', reportTitles: 'AppscanReport'])
                }
            }
        }

    }
}