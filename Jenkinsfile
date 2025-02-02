def TF_OPERATION = 'Create'
def TF_WORKSPACE = 'default'
def TF_VM_COUNT  = '1'

pipeline {
    agent {
        label "ansible && linux"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                sh "git clean -xdff"
            }
        }

        stage('Install Ansible') {
            steps {
                sh "virtualenv --system-site-packages -p python3 ${WORKSPACE}/ansible-tf-azure"

                withPythonEnv("${WORKSPACE}/ansible-tf-azure/") {
                    sh "pip install --upgrade pip"
                    sh "pip install -r requirements.txt"
                }
            }
        }

        stage('Install Terraform') {
            steps {
                sh "curl -o ${WORKSPACE}/terraform.zip https://releases.hashicorp.com/terraform/0.12.12/terraform_0.12.12_linux_amd64.zip"
                sh "unzip -o -d ${WORKSPACE}/ansible-tf-azure/bin/ ${WORKSPACE}/terraform.zip"
                sh "rm -f ${WORKSPACE}/terraform.zip"
                withCredentials([string(credentialsId: 'ARM_CLIENT_ID', variable: 'ARM_CLIENT_ID'), string(credentialsId: 'ARM_CLIENT_SECRET', variable: 'ARM_CLIENT_SECRET'), string(credentialsId: 'ARM_SUBSCRIPTION_ID', variable: 'ARM_SUBSCRIPTION_ID'), string(credentialsId: 'ARM_TENANT_ID', variable: 'ARM_TENANT_ID')]) {
                    sh "${WORKSPACE}/ansible-tf-azure/bin/terraform init -no-color"
                }
            }
        }

        stage('Select Terraform Operation') {
            steps {
                script {
                    TF_OPERATION = input (
                        ok: 'Next',
                        message: 'Please Select Terraform Operation ',
                        parameters: [
                            choice (
                                name: 'TF_OPERATION',
                                choices: ['Create', 'Delete']
                            )
                        ]
                    )
                }
                echo "Terraform Operation: ${TF_OPERATION}"
            }
        }

        stage('Input Terraform Workspace') {
            steps {
                script {
                    TF_WORKSPACE = input (
                        message: 'Please Input Terraform Workspace ',
                        parameters: [
                            string (
                                name: 'TF_WORKSPACE',
                                defaultValue: 'default',
                                trim: true
                            )
                        ]
                    )
                }
                echo "Terraform Workspace: ${TF_WORKSPACE}"
            }
        }

        stage('Terraform Destroy') {
            when {
                expression {
                    TF_OPERATION == 'Delete'
                }
            }
            steps {
                withCredentials([string(credentialsId: 'ARM_CLIENT_ID', variable: 'ARM_CLIENT_ID'), string(credentialsId: 'ARM_CLIENT_SECRET', variable: 'ARM_CLIENT_SECRET'), string(credentialsId: 'ARM_SUBSCRIPTION_ID', variable: 'ARM_SUBSCRIPTION_ID'), string(credentialsId: 'ARM_TENANT_ID', variable: 'ARM_TENANT_ID')]) {
                    sh "${WORKSPACE}/ansible-tf-azure/bin/terraform workspace select ${TF_WORKSPACE}"
                    sh "${WORKSPACE}/ansible-tf-azure/bin/terraform plan -input=false -no-color"
                    sh "${WORKSPACE}/ansible-tf-azure/bin/terraform destroy -input=false -no-color -auto-approve"
                }
            }
        }

        stage('Input VM Count') {
            when {
                expression {
                    TF_OPERATION == 'Create'
                }
            }
            steps {
                script {
                    TF_VM_COUNT = input (
                        message: 'Please Input VM Count ',
                        parameters: [
                            string (
                                name: 'TF_VM_COUNT',
                                defaultValue: '1',
                                trim: true
                            )
                        ]
                    )
                }
                echo "Terraform VM Count: ${TF_VM_COUNT}"
            }
        }

        stage('Terraform Apply') {
            when {
                expression {
                    TF_OPERATION == 'Create'
                }
            }
            steps {
                withCredentials([string(credentialsId: 'ARM_CLIENT_ID', variable: 'ARM_CLIENT_ID'), string(credentialsId: 'ARM_CLIENT_SECRET', variable: 'ARM_CLIENT_SECRET'), string(credentialsId: 'ARM_SUBSCRIPTION_ID', variable: 'ARM_SUBSCRIPTION_ID'), string(credentialsId: 'ARM_TENANT_ID', variable: 'ARM_TENANT_ID')]) {
                    sh "${WORKSPACE}/ansible-tf-azure/bin/terraform workspace new ${TF_WORKSPACE} || ${WORKSPACE}/ansible-tf-azure/bin/terraform workspace select ${TF_WORKSPACE}"
                    sh "${WORKSPACE}/ansible-tf-azure/bin/terraform plan -input=false -no-color -out t_plan -var count_of_VMs=${TF_VM_COUNT}"
                    sh "${WORKSPACE}/ansible-tf-azure/bin/terraform apply -input=false -no-color t_plan"
                }
            }
        }

        stage('Check Connection Target') {
            when {
                expression {
                    TF_OPERATION == 'Create'
                }
            }
            steps {
                withPythonEnv("${WORKSPACE}/ansible-tf-azure/") {
                    ansiblePlaybook colorized: true,
                                    installation: 'ansible',
                                    credentialsId: 'vignoadmin',
                                    playbook: 'winrm_ping.yml',
                                    extras: '-e target_hosts=all'
                }
            }
        }

        stage('Deploying WildFly') {
            when {
                expression {
                    TF_OPERATION == 'Create'
                }
            }
            steps {
                withPythonEnv("${WORKSPACE}/ansible-tf-azure/") {
                    ansiblePlaybook colorized: true,
                                    installation: 'ansible',
                                    credentialsId: 'vignoadmin',
                                    playbook: 'deploy_wildfly.yml',
                                    extras: '-e target_hosts=all'
                }
            }
        }

        stage('Check Installation Result') {
            when {
                expression {
                    TF_OPERATION == 'Create'
                }
            }
            steps {
                withPythonEnv("${WORKSPACE}/ansible-tf-azure/") {
                    ansiblePlaybook colorized: true,
                                    installation: 'ansible',
                                    credentialsId: 'vignoadmin',
                                    playbook: 'winrm_url.yml',
                                    extras: '-e target_hosts=all'
                }
            }
        }
    }
}
