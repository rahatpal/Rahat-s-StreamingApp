ipeline {
    agent any

    stages {
        stage('Environment Check') {
            steps {
                sh '''
                    echo "========== CHECKING DOCKER AVAILABILITY =========="
                    if command -v docker >/dev/null 2>&1; then
                        echo "✅ Docker is available"
                        docker --version
                    else
                        echo "❌ Docker is NOT available on this system"
                        echo "Available commands:"
                        which docker || echo "No docker command found"
                    fi
                    
                    echo "========== PROJECT STRUCTURE =========="
                    ls -la
                    echo ""
                    
                    echo "========== BACKEND SERVICES =========="
                    if [ -d "./backend" ]; then
                        ls -la ./backend
                    else
                        echo "❌ Backend directory not found"
                    fi
                '''
            }
        }

        stage('Build Test') {
            steps {
                sh '''
                    echo "========== BUILD TEST =========="
                    if command -v docker >/dev/null 2>&1; then
                        echo "Docker available - attempting build test"
                        # Just check if we can run a simple docker command
                        docker info >/dev/null 2>&1 && echo "✅ Docker daemon is running" || echo "❌ Docker daemon not accessible"
                    else
                        echo "Skipping Docker build - Docker not available"
                        echo "This is expected on Herovired Jenkins environment"
                    fi
                '''
            }
        }

        stage('Alternative Build') {
            steps {
                sh '''
                    echo "========== ALTERNATIVE BUILD =========="
                    echo "Checking if we can build without Docker:"
                    if [ -f "./package.json" ]; then
                        echo "✅ package.json found - npm commands might work"
                        npm --version 2>/dev/null && echo "✅ npm available" || echo "❌ npm not available"
                    fi
                    
                    if [ -d "./frontend" ] && [ -f "./frontend/package.json" ]; then
                        echo "✅ Frontend directory with package.json found"
                    fi
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}