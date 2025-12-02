pipeline {
  agent any

  environment {
    PYTHONUNBUFFERED = '1'
    APP_NAME = 'yt-sentiment-app'
    IMAGE_NAME = 'yt-sentiment'
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

    stage('Install Python deps') {
      steps {
        sh '''
          set -e
          python3 -m pip install --upgrade pip || python -m pip install --upgrade pip
          if [ -f requirements.txt ]; then
            pip install -r requirements.txt
          fi
        '''
      }
    }

    stage('Run Python files in src/') {
      steps {
        sh '''
          set -e
          echo "Running Python files in src/..."
          if [ -d src ]; then
            ran=0
            for f in src/*.py; do
              [ -e "$f" ] || continue
              case "$(basename "$f")" in
                _* ) echo "Skipping $f (leading underscore)"; continue;;
              esac
              echo ">>> Running $f"
              python "$f"
              ran=1
            done
            if [ "$ran" -eq 0 ]; then
              echo "No python files found in src/ to run."
            fi
          else
            echo "No src/ directory found."
          fi
        '''
      }
    }

    stage('Archive model artifacts') {
      steps {
        archiveArtifacts artifacts: '**/*.pkl', allowEmptyArchive: false, fingerprint: true
      }
    }

    stage('Build Docker image (if Docker available)') {
      steps {
        script {
          // compute short commit for tag
          env.SHORT_COMMIT = sh(script: "git rev-parse --short=7 HEAD", returnStdout: true).trim()
        }
        sh '''
          set -e
          if command -v docker >/dev/null 2>&1 && [ -f Dockerfile ]; then
            IMAGE_TAG="${IMAGE_NAME}:${SHORT_COMMIT}"
            echo "Building Docker image ${IMAGE_TAG} ..."
            docker build -t "${IMAGE_TAG}" .
            echo "${IMAGE_TAG}" > .built_image_tag || true
          else
            echo "Docker CLI not available or no Dockerfile — skipping Docker build."
          fi
        '''
      }
    }

    stage('Deploy container (if image built)') {
      steps {
        sh '''
          set -e
          if [ -f .built_image_tag ]; then
            IMAGE_TAG=$(cat .built_image_tag)
            # stop & remove any existing container with same name
            if docker ps -a --format '{{.Names}}' | grep -Eq "^${APP_NAME}$"; then
              echo "Stopping existing container ${APP_NAME} ..."
              docker rm -f ${APP_NAME} || true
            fi
            echo "Running container ${APP_NAME} from ${IMAGE_TAG} ..."
            docker run -d --name ${APP_NAME} -p 5000:5000 --restart unless-stopped ${IMAGE_TAG}
          else
            echo "No image built previously — skipping deploy."
          fi
        '''
      }
    }
  }

  post {
    success {
      echo "Pipeline finished: scripts run, artifacts archived, and app deployed if Docker was available."
    }
    failure {
      echo "Pipeline failed — check Console Output."
    }
  }
}
