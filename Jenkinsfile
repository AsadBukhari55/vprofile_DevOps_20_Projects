pipeline {
    
	agent any
	
	tools {
        maven "MAVEN3"
    }
	
    environment {
        NEXUS_USER = 'admin'
        NEXUS_PASS ='1010'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO ='vpro-maven-central-24'
        NEXUS_LOGIN = 'nexuslogin'
        NEXUSIP ='10.0.12.178'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group-24'
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "10.0.12.178:8081"
        NEXUS_REPOSITORY = "vprofile-release24"
	    NEXUS_REPOGRP_ID    = "vprofile-grp-repo"
        NEXUS_CREDENTIAL_ID = "nexuslogin"
        ARTVERSION = "${env.BUILD_ID}"
        SONARSERVER = "sonarserver"
        SONERSCANNER = "sonarscanner"
        /*NEXUSPASS = credentials('nexuspass') */
    }
	
    stages{
        
        stage('BUILD'){
            steps {
               /* sh 'mvn clean install -DskipTests' */
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

	stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

	stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
		
        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
          
		  environment {
             scannerHome = tool "${SONERSCANNER}"
          }

          steps {
            withSonarQubeEnv("${SONARSCANNER}") {
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
            }

            timeout(time: 10, unit: 'MINUTES') {
               waitForQualityGate abortPipeline: true
            }
          }
        }

        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version} ARTVERSION";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: NEXUS_REPOGRP_ID,
                            version: ARTVERSION,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );
                    } 
		    else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }

    stage('Ansible Deploy to staging'){
        steps {
            ansiblePlaybook([
            inventory   : 'ansible/stage.inventory',
            playbook    : 'ansible/site.yml',
            installation: 'ansible',
            colorized   : true,
            credentialsId: 'applogin',
            disableHostKeyChecking: true,
            extraVars  : [
                USER: "admin",
                PASS: "${NEXUSPASS}",
                nexusip: "",
                reponame: "vprofile-release24",
                groupid: "QA",
                time: "${env.BUILD_TIMESTAMP}",
                build: "${env.BUILD_ID}",
                artifactid: "vproapp",
                vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMp}.war"
            ]
            ])
        }
    }
    
    }
}
