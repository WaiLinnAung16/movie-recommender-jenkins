pipeline {
    agent any

    environment {
        IMAGE_NAME = "myapp6"
        NETWORK_NAME = "app-network"
        // Use the container name 'mysql-db-dast' for internal Docker network communication
        DB_URI_DOCKER = "mysql+pymysql://root:root@mysql-db-dast:3306/imdb_db"
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-credentials')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/WaiLinnAung16/movie-recommender-jenkins.git',
                    credentialsId: '4fb414c0-45f5-419f-98cc-e4c94b3e322a'
            }
        }

        stage('Setup Docker Network') {
            steps {
                sh 'docker network create ${NETWORK_NAME} || true'
            }
        }

        stage('Launch MySQL') {
            steps {
                sh '''
                # Forcefully remove the container even if it's "stuck" in removal
                docker rm -f mysql-db-dast || true
                
                # Run MySQL. 
                # If 3306 is still busy, change the left side to 3307:3306
                docker run -d --name mysql-db-dast \
                    -e MYSQL_ROOT_PASSWORD=root \
                    -e MYSQL_DATABASE=imdb_db \
                    --network ${NETWORK_NAME} \
                    -p 3306:3306 \
                    mysql:8.0
                
                echo "Waiting for MySQL..."
                sleep 20 
                '''
            }
        }

        stage('Setup Python & Lint') {
            steps {
                sh '''
                python3 -m venv venv
                . venv/bin/activate
                pip install -r requirements.txt
                pip install bandit safety pip-audit
                '''
            }
        }

        stage('SCA - Security Scans') {
            steps {
                sh '''
                . venv/bin/activate
                mkdir -p reports
                bandit -r . -f json -o bandit-report.json || true
                safety check --full-report > safety-report.txt || true
                '''
            }
        }

        stage('Build & Push') {
            steps {
                sh '''
                FULL_TAG="${DOCKERHUB_CREDENTIALS_USR}/${IMAGE_NAME}:latest"
                docker build -t $FULL_TAG .
                echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                docker push $FULL_TAG
                echo "FULL_IMAGE_TAG=$FULL_TAG" > image_tag.env
                '''
            }
        }
        stage('Unit Tests with Test Database') {
            steps {
                script {
                    // Run tests with MySQL accessible via localhost (published port)
                    sh """
                    . venv/bin/activate
                    export DB_URI="mysql+pymysql://root:root@localhost:3306/imdb_db"
                    
                    # Run tests with pytest
                    pytest -v --tb=short || echo "Tests completed with failures"
                    """
                }
            }
        }
        


        stage('Run App & DAST') {
            steps {
                sh '''
                # Existing app setup...
                docker stop myapp6 || true
                docker rm myapp6 || true
                docker run -d --name myapp6 --network app-network -p 5050:5050 \
                    -e DB_URI=mysql+pymysql://root:root@mysql-db-dast:3306/imdb_db \
                    myatmonoo/myapp6:latest
                
                echo "Waiting for app to initialize..."
                sleep 15
        
                
                # 1. Ensure the workspace is writable by the ZAP container user
                chmod 777 $(pwd)
                
                # 2. Run ZAP with the current user's UID to avoid permission mismatch
                # Or simply use the -u root flag to bypass permission checks (easiest for CI)
                docker run --user root --network app-network \
                    -v $(pwd):/zap/wrk:rw \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py -t http://myapp6:5050 -r zap-report.html -I
                
                '''
            }
        }


    }

    post {
        always {
            // Archive all reports
            archiveArtifacts artifacts: 'zap-report.html, bandit-report.json, safety-report.txt', allowEmptyArchive: true
            

        }
        

    }
}
