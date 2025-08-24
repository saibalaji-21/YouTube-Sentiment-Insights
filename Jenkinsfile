pipeline {
    agent any
    environment {
        VENV_DIR = 'venv'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Install Python') {
            steps {
                sh 'apt-get update && apt-get install -y python3 python3-venv python3-pip'
            }
        }
        stage('Set up Python venv') {
            steps {
                sh 'python3 -m venv $VENV_DIR'
                sh '. $VENV_DIR/bin/activate; python -m pip install --upgrade pip'
            }
        }
        stage('Install dependencies') {
            steps {
                sh '. $VENV_DIR/bin/activate; pip install -r requirements.txt'
            }
        }
        stage('Run Tests') {
            steps {
                sh '. $VENV_DIR/bin/activate; pytest scripts/'
            }
        }
    }
    post {
        always {
            junit '**/test-*.xml'
        }
    }
}
