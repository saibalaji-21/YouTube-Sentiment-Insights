pipeline {
    agent {
        docker {
            image 'python:3.11-slim'
        }
    }

    environment {
        PYTHONUNBUFFERED = "1"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python -m pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Run Python Scripts in src/') {
            steps {
                sh '''
                    echo "Running Python files from src/ ..."

                    # If you want to run ALL python files in src:
                    for f in src/*.py; do
                        echo "Running $f"
                        python "$f"
                    done
                '''
            }
        }

        stage('Archive Model Artifacts') {
            steps {
                archiveArtifacts artifacts: '**/*.pkl', allowEmptyArchive: false, fingerprint: true
            }
        }
    }

    post {
        success {
            echo "Pipeline executed successfully. Models archived."
        }
        failure {
            echo "Pipeline failed. Check logs."
        }
    }
}
