# Лабораторная работа №5

**Тема:** Реализация архитектуры на основе сервисов (микросервисной архитектуры)

**Цель работы:** Получить опыт работы организации взаимодействия сервисов с использованием контейнеров Docker

## Развертывание контейнеров

Развертывание каждого приложения происходит с помощью подобного docker-compose файла:
```yaml
version: '3.8'
services:
  sigma-app:
    image: ${IMAGE_TAG:-ghcr.io/melowetty/sigma-main-service:stable}
    deploy:
      restart_policy:
        condition: on-failure
      resources:
        limits:
          memory: 1.5GB
      update_config:
        parallelism: 1
      replicas: 2
    healthcheck:
      test: [ "CMD", "/workspace/health-check" ]
      interval: 10s
      timeout: 10s
      retries: 3
      start_period: 60s
    environment:
      JAVA_TOOL_OPTIONS: "-XX:MaxMetaspaceSize=256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps/heapdump.hprof"
      THC_PORT: 8000
      THC_PATH: '/actuator/health'
    volumes:
      - heapdumps:/dumps
    depends_on:
      - db
    env_file: ./.env
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "5"
    ports:
      - published: 8000
        target: 8000
        protocol: tcp
        mode: ingress
    networks:
      - internal
      - logging
      - message-broker_kafka-net
      - monitoring

  db:
    image: postgres:17.6-alpine
    env_file: ./.env
    ports:
      - published: 5433
        target: 5432
        protocol: tcp
        mode: ingress
    deploy:
      restart_policy:
        condition: on-failure
      resources:
        limits:
          memory: 512M
    volumes:
      - db:/var/lib/postgresql/data
    networks:
      - internal

volumes:
  db:
    name: main-service-db
  heapdumps:
    name: java-heapdumps

networks:
  internal:
    driver: overlay
    attachable: true
  message-broker_kafka-net:
    external: true
  monitoring:
    external: true
  logging:
    external: true
```

Запуск этой серии контейнеров производится с помощью команды
```bash
docker stack deploy -c docker-compose.yml main-service --with-registry-auth
```

Таким же образом стартуют еще два сервиса.

Тестирование контейнеров было произведено в 4 лабораторной работе.

## Непрерывная интеграция

Непрерывная интеграция была реализована с помощью GitHub Actions.

```yaml
name: Backend CI

on:
  pull_request:
    branches: [master, dev]
    paths:
      - common/**
      - notification-service/**
      - auth/**
      - 'src/main/**'
      - 'src/test/**'
      - 'build.gradle*'
      - 'settings.gradle*'
      - 'gradle.properties'
      - 'gradle/wrapper/gradle-wrapper.properties'
      - 'Dockerfile'
      - 'docker-compose*.yml'
      - 'scripts/**'
      - '.github/workflows/**'

jobs:
  quality:
    name: "Quality checks (detekt + tests + coverage)"
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: 'temurin'

      - name: Restore Cache Gradle dependencies
        id: cache
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-deps-ci-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-deps-ci-

      - name: Run quality checks
        run: |
          chmod +x gradlew
          ./gradlew qualityCheck --build-cache

      - name: Coverage summary
        shell: bash
        run: |
          FILES=$(find . -path '*/build/reports/jacoco/test/jacocoTestReport.csv' -type f)
          if [ -z "$FILES" ]; then
            echo "No JaCoCo CSV reports found" >> $GITHUB_STEP_SUMMARY
            exit 0
          fi

          COVERAGE=$(awk -F, 'NR>1 { mi+=$4; ci+=$5 } END { total=mi+ci; if (total>0) printf "%.2f", (ci/total)*100; else print "0.00" }' $FILES)
          echo "### Test Coverage" >> $GITHUB_STEP_SUMMARY
          echo "- Line coverage: ${COVERAGE}%" >> $GITHUB_STEP_SUMMARY

      - name: Upload quality artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: ci-quality-reports
          path: |
            **/build/reports/detekt/**
            **/build/reports/jacoco/test/**
```

