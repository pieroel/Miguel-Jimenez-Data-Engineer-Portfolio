pipeline {
    agent any

    triggers {
        // Se dispara automáticamente cuando se abre o actualiza un Pull Request en GitHub
        pollSCM('')
    }

    environment {
        GOOGLE_APPLICATION_CREDENTIALS = credentials('gcp-service-account-key')
        TF_VAR_project_id = 'mi-proyecto-gcp'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validación de formato') {
            steps {
                sh 'terraform fmt -check -recursive'
            }
        }

        stage('Inicializar Terraform') {
            steps {
                sh 'terraform init'
            }
        }

        stage('Validar sintaxis') {
            steps {
                sh 'terraform validate'
            }
        }

        stage('Plan de infraestructura') {
            steps {
                sh 'terraform plan -out=tfplan'
            }
        }

        stage('Análisis de seguridad') {
            steps {
                sh 'checkov -d . --framework terraform'
            }
        }

        stage('Aplicar cambios') {
            when {
                branch 'main'
            }
            steps {
                sh 'terraform apply -auto-approve tfplan'
            }
        }
    }

    post {
        success {
            echo '✅ Despliegue completado exitosamente en GCP'
        }
        failure {
            echo '❌ El pipeline falló. Revisa los logs.'
        }
    }
}
