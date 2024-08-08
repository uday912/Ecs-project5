pipeline {
    agent any
    environment {
        registry = "654654178716.dkr.ecr.us-east-1.amazonaws.com/ecs-jenkins"
        region = "us-east-1"
        clusterName = "ecs-jenkins-cluster" 
        serviceName = "ecs-jenkins-service-new" 
        containerName = "ecs-jenkins-container" 
        taskFamily = "ecs-jenkins-task-family" 
        cpu = "256" 
        memory = "512" 
        executionRoleArn = "arn:aws:iam::123456789012:role/ecsTaskExecutionRole" // Your execution role ARN
        loadBalancerArn = "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-load-balancer/50dc6c495c0c9188" // Your ALB ARN
        targetGroupArn = "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-target-group/6d0ecf831eec9f09" // Your target group ARN
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/uday912/Ecs-project-3.git'
            }
        }

        stage('Building image') {
            steps {
                script {
                    dockerImage = docker.build registry
                }
            }
        }

        stage('Pushing to ECR') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws_cred', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        sh 'aws ecr get-login-password --region ${region} | docker login --username AWS --password-stdin ${registry}'
                        sh 'docker push ${registry}'
                    }
                }
            }
        }

        stage('Stop previous containers') {
            steps {
                sh 'docker ps -f name=myContainer -q | xargs --no-run-if-empty docker container stop'
                sh 'docker container ls -a -f name=myContainer -q | xargs -r docker container rm'
            }
        }

        stage('Docker Run') {
            steps {
                script {
                    sh 'docker run -d -p 80:80 --rm --name myContainer ${registry}:latest'
                }
            }
        }

        stage('Update ECS Task Definition') {
            steps {
                script {
                    // Register a new task definition revision for Fargate
                    sh """
                    aws ecs register-task-definition \
                        --family ${taskFamily} \
                        --requires-compatibilities FARGATE \
                        --network-mode awsvpc \
                        --cpu ${cpu} \
                        --memory ${memory} \
                        --execution-role-arn ${executionRoleArn} \
                        --container-definitions '[{
                            "name": "${containerName}",
                            "image": "${registry}:latest",
                            "essential": true,
                            "portMappings": [{
                                "containerPort": 80,
                                "hostPort": 80
                            }],
                            "memory": 512,
                            
                        }]' \
                        --region ${region}
                    """
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    // Get the new task definition revision
                    def newTaskDefArn = sh(
                        script: "aws ecs describe-task-definition --task-definition ${taskFamily} --region ${region} | jq -r '.taskDefinition.taskDefinitionArn'",
                        returnStdout: true
                    ).trim()

                    // Update ECS service to use the new task definition
                    sh """
                    aws ecs update-service --cluster ${clusterName} --service ${serviceName} --task-definition ${newTaskDefArn} --desired-count 2 --region ${region} --force-new-deployment
                    """
                }
            }
        }

        stage('Configure Auto Scaling') {
            steps {
                script {
                    // Register ECS service with ALB
                    sh """
                    aws ecs update-service --cluster ${clusterName} --service ${serviceName} --load-balancers '[{
                        "targetGroupArn": "${targetGroupArn}",
                        "containerName": "${containerName}",
                        "containerPort": 80
                    }]' --region ${region}
                    """

                    // Configure ECS Service Auto Scaling
                    sh """
                    aws application-autoscaling register-scalable-target \
                        --service-namespace ecs \
                        --scalable-dimension ecs:service:DesiredCount \
                        --resource-id service/${clusterName}/${serviceName} \
                        --min-capacity 2 \
                        --max-capacity 4 \
                        --region ${region}
                    """

                    sh """
                    aws application-autoscaling put-scaling-policy \
                        --service-namespace ecs \
                        --scalable-dimension ecs:service:DesiredCount \
                        --resource-id service/${clusterName}/${serviceName} \
                        --policy-name ecs-scaling-policy \
                        --policy-type TargetTrackingScaling \
                        --target-tracking-scaling-policy-configuration '{
                            "TargetValue": 50.0,
                            "PredefinedMetricSpecification": {
                                "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
                            },
                            "ScaleOutCooldown": 60,
                            "ScaleInCooldown": 60
                        }' \
                        --region ${region}
                    """
                }
            }
        }
    }
}
