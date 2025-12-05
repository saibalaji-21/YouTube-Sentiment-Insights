pipeline {
  agent any

  environment {
    VENV_DIR = 'venv'
    // Set your actual Python path below for Windows
    PYTHON_EXE = 'C:\\Users\\SAI DHRUVA\\AppData\\Local\\Programs\\Python\\Python313\\python.exe'
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
    disableConcurrentBuilds()
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Clean venv') {
      steps {
        script {
          if (isUnix()) {
            sh 'rm -rf ${VENV_DIR} || true'
          } else {
            bat 'IF EXIST %VENV_DIR% ( rmdir /S /Q %VENV_DIR% )'
          }
        }
      }
    }

    stage('Set up Python venv') {
      steps {
        script {
          if (isUnix()) {
            sh '''
              python3 -m venv ${VENV_DIR} || python -m venv ${VENV_DIR}
              . ${VENV_DIR}/bin/activate
              python -m pip install --upgrade pip
            '''
          } else {
            bat '"%PYTHON_EXE%" -m venv %VENV_DIR%'
            bat '%VENV_DIR%\\Scripts\\python -m pip install --upgrade pip'
          }
        }
      }
    }

    stage('Install dependencies') {
      steps {
        script {
          if (fileExists('requirements.txt')) {
            if (isUnix()) {
              sh '''
                . ${VENV_DIR}/bin/activate
                python -m pip install -r requirements.txt
              '''
            } else {
              bat '%VENV_DIR%\\Scripts\\pip install -r requirements.txt'
            }
          } else {
            echo 'No requirements.txt found, skipping install.'
          }
        }
      }
    }

    stage('Run Tests') {
      steps {
        script {
          if (isUnix()) {
            sh '. ${VENV_DIR}/bin/activate; python -m pytest scripts/ --junitxml=test-results.xml || true'
          } else {
            bat '%VENV_DIR%\\Scripts\\pytest scripts/ --junitxml=test-results.xml || exit /b 0'
          }
        }
      }
    }
  }

  post {
    always {
      script {
        if (fileExists('test-results.xml')) {
          junit 'test-results.xml'
        } else {
          echo 'No test-results.xml found, skipping JUnit step.'
        }
      }
    }
    success {
      echo "Pipeline finished: tests run."
    }
    failure {
      echo "Pipeline failed — check Console Output."
    }
  }
}
