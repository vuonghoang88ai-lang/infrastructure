pipeline {
    agent any

    parameters {
        string(name: 'VM_ID', defaultValue: '303', description: 'ID cho máy ảo Proxmox')
        string(name: 'VM_NAME', defaultValue: 'Web-Live-Auto', description: 'Tên máy ảo')
        string(name: 'VM_IP', defaultValue: '192.168.2.89', description: 'IP tĩnh')
        choice(name: 'ENV_TARGET', choices: ['live', 'staging', 'testing'], description: 'Môi trường để chạy Security Hardening')
    }

    environment {
        TELEGRAM_TOKEN = "7834830282:AAGupEEZ4IYjfmO_FkNFFsmBVzd6F1JpxPg"
        TELEGRAM_CHAT_ID = "5094340711"
        VAULT_ADDR = "http://127.0.0.1:8200"
        VAULT_TOKEN = credentials('vault-root-token')
    }

    stages {
        stage('🔒 1. Mở két sắt Vault (Lấy mật khẩu)') {
            steps {
                script {
                    env.VAULT_PROXMOX_PASS = sh(script: "VAULT_TOKEN=${env.VAULT_TOKEN} vault kv get -field=api_password secret/proxmox", returnStdout: true).trim()
                    env.VAULT_CLOUDINIT_PASS = sh(script: "VAULT_TOKEN=${env.VAULT_TOKEN} vault kv get -field=cloudinit_password secret/proxmox", returnStdout: true).trim()
                    env.VAULT_SSH_PUB_KEY = sh(script: "VAULT_TOKEN=${env.VAULT_TOKEN} vault kv get -field=ssh_pub_key secret/proxmox", returnStdout: true).trim()
                }
            }
        }

        stage('💻 2. Tự động cấp phát máy ảo (Proxmox)') {
            steps {
                sh """
                ansible-galaxy install -r requirements.yml --force
                ansible-playbook 1_provision.yml \
                  -e "new_vm_id=${params.VM_ID}" \
                  -e "new_vm_name=${params.VM_NAME}" \
                  -e "new_vm_ip=${params.VM_IP}" \
                  -e "new_vm_gw=192.168.2.1"
                """
            }
        }

        stage('🛡 3. Củng cố bảo mật hệ điều hành (Security)') {
            steps {
                sh """
                echo "⏳ Đang chờ 45 giây để máy ảo mới (${params.VM_IP}) khởi động dịch vụ SSH..."
                sleep 45
                
                echo "Tự động chèn IP ${params.VM_IP} vào nhóm [${params.ENV_TARGET}] để chạy bảo mật..."
                sed -i "/\\[${params.ENV_TARGET}\\]/a ${params.VM_IP}" inventory.ini
                
                ansible-playbook -i inventory.ini 2_security.yml -e "target_env=${params.ENV_TARGET}"
                """
            }
        }
    }

    post {
        success {
            script {
                def msg = "✅ [INFRA SUCCESS] Dựng máy ảo & Bảo mật thành công: ${params.VM_NAME} (IP: ${params.VM_IP})"
                sh "curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage -d chat_id=${TELEGRAM_CHAT_ID} -d text=\"${msg}\""
            }
        }
        failure {
            script {
                def msg = "❌ [INFRA FAILED] Lỗi khi tạo máy ảo hạ tầng ${params.VM_NAME}!"
                sh "curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage -d chat_id=${TELEGRAM_CHAT_ID} -d text=\"${msg}\""
            }
        }
    }
}
