pipeline {
    agent any

    environment {
        VENV_DIR   = 'venv'
        // 🔁 CHANGE this to the output from `where python`
        PYTHON_EXE = 'C:\\Users\\SAI DHRUVA\\AppData\\Local\\Programs\\Python\\Python310\\python.exe'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Clean venv') {
            steps {
                bat """
                IF EXIST %VENV_DIR% (
                    rmdir /S /Q %VENV_DIR%
                )
                """
            }
        }

        stage('Set up Python venv') {
            steps {
                // create venv and upgrade pip
                bat "\"%PYTHON_EXE%\" -m venv %VENV_DIR%"
                bat "%VENV_DIR%\\Scripts\\python -m pip install --upgrade pip"
            }
        }

        stage('Install dependencies') {
            steps {
                bat "%VENV_DIR%\\Scripts\\pip install -r requirements.txt"
            }
        }

        stage('Run Tests') {
            steps {
                // generate junit xml for Jenkins
                bat "%VENV_DIR%\\Scripts\\pytest scripts/ --junitxml=test-results.xml"
            }
        }
    }

    post {
        always {
            // Only publish JUnit report if it exists (to avoid that error)
            script {
                if (fileExists('test-results.xml')) {
                    junit 'test-results.xml'
                } else {
                    echo 'No test-results.xml found, skipping JUnit step.'
                }
            }
        }
    }
}
