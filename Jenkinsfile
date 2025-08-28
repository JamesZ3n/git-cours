pipeline {
    agent any

    tools {
        nodejs "NodeJS-18"
    }
    
    environment {
        NODE_VERSION = '18'
        APP_NAME = 'mon-app-js'
        DEPLOY_DIR = '/var/www/html/mon-app'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Récupération du code source...'
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo 'Installation des dépendances Node.js...'
                sh '''
                    node --version
                    npm --version
                    npm ci
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                echo 'Exécution des tests...'
                sh 'npm run test:ci'
            }
            post {
                always {
                    junit 'test-results/junit.xml'
                }
            }
        }

        stage('Coverage') {
    steps {
        echo 'Analyse de la couverture...'
        recordCoverage(
            tools: [[
                parser: 'COBERTURA',
                pattern: 'coverage/cobertura-coverage.xml',
            ]],
            sourceCodeRetention: 'EVERY_BUILD',
            qualityGates: [
                [threshold: 80.0, metric: 'LINE'],
                [threshold: 70.0, metric: 'BRANCH']
            ]
        )
    }
}




        
        stage('Code Quality Check') {
            steps {
                echo 'Vérification de la qualité du code...'
                sh '''
                    echo "Vérification de la syntaxe JavaScript..."
                    find src -name "*.js" -exec node -c {} \\;
                    echo "Vérification terminée"
                '''
            }
        }
        
        stage('Build') {
            steps {
                echo 'Construction de l\'application...'
                sh '''
                    npm run build
                    ls -la dist/
                '''
            }
        }

        stage('Archive Artifacts') {
            steps {
                echo 'Archivage des artifacts...'
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
            }
        }
        
        stage('Security Scan') {
            steps {
                echo 'Analyse de sécurité...'
                sh '''
                    echo "Vérification des dépendances..."
                    npm audit --audit-level=high
                '''
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Déploiement vers l\'environnement de staging...'
                sh '''
                    echo "Sauvegarde de la version précédente..."
                    if [ -d "${DEPLOY_DIR}" ]; then
                        cp -r ${DEPLOY_DIR} ${DEPLOY_DIR}_develop_backup_$(date +%Y%m%d_%H%M%S)
                    fi
                    
                    echo "Déploiement de la nouvelle version..."
                    mkdir -p ${DEPLOY_DIR}/develop
                    cp -r dist/* ${DEPLOY_DIR}/develop/
                    
                    echo "Vérification du déploiement..."
                    ls -la ${DEPLOY_DIR}
                '''
            }
        }

        stage('Deploy to Production') {
            
            when {
                branch 'main'
            }
            steps {
                echo 'Déploiement vers la production...'
                sh '''
                    echo "Sauvegarde de la version précédente..."
                    if [ -d "${DEPLOY_DIR}" ]; then
                        cp -r ${DEPLOY_DIR} ${DEPLOY_DIR}_main_backup_$(date +%Y%m%d_%H%M%S)
                    fi
                    
                    echo "Déploiement de la nouvelle version..."
                    mkdir -p ${DEPLOY_DIR}/main
                    cp -r dist/* ${DEPLOY_DIR}/main/
                    
                    echo "Vérification du déploiement..."
                    ls -la ${DEPLOY_DIR}
                '''
            }
        }
        
        stage('Health Check') {
            steps {
                echo 'Vérification de santé de l\'application...'
                script {
                    // Choisir l'URL en fonction de la branche
                    def url = "http://localhost/mon-app/"
                    if (env.BRANCH_NAME == "develop") {
                        url += "develop"
                    } else if (env.BRANCH_NAME == "main") {
                        url += "main"
                    }

                    echo "🔎 Health check sur: ${url}"

                    // Récupérer le code HTTP
                    def status = sh(
                        script: "curl -L -o /dev/null -s -w '%{http_code}' ${url}",
                        returnStdout: true
                    ).trim()

                    if (status == '200') {
                        echo "✅ Application OK (HTTP ${status})"
                    } else {
                        echo "❌ Application KO (HTTP ${status})"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Nettoyage des ressources temporaires...'
            sh '''
                rm -rf node_modules/.cache
                rm -rf staging
            '''
        }
        success {
            echo 'Pipeline exécuté avec succès!'
            withCredentials([string(credentialsId: 'discord-webhook-url', variable: 'DISCORD_WEBHOOK')]) {
                sh(script: """
                    curl -H "Content-Type: application/json" \
                        -X POST \
                        -d '{"content":"✅ Build Success: ${JOB_NAME} #${BUILD_NUMBER} (${BRANCH_NAME})"}' \
                        "\$DISCORD_WEBHOOK"
                """)
            }
        }
        failure {
            echo 'Le pipeline a échoué!'
            withCredentials([string(credentialsId: 'discord-webhook-url', variable: 'DISCORD_WEBHOOK')]) {
                sh(script: """
                    curl -H "Content-Type: application/json" \
                        -X POST \
                        -d '{"content":"❌ Build Failed: ${JOB_NAME} #${BUILD_NUMBER} (${BRANCH_NAME})"}' \
                        "\$DISCORD_WEBHOOK"
                """)
            }
        }
        unstable {
            echo 'Build instable - des avertissements ont été détectés'
        }
    }
}