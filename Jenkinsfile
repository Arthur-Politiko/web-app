// Сборка и деплой приложения.
// собрать образ из server/
// и прокатить его в кластер.
//
pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  stages {
    // Явная проверка окружения: если контейнер не передал переменные,
    // сборка падает здесь с понятным текстом, а не на середине команды.
    stage('Check environment') {
      steps {
        sh '''
          : "${REGISTRY:?REGISTRY не задан — его передаёт контейнер Jenkins}"
          : "${APP_NAMESPACE:?APP_NAMESPACE не задан}"
          : "${APP_DEPLOYMENT:?APP_DEPLOYMENT не задан}"
          : "${REGISTRY_KEY_FILE:?REGISTRY_KEY_FILE не задан}"
        '''
      }
    }

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

          cat "$REGISTRY_KEY_FILE" | docker login \
            --username json_key --password-stdin "${REGISTRY%%/*}"

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
          kubectl -n "$APP_NAMESPACE" set image deployment/$APP_DEPLOYMENT \
            $APP_DEPLOYMENT="$REGISTRY:$TAG_NAME"
          kubectl -n "$APP_NAMESPACE" rollout status deployment/$APP_DEPLOYMENT --timeout=180s
          kubectl -n "$APP_NAMESPACE" get deployment $APP_DEPLOYMENT -o wide
        '''
      }
    }
  }
}
