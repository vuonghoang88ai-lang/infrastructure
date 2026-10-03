pipeline {
    agent any

    parameters {
        choice(name: 'ENV_TARGET', choices: ['live', 'staging', 'testing'], description: 'Chọn môi trường để tự động cấp phát IP và máy chủ')
    }

    environment {
        TELEGRAM_TOKEN = "7834830282:AAGupEEZ4IYjfmO_FkNFFsmBVzd6F1JpxPg"
        TELEGRAM_CHAT_ID = "5094340711"
        VAULT_ADDR = "https://127.0.0.1:8200"
        VAULT_SKIP_VERIFY = "true"
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

        stage('💻 2. Tự động tìm IP & Cấp phát máy ảo (Ansible)') {
            steps {
                sh """
                ansible-galaxy install -r requirements.yml --force
                
                # Jenkins chỉ cần ném biến môi trường, Ansible sẽ tự suy luận ID và IP
                ansible-playbook 1_provision.yml -e "target_env=${params.ENV_TARGET}"
                """
            }
        }

        stage('🛡 3. Củng cố bảo mật hệ điều hành (Security)') {
            steps {
                sh """
                echo "⏳ Đang chờ 90 giây để máy ảo mới khởi động dịch vụ SSH..."
                sleep 90
                
                # Bỏ lệnh sed vì Ansible đã tự động cập nhật inventory.ini ở Stage 2
                ansible-playbook -i inventory.ini 2_security.yml -e "target_env=${params.ENV_TARGET}"
                """
            }
        }
    }

    post {
        success {
            script {
                def msg = "✅ [INFRA SUCCESS] Dựng máy ảo & Bảo mật tự động hoàn tất cho cụm: ${params.ENV_TARGET.toUpperCase()}"
                sh "curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage -d chat_id=${TELEGRAM_CHAT_ID} -d text=\"${msg}\""
            }
        }
        failure {
            script {
                def msg = "❌ [INFRA FAILED] Lỗi khi tạo máy ảo hạ tầng cụm: ${params.ENV_TARGET.toUpperCase()}!"
                sh "curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage -d chat_id=${TELEGRAM_CHAT_ID} -d text=\"${msg}\""
            }
        }
    }
}
