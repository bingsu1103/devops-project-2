pipeline {
    agent { label 'individual-agent' }

    environment {
        // 1. Thay tên tài khoản Docker Hub thật của bạn vào đây (Viết thường hoàn toàn)
        DOCKER_ACCOUNT = 'bingsu1103'
        REGISTRY_HOST  = 'docker.io'
        
        // 2. Đường dẫn repo Manifest của bạn (Thay đổi account/repo tương ứng)
        MANIFEST_REPO  = 'github.com/bingsu1103/gitops-manifest-k8s.git'
        
        // Tạo Tag độc nhất cho mỗi lần build bằng chuỗi "build-[Số-thứ-tự-build]"
        IMAGE_TAG      = "build-${BUILD_NUMBER}"
    }

    stages {
        stage('Monorepo Build & Push Image') {
            steps {
                script {
                    // Định nghĩa danh sách file thay đổi
                    def changedFiles = []
                    
                    // Thử lấy danh sách file thay đổi, nếu lỗi (do build lần đầu) thì catch lại và log ra
                    try {
                        changedFiles = sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true).trim().split('\n')
                    } catch (Exception e) {
                        echo "⚠️ Không tìm thấy lịch sử commit trước (Có thể là build lần đầu). Tiến hành quét toàn bộ Monorepo!"
                    }
                    
                    // Danh sách toàn bộ các service trong Monorepo
                    def services = [
                        'media', 'product', 'order', 'inventory', 'payment', 'promotion', 
                        'rating', 'delivery', 'sampledata', 'recommendation',
                        'customer', 'location', 'cart', 'tax', 'search', 'webhook',
                        'backoffice-bff', 'storefront-bff', 'payment-paypal'
                    ]

                    // Đăng nhập vào Docker Hub một lần duy nhất trước khi vòng lặp chạy
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                    }

                    // Duyệt qua từng service
                    for (service in services) {
                        // NẾU build lần đầu (changedFiles rỗng) HOẶC có file thay đổi thuộc thư mục của service đó
                        if (changedFiles.isEmpty() || changedFiles.any { it.startsWith("${service}/") }) {
                            echo "🚀 Bắt đầu đóng gói và build image cho service: ${service}"
                            
                            // 1. Build package jar (Bỏ qua chạy test để build siêu tốc)
                            sh "mvn package -DskipTests -pl ${service} -am"
                            
                            // 2. Build và Push Docker Image lên Docker Hub
                            def fullImageName = "${REGISTRY_HOST}/${DOCKER_ACCOUNT}/${service}-service"
                            sh """
                                docker build -t ${fullImageName}:${IMAGE_TAG} -t ${fullImageName}:latest ./${service}
                                docker push ${fullImageName}:${IMAGE_TAG}
                                docker push ${fullImageName}:latest
                            """
                            
                            // Ghi nhận lại dịch vụ nào vừa được build để tí nữa cập nhật Manifest
                            env["BUILD_TRIGGERED_${service}"] = "true"
                        } else {
                            echo "⏭️ ${service} không có thay đổi, bỏ qua."
                        }
                    }
                }
            }
        }

        stage('Tự động cập nhật K8s Manifest (GitOps)') {
            steps {
                script {
                    def services = [
                        'media', 'product', 'order', 'inventory', 'payment', 'promotion', 
                        'rating', 'delivery', 'sampledata', 'recommendation',
                        'customer', 'location', 'cart', 'tax', 'search', 'webhook',
                        'backoffice-bff', 'storefront-bff', 'payment-paypal'
                    ]

                    // Kiểm tra xem có bất kỳ service nào được build ở stage trước không
                    def dynamicTriggers = services.findAll { env["BUILD_TRIGGERED_${it}"] == "true" }
                    
                    if (dynamicTriggers.isEmpty()) {
                        echo "No images were built. Skipping Manifest update."
                        return
                    }

                    // Dùng token Git để Jenkins có quyền clone và push lên repo Manifest
                    withCredentials([usernamePassword(credentialsId: 'git-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                        sh """
                            git config --global user.email "jenkins-bot@ci-cd.com"
                            git config --global user.name "Jenkins Automation Bot"
                            
                            # Xóa thư mục tạm nếu có và tiến hành clone repo manifest về
                            rm -rf temp-manifest
                            git clone https://${GIT_USER}:${GIT_TOKEN}@${MANIFEST_REPO} temp-manifest
                            cd temp-manifest/environments/dev
                        """

                        // Chạy qua những thằng vừa được build để sửa tag trong file Kustomize/YAML
                        for (service in dynamicTriggers) {
                            echo "✍️ Đang ghi tag mới ${IMAGE_TAG} cho ${service}-service vào manifest..."
                            
                            // Sử dụng kustomize để set image tự động, đổi tên port name chuẩn hóa
                            sh """
                                cd temp-manifest/environments/dev
                                kustomize edit set image ${REGISTRY_HOST}/${DOCKER_ACCOUNT}/${service}-service=${REGISTRY_HOST}/${DOCKER_ACCOUNT}/${service}-service:${IMAGE_TAG}
                            """
                        }

                        // Commit và Push tất cả thay đổi lên repo K8s Manifest để ArgoCD nhận diện
                        sh """
                            cd temp-manifest
                            git add .
                            git commit -m "image(ci): tự động cập nhật tag cho các service: ${dynamicTriggers.join(', ')} sang [${IMAGE_TAG}]"
                            git push origin main
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Dọn dẹp Workspace..."
            cleanWs()
        }
    }
}