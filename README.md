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
