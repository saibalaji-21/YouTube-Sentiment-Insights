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
        stage('Set up Python') {
            steps {
                sh 'python -m venv $VENV_DIR'
                sh '. $VENV_DIR/Scripts/activate; python -m pip install --upgrade pip'
            }
        }
        stage('Install dependencies') {
            steps {
                sh '. $VENV_DIR/Scripts/activate; pip install -r requirements.txt'
            }
        }
        stage('Run Tests') {
            steps {
                sh '. $VENV_DIR/Scripts/activate; pytest scripts/'
            }
        }
    }
    post {
        always {
            junit '**/test-*.xml'
        }
    }
}
