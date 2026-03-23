pipeline {
    // 3. Определение агента
    agent any

    // 7. Добавление параметров
    parameters {
        string(name: 'APP_VERSION', defaultValue: '1.0.0', description: 'Версия приложения')
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'stage', 'prod'], description: 'Окружение для развертывания')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Пропустить тесты')
    }

    // Глобальные переменные окружения
    environment {
        BUILD_ID = "${BUILD_NUMBER}"
    }

    stages {
        // 4. Этап: Сборка (Build)
        stage('Build') {
            steps {
                echo "🚀 Начало сборки версии ${params.APP_VERSION}"
                // 5. Шаги сборки (пример для Maven/Gradle или npm)
                sh 'echo "Установка зависимостей..."'
                sh 'echo "Компиляция кода..."'
                // Имитация команды сборки
                // sh 'mvn clean package' 
            }
        }

        // 4. Этап: Тестирование (Test)
        stage('Test') {
            // 6. Условие выполнения (если не пропущено параметром)
            when {
                expression { return params.SKIP_TESTS == false }
            }
            steps {
                echo "🧪 Запуск тестов..."
                // 8. Обработка ошибок внутри шага
                script {
                    try {
                        sh 'echo "Запуск юнит-тестов..."'
                        // sh 'mvn test'
                        // Имитация проверки кода возврата
                        def status = sh(script: 'echo "Tests passed"', returnStatus: true)
                        if (status != 0) {
                            error "Тесты не прошли!"
                        }
                    } catch (Exception e) {
                        echo "❌ Ошибка при тестировании: ${e.message}"
                        throw e // Пробрасываем ошибку дальше, чтобы пайплайн упал
                    }
                }
            }
        }

        // 4. Этап: Развертывание (Deploy)
        stage('Deploy') {
            // 6. Условие: только для ветки main и окружения prod
            when {
                branch 'main'
                expression { return params.DEPLOY_ENV == 'prod' }
            }
            steps {
                echo "🌍 Развертывание на ${params.DEPLOY_ENV}..."
                // Пример шага деплоя
                sh '''
                    echo "Подключение к серверу..."
                    echo "Копирование артефакта версии ${APP_VERSION}..."
                    echo "Перезапуск сервиса..."
                '''
            }
        }
    }

    // 8. Глобальная обработка ошибок и пост-действия
    post {
        always {
            echo "🏁 Пайплайн завершен."
            // Очистка рабочего пространства
            cleanWs()
        }
        success {
            echo "✅ Сборка успешна!"
            // Можно отправить уведомление в Slack/Telegram
        }
        failure {
            echo "❌ Сборка провалилась! Проверьте логи."
            // Отправка уведомления об ошибке
            // emailext subject: "Failed: ${env.JOB_NAME}", body: "Check console output", to: 'admin@example.com'
        }
        unstable {
            echo "⚠️ Сборка нестабильна (например, упали тесты)."
        }
    }
}