При мерж-реквесте в мастер или дев ветку запускается пайплайн и проверяется билд проекта и прохождение всех тестов

## Интеграционные тесты

Интеграционные тесты были написаны в коде сервиса с поднятием тестового окружения через Testcontainers. Тестируется поднятие сервиса и выполнение CRUD-операций с бд.

```kotlin
...

class OrderRepositoryIntegrationTest : IntegrationTest() {

    @Autowired
    private lateinit var orderRepository: OrderRepository

    @Autowired
    private lateinit var userRepository: UserRepository

    @BeforeEach
    fun setUp() {
        userRepository.save(user)
    }

    @AfterEach
    fun clear() {
        orderRepository.deleteAll()
        userRepository.deleteAll()
    }

    @Nested
    inner class tests_findByPaymentId {
        @Test
        fun positive() {
            // given
            orderRepository.saveAndFlush(createOrder())

            // when
            val order = orderRepository.findByPaymentId(paymentId)

            // then
            assertThat(order).isNotNull
            assertThat(order!!.paymentId).isEqualTo(paymentId)
            assertThat(order.status).isEqualTo(OrderStatus.PAID)
        }

        @Test
        fun negative_notFound() {
            // when
            val order = orderRepository.findByPaymentId("non_existent_payment_id")

            // then
            assertThat(order).isNull()
        }
    }

    @Nested
    inner class tests_updateStatusByPaymentId {
        @Test
        fun positive() {
            // given
            orderRepository.saveAndFlush(createOrder())

            // when
            orderRepository.updateStatusByPaymentId(paymentId, OrderStatus.REFUND)

            // then
            val order = orderRepository.findAll().first()

            assertThat(order.paymentId).isEqualTo(paymentId)
            assertThat(order.status).isEqualTo(OrderStatus.REFUND)
        }
    }

    @Nested
    inner class tests_updateReceiptIdByPaymentId {
        @Test
        fun positive() {
            // given
            orderRepository.saveAndFlush(createOrder())
            val receiptId = "receiptId123"

            // when
            orderRepository.updateReceiptIdByPaymentId(paymentId, receiptId)

            // then
            val order = orderRepository.findAll().first()

            assertThat(order.paymentId).isEqualTo(paymentId)
            assertThat(order.receiptId).isEqualTo(receiptId)
        }
    }

    companion object {
        private const val paymentId = "paymentId"
        private const val income = 1000

        private val user = TestDataHolder.generateUser()

        private fun createOrder() = OrderEntity(
            userId = user.id,
            paymentId = paymentId,
            income = income,
            status = OrderStatus.PAID,
        )
    }
}

...

@Testcontainers
@SpringBootTest
@TestPropertySource(
    properties = [
        "spring.datasource.url=jdbc:tc:postgresql:16-alpine:///db"
    ]
)
abstract class IntegrationTest {
    companion object {

        @Container
        val kafka = ConfluentKafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.8.0"))

        @DynamicPropertySource
        @JvmStatic
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers)
        }
    }
}
```

Интеграционные тесты в пайплайне не отличаются от обычных поэтому проходят в общем воркфлоу.

## Непрерывное развертывание

Непрерывное развертывание включает в себя публикацию образа сервиса в GHCR и его деплой на продакшене.

