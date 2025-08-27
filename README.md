# TP CI/CD - Jenkins

Ce projet présente l'installation et la configuration de Jenkins pour un pipeline CI/CD.

## 🚀 Installation de Jenkins

### 1. Installation des dépendances

Installez Java 17 JRE et wget :

```bash
sudo apt update
sudo apt install -y openjdk-17-jre wget
```

### 2. Téléchargement du package Jenkins

Récupérez la dernière version stable de Jenkins :

```bash
wget https://pkg.jenkins.io/debian-stable/binary/jenkins_2.462.1_all.deb
```

### 3. Installation de Jenkins

Installez le package téléchargé :

```bash
sudo apt install -y ./jenkins_2.462.1_all.deb
```

### 4. Démarrage du service

Activez et démarrez le service Jenkins :

```bash
sudo systemctl enable jenkins

sudo systemctl start jenkins
```

## 🔧 Configuration initiale

Une fois Jenkins installé, accédez à l'interface web :

1. Ouvrez votre navigateur et allez sur : `http://localhost:8080`
2. Récupérez le mot de passe administrateur initial :

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

3. Suivez les étapes de configuration dans l'interface web

## 📁 Structure du projet

```
├── src/
│   ├── app.js          
│   ├── index.html      
│   ├── style.css       
│   └── utils.js        
├── tests/
│   └── app.test.js     
├── Jenkinsfile         
├── package.json        
└── README.md          
```

## Exercices

### Exercice 1

#### Problème n°1
J'ai eu un problème concernant la version de nodeJS, donc j'ai du rajouter quelques lignes dans le Jenkinsfile pour spécifier le tool utilisé pour nodejs :

```
tools {
        nodejs "NodeJS-18"
    }
```

Il m'a fallu donc créér un tool pour nodeJs en spécifiant la version 18 à utiliser...
Dans manage jenkins > tools > nodeJS (avec le plugin NodeJS installé)

#### Problème n°2

J'ai eu une erreur :

```
java.lang.NoSuchMethodError: No such DSL method 'publishTestResults' found among steps [...]
```

J'ai donc utilisé Junit et remplacer au niveau du post dans "Run tests" :

```
post {
    always {
        junit 'test-results/junit.xml'
    }
}
```

Et pour générer ce fichier junit.xml, j'ai installé jest-junit :

```
npm install --save-dev jest-junit
```

Et ensuite j'ai rajouté des lignes dans le fichier package.json pour spécifier ou creer le report:

```
"jest": {
    "reporters": [
      "default",
      [
        "jest-junit",
        {
          "outputDirectory": "test-results",
          "outputName": "junit.xml"
        }
      ]
    ]
  }
```

### Exercice 2

J'ai ajouté une branch develop, j'ai ensuite ajouté une fonction (pas utilisée) dans le fichier app.js, mais après avoir commit et push, je me suis apperçu que le déploiement staging ne fonctionnait pas.
Je me suis également apperçu que le développement production ne fonctionnait pas.

J'ai donc créé un nouvel item : pipeline multibranch.
Cela m'a donc permis d'avoir une branch pour la production -> main et une branche pour staging -> develop.

Je n'ai rien eu à modifier dans le fichier Jenkinsfile car il y avait dejà la condition "when" avec "branch develop" ou "branch main".

### Exercice 3

Toujours sur develop,

J'ai déplacé ma fonction "squareNumber" dans "utils.js" et ensuite j'ai créé un test qui fail pour vérifier que le stage "Tests" plante, donc le reste est skip.

J'ai ensuite résolu ce test, qui vérifie qu'un nombre est mis au carré ".

### Exercice 4

1. Ajout de notifications

Lorsque l'application est bien déployé, je veux avoir un message envoyé sur Discord. Donc j'ai créé un serveur discord, et un webhook sur ce serveur discord. Ensuite, j'ai complété le "success" dans le jenkinsfile :

```
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
```

Ici j'utilise "withCredentials" pour utiliser les secrets définis dans jenkins. Et j'utilise 'discord-webhook-url' pour ensuite executer une commande curl et poster mon message sur mon serveur discord.
J'utilise les secrets sur jenkins pour éviter de push directement l'url de mon webhook, pour éviter que tout le monde puisse avoir ce lien et l'utiliser pour poster des messages.


Je fais ainsi la même chose quand il y a un fail :

```
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
```

Voici les messages qui sont envoyés dans le build réussi ou échoue :

![alt text](discord_webhook.png)

2. Coverage

Pour le coverage, j'ai installé le plugin "Coverage" sur jenkins.
Ensuite j'ai ajouté la ligne suivante dans la partie scripts dans le package.json:
```
"test:ci": "jest --coverage --coverageReporters=cobertura",
```

Cette ligne va permettre de générer un rapport cobertura en .xml quand on va faire la commande npm run test:ci, en plus d'executer les tests.

J'ai donc modifié mon stage "Tests" comme ceci :
```
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
```

Ce stage va donc lancer la génération du rapport .xml

Puis j'ai créé un nouveau stage "Coverage" qui va venir lire le rapport .xml cobertura et ainsi générer le coverage sur jenkins.

```
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
```

J'ai défini un seuil de 80% pour les lignes et 70% sur la branche pour que le build soit validé.

Ensuite dans le build status on peut voir ceci :

![alt text](coverage_report.png)

![alt text](code_coverage_trend.png)

3. Archivage des artifacts



