pipeline {
	options {
		disableConcurrentBuilds()
	}
    agent { 
        kubernetes {
            inheritFrom 'default-slave' 
        }
    }
    stages {
        stage('Initialize Pipeline') {
            steps {
				container ('default-slave' ){
					script {
						common.setEnvVars()
                        
                        def props = readProperties file: 'JenkinsParams'
                        env.REGISTRY_URL = props['REGISTRY_URL']
                        env.REGISTRY_NAME = props['REGISTRY_NAME']

                        def ver_file = readJSON file: 'package.json'
                        env.VERSION = ver_file['version']

                        if ( env.BRANCH_NAME && env.BRANCH_NAME ==~ /(development|release|master)/) { 
                            env.TAG_NAME = env.BRANCH_NAME
                        }
                        if ( env.CHANGE_TARGET && env.CHANGE_TARGET ==~ /(development|release|master)/) { 
                            env.TAG_NAME = env.CHANGE_TARGET
						}
                    }
                }
            }
        }

        stage('Download Dependencies') {
            when {
                anyOf {
                    expression { env.BRANCH_NAME != null && env.BRANCH_NAME ==~ /(feature|bugfix|development).*/ }
                    expression { env.CHANGE_TARGET != null && CHANGE_TARGET ==~ /(development|release)/ }
                }
            }
            steps {
                container ('default-slave' ){
                    script {
                        // Init Stage Slack Message
                        slackNotif.initStageMessage()
                        sh 'npm install'
                        //Update Stage Notificacion
                    }
                }
            }
        }


        stage('Build app') {
            when {
                anyOf {
                    expression { env.BRANCH_NAME != null && env.BRANCH_NAME ==~ /(feature|bugfix|development).*/ }
                    expression { env.CHANGE_TARGET != null && CHANGE_TARGET ==~ /(development|release)/ }
                }
            }
            steps {
				container ('default-slave' ){
					script {
                        // Init Stage Slack Message
                        slackNotif.initStageMessage()
                        def feature_pattern = '(feature|bugfix|development).*'
                        def pr_pattern = '(PR-.*)'

                        switch (env.BRANCH_NAME) {
                            case ~/$feature_pattern/:
                                env.BUILD = 'dev'
                                env.TAG_NAME =  'dev'
                                break
                            case 'release':
                                env.BUILD = 'release'
                                env.TAG_NAME =  'release'
                                break
                            case 'master':
                                env.BUILD = 'prod'
                                break
                            case ~/$pr_pattern/:
                                if (env.CHANGE_TARGET == 'development') { 
                                    env.BUILD = 'dev'
                                    env.TAG_NAME =  'dev'
                                }
                                if (env.CHANGE_TARGET == 'release') {
                                    env.BUILD = 'release'
                                    env.TAG_NAME =  'release'
                                }
                                if (env.CHANGE_TARGET == 'master') {
                                    env.BUILD = 'prod'
                                }
                                break
                        }

                        sh "npm run build"
						//Update Stage Notificacion
					}
                }
            }
        }
        

		stage('Build Docker Image') {
            when {
				anyOf {
                    expression { env.BRANCH_NAME != null && env.BRANCH_NAME ==~ /(development)/ }
                    expression { env.CHANGE_TARGET != null && CHANGE_TARGET ==~ /(release)/ }
				}
            }
            steps{
				container ('default-slave') {
					script {
                        // Init Stage Slack Message
                        slackNotif.initStageMessage()
                        docker.withRegistry( "https://${env.REGISTRY_URL}", 'ecr:us-east-1:aws-credential-registry') {
                            sh "DOCKER_BUILDKIT=1  docker build --network=host -t ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_B${env.BUILD_NUMBER}_C${env.SHORT_COMMIT} ."
                        }
                        // Scanning Docker Image with Qualys
                        sh "sudo chmod a+rw /var/run/docker.sock"
                        env.IMAGE_ID = sh returnStdout: true, script: "docker images ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_B${env.BUILD_NUMBER}_C${env.SHORT_COMMIT} -q"
                        echo "Escaneando imagen con id: ${env.IMAGE_ID}"
                        getImageVulnsFromQualys imageIds: "${env.IMAGE_ID}", useGlobalConfig: true
                        // Pushing to ECR Repository
                        docker.withRegistry( "https://${env.REGISTRY_URL}", 'ecr:us-east-1:aws-credential-registry') {
                            sh "docker tag ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_B${env.BUILD_NUMBER}_C${env.SHORT_COMMIT} ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"
                            sh "docker push ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"
                        }
					}
				}
			}
		}

        stage('DEPLOY') {
            when { expression { BRANCH_NAME ==~ /(development|release|master)/ } }
            parallel {
                stage('Deploy to Development') {
                    when { expression { env.BRANCH_NAME ==~ /(development)/ }}
                    steps{
						container ('default-slave' ){
							script {
								// Init Stage Slack Message
								slackNotif.initStageMessage()
                                echo "Deploying version... ${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"         
                                docker.withRegistry("https://${env.REGISTRY_URL}", 'ecr:us-east-1:aws-credential-registry') {
                                    sh "docker pull ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"
                                    sh "docker tag ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION} ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"
                                    sh "docker push ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"
                                }
                            }
                        }
                    }
                }

                stage('Deploy to PP') {
                    when { expression { BRANCH_NAME ==~ /(release)/ } }
                    steps{
						container ('default-slave' ){
							script {
								// Init Stage Slack Message
								slackNotif.initStageMessage()
                                echo "Deploying version... ${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"
                                sh 'git config --local credential.helper "!p() { echo username=\\$GIT_USERNAME; echo password=\\$GIT_PASSWORD; }; p"'
                                sh "git tag ${env.VERSION}"
                                withCredentials([usernamePassword(credentialsId: 'bitbucket', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                                    sh "git push origin : ${env.VERSION}"
                                }

                                docker.withRegistry("https://${env.REGISTRY_URL}", 'ecr:us-east-1:aws-credential-registry') {
                                    sh "docker pull ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"
                                    sh "docker tag ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION} ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"
                                    sh "docker push ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:${env.GIT_REPO_NAME}_${env.TAG_NAME}_${env.VERSION}"
                                }
								
                            }
                        }
                    }
                }

                stage('Deploy to Production') {
                    when { expression { BRANCH_NAME ==~ /(master)/ }}
                    steps{
						container ('default-slave' ){
							script{
								// Init Stage Slack Message
								slackNotif.initStageMessage()
                                echo "Deploying version... ${env.GIT_REPO_NAME}_v${env.VERSION}"

                                docker.withRegistry( "https://${env.REGISTRY_URL}", 'ecr:us-east-1:aws-credential-registry') {
                                    sh "docker pull ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_${env.GIT_REPO_NAME}_release_${env.VERSION}"
                                    sh "docker tag ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:build_${env.GIT_REPO_NAME}_release_${env.VERSION} ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:${env.GIT_REPO_NAME}_v${env.VERSION}"
                                    sh "docker push ${env.REGISTRY_URL}/${env.REGISTRY_NAME}:${env.GIT_REPO_NAME}_v${env.VERSION}"
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}