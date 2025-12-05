pipeline {
  agent any

  environment {
    // VENV_DIR used on both platforms
    VENV_DIR = 'venv'
    // Default python executable (Windows users may override via Jenkins config)
    PYTHON_EXE = 'python'
    APP_NAME = 'yt-sentiment-app'
    IMAGE_NAME = 'yt-sentiment'
    PIP_CACHE_DIR = '${WORKSPACE}/.pip_cache'
  }

  // Useful pipeline-level options
  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '25'))
    disableConcurrentBuilds()
    ansiColor('xterm')
  }

  parameters {
    booleanParam(defaultValue: true, description: 'Run DVC repro if DVC is available', name: 'RUN_DVC')
    booleanParam(defaultValue: false, description: 'Build Docker image', name: 'BUILD_IMAGE')
    booleanParam(defaultValue: false, description: 'Push Docker image (requires credentials)', name: 'PUSH_IMAGE')
    booleanParam(defaultValue: false, description: 'Deploy built image on agent (Unix-only)', name: 'DEPLOY')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Prepare') {
      steps {
        script {
          // Ensure pip cache directory exists (helps speed up repeated builds)
          sh script: "mkdir -p ${PIP_CACHE_DIR}", label: 'Create pip cache' , returnStatus: true
        }
      }
    }

    stage('Setup Python') {
      steps {
        script {
          if (isUnix()) {
            sh '''
              set -e
              python3 -m venv ${VENV_DIR} || python -m venv ${VENV_DIR}
              . ${VENV_DIR}/bin/activate
              python -m pip install --upgrade pip
            '''
          } else {
            bat "%PYTHON_EXE% -m venv %VENV_DIR%"
            bat "%VENV_DIR%\\Scripts\\python -m pip install --upgrade pip"
          }
        }
      }
    }

    stage('Install Dependencies') {
      steps {
        script {
          if (fileExists('requirements.txt')) {
            if (isUnix()) {
              sh '''
                set -e
                . ${VENV_DIR}/bin/activate
                python -m pip install --upgrade pip
                python -m pip install --cache-dir ${PIP_CACHE_DIR} -r requirements.txt
              '''
            } else {
              bat "%VENV_DIR%\\Scripts\\pip install --cache-dir ${PIP_CACHE_DIR} -r requirements.txt"
            }
          } else {
            echo 'No requirements.txt — skipping install.'
          }
        }
      }
    }

    stage('Lint (optional)') {
      when { expression { return fileExists('requirements.txt') && (readFile('requirements.txt').contains('flake8') || readFile('requirements.txt').contains('ruff')) } }
      steps {
        script {
          if (isUnix()) {
            sh '''
              . ${VENV_DIR}/bin/activate
              if command -v flake8 >/dev/null 2>&1; then
                flake8 . || true
              elif command -v ruff >/dev/null 2>&1; then
                ruff check . || true
              else
                echo 'No linter installed.'
              fi
            '''
          } else {
            bat "%VENV_DIR%\\Scripts\\python -m flake8 . || echo Lint not installed"
          }
        }
      }
    }

    stage('Run Tests') {
      steps {
        script {
          if (isUnix()) {
            sh '''
              set -e
              . ${VENV_DIR}/bin/activate
              python -m pytest scripts/ --junitxml=test-results.xml || true
            '''
          } else {
            bat "%VENV_DIR%\\Scripts\\pytest scripts/ --junitxml=test-results.xml || exit /b 0"
          }
        }
      }
    }

    stage('DVC repro (optional)') {
      when { expression { return params.RUN_DVC } }
      steps {
        script {
          if (isUnix()) {
            sh '''
              set -e
              if command -v dvc >/dev/null 2>&1 && [ -f dvc.yaml ]; then
                dvc pull || true
                dvc repro || true
              else
                echo 'DVC not available or no dvc.yaml present.'
              fi
            '''
          } else {
            bat "where dvc || echo DVC not available"
          }
        }
      }
    }

    stage('Package / Build artifacts') {
      steps {
        script {
          if (fileExists('setup.py')) {
            if (isUnix()) {
              sh '''
                . ${VENV_DIR}/bin/activate
                python setup.py sdist bdist_wheel || true
              '''
            } else {
              bat "%VENV_DIR%\\Scripts\\python setup.py sdist bdist_wheel || echo packaging failed"
            }
          } else {
            echo 'No setup.py — skipping packaging.'
          }
        }
      }
    }

    stage('Archive artifacts') {
      steps {
        archiveArtifacts artifacts: 'dist/**,**/*.pkl,**/models/**', allowEmptyArchive: true
      }
    }

    stage('Build Docker Image') {
      when { expression { return params.BUILD_IMAGE && isUnix() && fileExists('Dockerfile') } }
      steps {
        script {
          env.SHORT_COMMIT = sh(script: "git rev-parse --short=7 HEAD", returnStdout: true).trim()
          sh '''
            IMAGE_TAG="${IMAGE_NAME}:${SHORT_COMMIT}"
            docker build -t "${IMAGE_TAG}" .
            echo "${IMAGE_TAG}" > .built_image_tag
          '''
        }
      }
    }

    stage('Push Docker Image') {
      when { expression { return params.PUSH_IMAGE && isUnix() && fileExists('.built_image_tag') } }
      steps {
        // Use Jenkins credential with id DOCKERHUB_CREDENTIALS (username/password)
        withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_CREDENTIALS', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            IMAGE_TAG=$(cat .built_image_tag)
            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
            docker push "$IMAGE_TAG"
          '''
        }
      }
    }

    stage('Deploy (agent)') {
      when { expression { return params.DEPLOY && isUnix() && fileExists('.built_image_tag') } }
      steps {
        script {
          sh '''
            IMAGE_TAG=$(cat .built_image_tag)
            if docker ps -a --format '{{.Names}}' | grep -Eq "^${APP_NAME}$"; then
              docker rm -f ${APP_NAME} || true
            fi
            docker run -d --name ${APP_NAME} -p 5000:5000 --restart unless-stopped ${IMAGE_TAG}
          '''
        }
      }
    }

  }

  post {
    always {
      script {
        if (fileExists('test-results.xml')) {
          junit allowEmptyResults: true, testResults: 'test-results.xml'
        } else {
          echo 'No JUnit results found.'
        }
      }
      cleanWs()
    }
    success {
      echo 'Pipeline succeeded.'
    }
    failure {
      echo 'Pipeline finished with failures. Check logs.'
    }
  }
}