```yaml
name: Backend CD

on:
  push:
    branches: [master, dev]
    paths:
      - common/**
      - notification-service/**
      - auth/**
      - 'src/main/**'
      - 'src/test/**'
      - 'build.gradle*'
      - 'settings.gradle*'
      - 'gradle.properties'
      - 'gradle/wrapper/gradle-wrapper.properties'
      - 'Dockerfile'
      - 'docker-compose*.yml'
      - 'scripts/**'
      - '.github/workflows/**'
  workflow_dispatch:
    inputs:
      confirm_rollback:
        description: "Type ROLLBACK to deploy stable image tag"
        required: true
        default: "ROLLBACK"

concurrency:
  group: backend-cd-${{ github.ref }}
  cancel-in-progress: true

env:
  IMAGE_NAME: sigma-main-service
  STABLE_TAG: stable
  MASTER_SERVER_DEPLOY_PATH: /services/sigma-app

jobs:
  quality:
    name: "Quality checks (detekt + tests + coverage)"
    runs-on: ubuntu-latest
    if: github.event_name == 'push'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: 'temurin'

      - name: Cache Gradle dependencies
        id: cache
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-deps-cd-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-deps-cd-

      - name: Run quality checks
        run: |
          chmod +x gradlew
          ./gradlew qualityCheck --build-cache

      - name: Coverage summary
        shell: bash
        run: |
          FILES=$(find . -path '*/build/reports/jacoco/test/jacocoTestReport.csv' -type f)
          if [ -z "$FILES" ]; then
            echo "No JaCoCo CSV reports found" >> $GITHUB_STEP_SUMMARY
            exit 0
          fi

          COVERAGE=$(awk -F, 'NR>1 { mi+=$4; ci+=$5 } END { total=mi+ci; if (total>0) printf "%.2f", (ci/total)*100; else print "0.00" }' $FILES)
          echo "### Test Coverage" >> $GITHUB_STEP_SUMMARY
          echo "- Line coverage: ${COVERAGE}%" >> $GITHUB_STEP_SUMMARY

      - name: Upload quality artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: cd-quality-reports
          path: |
            **/build/reports/detekt/**
            **/build/reports/jacoco/test/**

  build-image:
    name: "Build docker image"
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    needs: quality
    permissions:
      contents: read
      packages: write

    outputs:
      image_id: ${{ steps.image-build.outputs.image_id }}
      tag: ${{ steps.image-build.outputs.tag }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: 'temurin'

      - name: Restore Cache Gradle dependencies
        id: cache
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-deps-cd-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-deps-cd-

      - name: Build docker image with cache
        run: |
          chmod +x gradlew
          ./gradlew bootBuildImage --imageName=$IMAGE_NAME --build-cache

      - name: Login to GitHub Container Registry
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Tag & publish to GitHub Container Registry
        id: image-build
        run: |
          REPOSITORY_OWNER=$(echo "${{ github.repository_owner }}" | tr '[:upper:]' '[:lower:]')
          IMAGE_ID=ghcr.io/$REPOSITORY_OWNER/$IMAGE_NAME
          SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-7)
          BRANCH_NAME=$(echo "${{ github.ref_name }}" | tr '/' '-')
          TAG="${BRANCH_NAME}-${SHORT_SHA}"
          echo "tag=$TAG" >> $GITHUB_OUTPUT
          echo "image_id=$IMAGE_ID" >> $GITHUB_OUTPUT
          IMAGE=$IMAGE_ID:$TAG
          docker tag $IMAGE_NAME $IMAGE
          docker push $IMAGE

  deploy:
    name: "Deploy new project version"
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    needs: build-image

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy new project version to master server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.KEY }}
          port: ${{ secrets.PORT }}
          script: |
            cd ${{ env.MASTER_SERVER_DEPLOY_PATH }}
            echo "Running on ${{ needs.build-image.outputs.tag }} environment"
            ./deploy.sh "${{ needs.build-image.outputs.tag }}"

      - name: Mark as stable after success
        if: success()
        env:
          IMAGE_ID: ${{ needs.build-image.outputs.image_id }}
          TAG: ${{ needs.build-image.outputs.tag }}
        run: |
          PR_IMAGE=$IMAGE_ID:$TAG
          STABLE_IMAGE=$IMAGE_ID:$STABLE_TAG
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker pull $PR_IMAGE
          docker tag $PR_IMAGE $STABLE_IMAGE
          docker push $STABLE_IMAGE

  rollback:
    name: "Rollback to stable"
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch' && github.event.inputs.confirm_rollback == 'ROLLBACK'
    steps:
      - name: Deploy stable image tag to master server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.KEY }}
          port: ${{ secrets.PORT }}
          script: |
            cd ${{ env.MASTER_SERVER_DEPLOY_PATH }}
            echo "Manual rollback to stable tag"
            ./deploy.sh "${{ env.STABLE_TAG }}"
```