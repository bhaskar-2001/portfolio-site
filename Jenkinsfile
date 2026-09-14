pipeline {
    agent any

    environment {
        ARTIFACT_BUCKET = 'portfolio-pipeline-artifacts-sysopx-4821'
        ASG_NAME        = 'portfolio-asg'
    }

    stages {
        stage('Sync to S3') {
            steps {
                sh '''
                    aws s3 sync . s3://$ARTIFACT_BUCKET/site \
                        --exclude ".git/*" \
                        --exclude "Jenkinsfile" \
                        --exclude "appspec.yml" \
                        --exclude "scripts/*"
                '''
            }
        }

        stage('Deploy to fleet via SSM') {
            steps {
                sh '''
                    INSTANCE_IDS=$(aws autoscaling describe-auto-scaling-groups \
                        --auto-scaling-group-names $ASG_NAME \
                        --query "AutoScalingGroups[0].Instances[?LifecycleState=='InService'].InstanceId" \
                        --output text)

                    echo "Deploying to: $INSTANCE_IDS"

                    aws ssm send-command \
                        --instance-ids $INSTANCE_IDS \
                        --document-name "AWS-RunShellScript" \
                        --parameters commands=["aws s3 sync s3://$ARTIFACT_BUCKET/site /usr/share/nginx/html --delete","systemctl restart nginx"] \
                        --comment "Portfolio deploy from Jenkins build $BUILD_NUMBER"
                '''
            }
        }
    }
}
