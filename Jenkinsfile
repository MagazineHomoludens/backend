pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'homoludensmz/homoludens-backend'
        DOCKER_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
        DB_URL = credentials('db_url')
        DB_USERNAME = credentials('db_username')
        DB_PASSWORD = credentials('db_password')
        SERVER_PORT = credentials('server_port')
        PROD_SERVER_IP = credentials('prod-server-ip')
    }

    stages {
        stage('CI: Checkout Code') {
            steps {
                echo '🔄 [CI] 코드 체크아웃 단계 시작...'
                git branch: "${env.BRANCH_NAME}", credentialsId: 'homoludens_jenkins_token', url: 'https://github.com/MagazineHomoludens/Backend.git'
            }
        }

        stage('CI: Build Project') {
            steps {
                echo '🔄 [CI] Gradle 빌드 단계 시작...'
                sh 'chmod +x ./gradlew'
                sh './gradlew clean build'
            }
        }

        stage('CI: Build and Push Docker Image') {
            steps {
                echo '🔄 [CI] Docker 이미지 빌드 및 푸시 단계 시작...'
                withCredentials([usernamePassword(credentialsId: 'docker_hub_credentials', usernameVariable: 'DOCKER_HUB_USER', passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
                    sh '''
                    echo $DOCKER_HUB_PASSWORD | docker login -u $DOCKER_HUB_USER --password-stdin
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                    docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                    '''
                }
            }
        }

        stage('CD: Deploy to Development Server') {
            when {
                branch 'develop'
            }
            steps {
                echo '🚀 [CD] 개발 서버 배포 시작...'
                sh '''
                docker compose pull backend
                docker compose up -d backend
                '''
            }
        }

        stage('CD: Deploy to Production Server') {
            when {
                branch 'main'
            }
            steps {
                echo '🚀 [CD] 운영 서버 배포 시작...'
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'prod-ssh-key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                    usernamePassword(credentialsId: 'docker_hub_credentials', usernameVariable: 'DOCKER_HUB_USER', passwordVariable: 'DOCKER_HUB_PASSWORD')
                ]) {
                    sh '''
                    # 🔑 Docker Hub 로그인 (Jenkins 서버에서 수행)
                    echo $DOCKER_HUB_PASSWORD | docker login -u $DOCKER_HUB_USER --password-stdin

                    # 🚀 SSH로 운영 서버에 환경 변수 전달 및 배포
                    ssh -i $SSH_KEY $SSH_USER@${PROD_SERVER_IP} << ENDSSH
                    set -e

                    # 🛠️ 환경 변수 설정
                    export DOCKER_TAG="${DOCKER_TAG}"
                    export DB_URL="${DB_URL}"
                    export DB_USERNAME="${DB_USERNAME}"
                    export DB_PASSWORD="${DB_PASSWORD}"
                    export SERVER_PORT="${SERVER_PORT}"

                    # 📥 docker-compose.yml 다운로드
                    mkdir -p /home/ubuntu/backend
                    cd /home/ubuntu/backend
                    curl -o docker-compose.yml https://raw.githubusercontent.com/MagazineHomoludens/Backend/main/docker-compose-prod.yml

                    echo "✅ 환경 변수 설정 확인:"
                    echo "DOCKER_TAG=${DOCKER_TAG}"
                    echo "DB_URL=${DB_URL}"
                    echo "DB_USERNAME=${DB_USERNAME}"
                    echo "DB_PASSWORD=${DB_PASSWORD}"
                    echo "SERVER_PORT=${SERVER_PORT}"

                    # 🐳 Docker Compose 실행
                    docker compose -f docker-compose.yml pull backend
                    docker compose -f docker-compose.yml up -d backend
                    ENDSSH
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ 전체 배포 성공!'
        }
        failure {
            echo '❌ 전체 배포 실패. 로그 확인 필요!'
        }
    }
}
