pipeline {
    agent any

    parameters {
        choice(name: 'ENV_TARGET', choices: ['live', 'staging', 'testing'], description: 'Chọn môi trường để cấp phát')
    }

    environment {
        // Cấu hình kết nối Vault (đã sửa thành HTTPS và bỏ qua xác thực SSL)
        VAULT_ADDR = 'https://127.0.0.1:8200'
        VAULT_SKIP_VERIFY = 'true'
        
        // Tùy thuộc vào ID credential của bạn trên Jenkins, tên ở đây có thể khác đôi chút (ví dụ 'vault-token')
        VAULT_TOKEN = credentials('vault-token') 
    }

    stages {
        stage('🔒 1. Mở két sắt Vault (Lấy mật khẩu)') {
            steps {
                script {
                    // SỬA LỖI: Thêm returnStdout: true và .trim() để lưu giá trị thực sự vào biến
                    env.VAULT_PROXMOX_PASS = sh(script: "vault kv get -field=api_password secret/proxmox", returnStdout: true).trim()
                    env.VAULT_CLOUDINIT_PASS = sh(script: "vault kv get -field=cloudinit_password secret/proxmox", returnStdout: true).trim()
                    env.VAULT_SSH_PUB_KEY = sh(script: "vault kv get -field=ssh_pub_key secret/proxmox", returnStdout: true).trim()
                }
            }
        }

        stage('💻 2. Tự động tìm IP & Cấp phát máy ảo (Ansible)') {
            steps {
                sh 'ansible-galaxy install -r requirements.yml --force'
                
                // Cập nhật tham số ép dùng khóa ED25519
                sh 'ansible-playbook 1_provision.yml -e "target_env=${params.ENV_TARGET}" -e "ansible_ssh_private_key_file=~/.ssh/id_ed25519"'
            }
        }

        stage('🛡 3. Củng cố bảo mật hệ điều hành (Security)') {
            steps {
                // Cập nhật tham số ép dùng khóa ED25519
                sh 'ansible-playbook -i inventory.ini 2_security.yml -e "target_env=${params.ENV_TARGET}" -e "ansible_ssh_private_key_file=~/.ssh/id_ed25519"'
            }
        }
    }

    post {
        failure {
            script {
                sh '''
                curl -s -X POST https://api.telegram.org/bot7834830282:AAGupEEZ4IYjfmO_FkNFFsmBVzd6F1JpxPg/sendMessage \
                -d chat_id=5094340711 \
                -d text="❌ [INFRA FAILED] Lỗi khi tạo máy ảo hạ tầng cụm: ${params.ENV_TARGET.toUpperCase()}!"
                '''
            }
        }
        success {
            script {
                sh '''
                curl -s -X POST https://api.telegram.org/bot7834830282:AAGupEEZ4IYjfmO_FkNFFsmBVzd6F1JpxPg/sendMessage \
                -d chat_id=5094340711 \
                -d text="✅ [INFRA SUCCESS] Cấp phát thành công máy ảo hạ tầng cụm: ${params.ENV_TARGET.toUpperCase()}!"
                '''
            }
        }
    }
}
