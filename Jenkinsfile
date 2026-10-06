// Сборка и деплой тестового приложения.
//
// Job в Jenkins — multibranch: она видит и ветку main, и теги. Разделение
// сценариев поэтому делается одной строкой — заполнен ли TAG_NAME, — а не
// двумя сборками и угадыванием версии, как было в TeamCity.
pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    REGISTRY   = 'cr.yandex/crpj27virc0v36d3u7rq/hub'
    NAMESPACE  = 'web-app'
    DEPLOYMENT = 'web-app'
  }

  stages {
    stage('Build and push image') {
      steps {
        // Контекст сборки — server/: там лежат Dockerfile, nginx.conf и index.html.
        sh '''
          set -e
          if [ -n "${TAG_NAME:-}" ]; then
            VERSION="$TAG_NAME"
          else
            VERSION="${BRANCH_NAME}-${BUILD_NUMBER}"
          fi
          echo "Собираю $REGISTRY:$VERSION"

          cat /keys/key.json | docker login --username json_key --password-stdin cr.yandex

          # Docker 29 собирает OCI-манифест с аттестациями (provenance/sbom),
          # а реестр Yandex Cloud принимает только классический Docker v2 schema 2.
          # Без этих флагов push падает с «Cannot read manifest data».
          docker buildx build --provenance=false --sbom=false \
            --output type=image,oci-mediatypes=false,push=true \
            -t "$REGISTRY:$VERSION" \
            server/

          # latest двигаем только на коммите в main: у релизного тега своя метка.
          if [ -z "${TAG_NAME:-}" ]; then
            docker buildx build --provenance=false --sbom=false \
              --output type=image,oci-mediatypes=false,push=true \
              -t "$REGISTRY:latest" \
              server/
          fi
        '''
      }
    }

    stage('Deploy on tag') {
      when { expression { env.TAG_NAME != null && !env.TAG_NAME.isEmpty() } }
      steps {
        sh '''
          set -e
          echo "Прокатываю $REGISTRY:$TAG_NAME"
          kubectl -n "$NAMESPACE" set image deployment/$DEPLOYMENT \
            $DEPLOYMENT="$REGISTRY:$TAG_NAME"
          kubectl -n "$NAMESPACE" rollout status deployment/$DEPLOYMENT --timeout=180s
          kubectl -n "$NAMESPACE" get deployment $DEPLOYMENT -o wide
        '''
      }
    }
  }
}
