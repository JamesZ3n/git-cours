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
                sh 'npm test --coverage'
            }
            post {
                always {
                    junit 'test-results/junit.xml'
                }
            }
        }

        stage('Coverage') {
            steps {
                recordCoverage(
                    tools: [
                        [type: 'Cobertura', pattern: 'coverage/cobertura-coverage.xml']
                    ],
                    failOnError: true
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
                    echo "Déploiement staging simulé"
                    mkdir -p staging
                    cp -r dist/* staging/
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
                        cp -r ${DEPLOY_DIR} ${DEPLOY_DIR}_backup_$(date +%Y%m%d_%H%M%S)
                    fi
                    
                    echo "Déploiement de la nouvelle version..."
                    mkdir -p ${DEPLOY_DIR}
                    cp -r dist/* ${DEPLOY_DIR}/
                    
                    echo "Vérification du déploiement..."
                    ls -la ${DEPLOY_DIR}
                '''
            }
        }
        
        stage('Health Check') {
            steps {
                echo 'Vérification de santé de l\'application...'
                script {
                    try {
                        sh '''
                            echo "Test de connectivité..."
                            # Simulation d'un health check
                            echo "Application déployée avec succès"
                        '''
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo "Warning: Health check failed: ${e.getMessage()}"
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