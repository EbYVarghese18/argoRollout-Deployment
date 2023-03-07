pipeline {
    agent any

    environment {
        // DATE = new Date().format('yy.M')
        // TAG = "${DATE}.${BUILD_NUMBER}"
        AWS_REGION = 'us-east-1'
        AWS_OUTPUT_FORMAT = 'json'
        APP_NAME = 'myapp'
        CHART_NAME = 'myapp'
        CHART_VERSION = '0.${BUILD_NUMBER}'
        ECR_REPOSITORY = '095919053879.dkr.ecr.us-east-1.amazonaws.com'
    }
    
    stages {

        stage('SCM checkout') {
            steps {
                echo 'checkout starts'
                checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/EbYVarghese18/argoRollout-Deployment']])
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Build Dockerimage starts'
                script{
                    sh "docker build -t ${APP_NAME}:0.${BUILD_NUMBER} ."
                }
            }
        }
        
        stage('Docker Login to AWS ECR') {
			steps {
                echo 'Docker Login to AWS ECR starts'
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-access-key', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh 'aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID'
                    sh 'aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY'
                    sh 'aws configure set default.region $AWS_REGION'
                    sh 'aws configure set default.output $AWS_OUTPUT_FORMAT'
                }
                script{
                    sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_REPOSITORY}"
                }
            }
		}

        stage('Tag and Push image to AWS ECR') {
			steps {
                script{
                    echo 'Push the image to ECR starts'
                    sh "docker tag ${APP_NAME}:0.${BUILD_NUMBER} ${ECR_REPOSITORY}/${APP_NAME}:latest"
                    sh "docker push ${ECR_REPOSITORY}/${APP_NAME}:latest"
                }
			}
		}

        stage('Build Helm chart and Push to ECR') {
            steps {
                script{
                    echo 'creating helm chart'
                    sh "helm create ${CHART_NAME}"

                    echo 'updating image tag in value file'
                    sh "sed -i 's|tag: \".*\"|tag: \"0.${BUILD_NUMBER}\"|' ${CHART_NAME}/values.yaml"

                    // echo 'updating the repository field'
                    // sh "sed -i 's|repository: \".*\"|repository: \"${ECR_REPOSITORY}/${APP_NAME}:latest\"|' ${CHART_NAME}/values.yaml"

                    echo 'Builing helm package'
                    sh "helm package ${CHART_NAME} --version ${CHART_VERSION}"

                    echo "pushing the chart to ECR"
                    sh "aws ecr get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin ${ECR_REPOSITORY}"
                    sh "helm push ${CHART_NAME}-${CHART_VERSION}.tgz oci://${ECR_REPOSITORY}"
                    
                    echo 'Cleanig up the files'
                    sh "rm -rf ${CHART_NAME}"
                    sh "rm -rf ${CHART_NAME}-${CHART_VERSION}.tgz"
                }       
            }
        }

        // stage('Invoke Build Number to Pipeline appnginx-pullanddeploy') {
        //     steps {
        //         build job: 'appnginx-pullanddeploy', parameters : [[ $class: 'StringParameterValue', name: 'buildnumber', value: "${BUILD_NUMBER}"]]
        //     }
        // }

	}
}