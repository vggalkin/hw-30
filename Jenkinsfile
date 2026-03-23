// Jenkinsfile - Declarative Pipeline
pipeline {
    
    // 2. Определение агента
    agent {
        // Используем Docker-агент для консистентности окружения [[32]]
        docker {
            image 'maven:3.9.3-eclipse-temurin-17'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
            label 'linux'
        }
    }
    
    // 6. Параметры Pipeline
    parameters {
        string(name: 'APP_VERSION', defaultValue: '1.0.0', description: 'Версия приложения')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Окружение для деплоя')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Запускать ли тесты?')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Ветка для сборки')
        password(name: 'DOCKER_PASSWORD', defaultValue: '', description: 'Пароль для Docker Registry')
    }
    
    // 7. Обработка ошибок и опции
    options {
        timeout(time: 30, unit: 'MINUTES')  // Таймаут всего пайплайна
        disableConcurrentBuilds()            // Запрет параллельных сборок
        buildDiscarder(logRotator(numToKeepStr: '10'))  // Хранить последние 10 сборок
        timestamps()                         // Добавлять время в лог
        skipDefaultCheckout()                // Пропустить стандартный checkout
    }
    
    // Environment variables
    environment {
        APP_NAME = 'my-web-app'
        DOCKER_REGISTRY = 'registry.example.com'
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${APP_NAME}"
        // Безопасное использование креденшалов [[6]]
        DOCKER_CREDS = credentials('docker-registry-creds')
    }
    
    stages {
        
        // 3. Этап: Сборка (Build)
        stage('Build') {
            when {
                // 5. Условие: только для определённых веток
                anyOf {
                    branch 'main'
                    branch 'develop'
                    expression { params.ENVIRONMENT != 'prod' }
                }
            }
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                // 4. Шаги с использованием плагинов
                script {
                    // Клонирование репозитория
                    checkout scm: [
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.GIT_BRANCH}"]],
                        extensions: [
                            [$class: 'CloneOption', depth: 1, shallow: true]
                        ],
                        userRemoteConfigs: [[url: 'https://github.com/username/repo.git']]
                    ]
                }
                
                // Сборка с помощью Maven
                sh '''
                    echo "🔨 Сборка версии ${params.APP_VERSION}"
                    mvn clean package -DskipTests -Dapp.version=${params.APP_VERSION}
                '''
                
                // Проверка успешности сборки
                script {
                    if (!fileExists('target/*.jar')) {
                        error "❌ Артефакты сборки не найдены!"
                    }
                }
            }
        }
        
        // 3. Этап: Тестирование (Test)
        stage('Test') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                sh '''
                    echo "🧪 Запуск тестов..."
                    mvn test
                '''
                
                // Публикация результатов тестов
                junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                
                // Сбор покрытия кода
                jacoco execPattern: 'target/jacoco.exec'
            }
            post {
                always {
                    // Архивация отчётов независимо от результата
                    archiveArtifacts artifacts: 'target/*.log', allowEmptyArchive: true
                }
                unstable {
                    echo "⚠️ Тесты прошли с предупреждениями"
                }
            }
        }
        
        // 3. Этап: Создание Docker-образа
        stage('Create Docker Image') {
            steps {
                script {
                    // 10b,c. Создание Docker-образа
                    def imageTag = "${DOCKER_IMAGE}:${params.APP_VERSION}-${env.BUILD_NUMBER}"
                    env.DOCKER_IMAGE_TAG = imageTag
                    
                    sh '''
                        echo "🐳 Сборка Docker-образа: ${DOCKER_IMAGE_TAG}"
                        docker build \
                            --build-arg VERSION=${params.APP_VERSION} \
                            --build-arg BUILD_NUMBER=${BUILD_NUMBER} \
                            -t ${DOCKER_IMAGE_TAG} \
                            -t ${DOCKER_IMAGE}:latest \
                            .
                    '''
                }
            }
        }
        
        // 3. Этап: Развёртывание (Deploy)
        stage('Deploy') {
            when {
                // Только для успешных сборок и определённых окружений
                allOf {
                    branch 'main'
                    expression { 
                        currentBuild.result == null || currentBuild.result == 'SUCCESS' 
                    }
                }
            }
            steps {
                script {
                    // 10d. Запуск Docker-контейнера
                    def containerName = "${APP_NAME}-${env.BUILD_NUMBER}"
                    
                    // 11. Удаление предыдущей версии контейнера
                    sh '''
                        echo "🗑️ Очистка предыдущих версий..."
                        docker stop ${APP_NAME}-latest 2>/dev/null || true
                        docker rm ${APP_NAME}-latest 2>/dev/null || true
                        
                        # Очистка старых образов (оставляем последние 5)
                        docker images ${DOCKER_REGISTRY}/${APP_NAME} --format "{{.ID}}" | tail -n +6 | xargs -r docker rmi
                    '''
                    
                    // Запуск нового контейнера
                    sh '''
                        echo "🚀 Деплой контейнера: ${DOCKER_IMAGE_TAG}"
                        docker run -d \
                            --name ${APP_NAME}-latest \
                            --restart unless-stopped \
                            -p 8080:8080 \
                            -e ENV=${params.ENVIRONMENT} \
                            -e VERSION=${params.APP_VERSION} \
                            ${DOCKER_IMAGE_TAG}
                    '''
                }
            }
            post {
                success {
                    echo "✅ Деплой завершён успешно!"
                }
                failure {
                    // Обработка ошибки деплоя
                    script {
                        echo "❌ Ошибка деплоя! Откат изменений..."
                        sh 'docker logs ${APP_NAME}-latest || true'
                    }
                }
            }
        }
        
        // 10e. Проверка доступности приложения
        stage('Health Check') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Повторные попытки проверки (обработка ошибок)
                    def maxRetries = 5
                    def retryCount = 0
                    def isHealthy = false
                    
                    while (retryCount < maxRetries && !isHealthy) {
                        try {
                            sh '''
                                curl -f --max-time 10 http://localhost:8080/health || exit 1
                            '''
                            isHealthy = true
                            echo "✅ Приложение доступно!"
                        } catch (Exception e) {
                            retryCount++
                            echo "⏳ Попытка ${retryCount}/${maxRetries}: Ожидание запуска..."
                            sleep(time: 10, unit: 'SECONDS')
                            
                            if (retryCount == maxRetries) {
                                error "❌ Приложение не запустилось после ${maxRetries} попыток"
                            }
                        }
                    }
                }
            }
        }
    }
    
    // Глобальная обработка пост-условий
    post {
        always {
            // Логирование результата — выполняется всегда
            echo "📊 Сборка #${env.BUILD_NUMBER} завершена: ${currentBuild.currentResult}"
        
            // Архивация критичных логов для отладки
            archiveArtifacts artifacts: '**/*.log', allowEmptyArchive: true, fingerprint: true
        }
    
        success {
            // Очистка только при успехе
            script {
                echo "✅ Сборка успешна — очищаем рабочее пространство"
                cleanWs(
                    deleteDirs: true,
                    notFailBuild: true,
                    cleanWhenAbsent: false
                )
            }
        
             // Уведомление для продакшена
            script {
                if (params.ENVIRONMENT == 'prod') {
                    echo "🎉 Продакшен-деплой версии ${params.APP_VERSION} успешен!"
                    // slackSend channel: '#deployments', message: "✅ Deployed ${params.APP_VERSION}"
                }
            }
        }
    
        failure {
            // При ошибке НЕ очищаем workspace — сохраняем для отладки
            echo "❌ Сборка не удалась — workspace сохранён для анализа"
        
            script {
                // Сохраняем дополнительные диагностические данные
                sh 'docker logs ${APP_NAME}-latest --tail 100 > docker-error.log 2>&1 || true'
                archiveArtifacts artifacts: 'docker-error.log', allowEmptyArchive: true
            }
        }
    
        unstable {
            echo "⚠️ Сборка завершена с предупреждениями (например, упали тесты)"
            // Можно отправить уведомление команде
        }
    
        changed {
            // Уведомление только если статус сборки изменился
            echo "🔄 Статус сборки изменился: ${currentBuild.previousBuild?.result} → ${currentBuild.currentResult}"
        }
    }
}

