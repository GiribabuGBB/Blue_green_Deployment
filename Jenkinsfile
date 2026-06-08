pipeline {
    agent any

    // ============================================================
    // ENVIRONMENT VARIABLES
    // These are global variables used across all stages.
    // Change these values if your AWS resources change.
    // ============================================================
    environment {
        AWS_REGION        = "ap-south-1"                  // AWS region where all resources are deployed
        ECR_REPO          = "356627769740.dkr.ecr.ap-south-1.amazonaws.com/myapp" // ECR registry URL where Docker images are stored
        CLUSTER           = "Blue_Green_Cluster"          // ECS cluster name that runs our containers
        SERVICE           = "Blue_Green-service-b7ifaw2f" // ECS service name that manages our tasks
        TARGET_GROUP_GREEN = "arn:aws:elasticloadbalancing:ap-south-1:356627769740:targetgroup/BlueDeployment/6931a5cf0af3eaec" // ALB target group ARN (active deployment)
        LISTENER_ARN      = "arn:aws:elasticloadbalancing:ap-south-1:356627769740:listener/app/ALBLoadbalancerBluegreen/c4e8f0d58f601261/f09c0f6b6d27d58c" // ALB listener ARN that routes user traffic
        ALB_DNS           = "ALBLoadbalancerBluegreen-963860051.ap-south-1.elb.amazonaws.com" // Public DNS of the load balancer to access the app
    }

    stages {

        // ============================================================
        // STAGE 1: CHECKOUT
        // Purpose : Pull the latest source code from GitHub
        // What happens : Jenkins clones/fetches the 'main' branch
        //                from the GitHub repository onto the Jenkins
        //                workspace on the EC2 instance.
        // Why needed : Every build must start with fresh, latest code
        //              so we always deploy what's in GitHub.
        // ============================================================
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/GiribabuGBB/Blue_Green_Deployment.git'
            }
        }

        // ============================================================
        // STAGE 2: BUILD DOCKER IMAGE
        // Purpose : Package the application into a Docker image
        // What happens : Reads the Dockerfile in the repo, builds an
        //                image, and tags it with the Jenkins build number
        //                (e.g. myapp:14) so every build has a unique tag.
        // Why needed : ECS runs containers, not raw code. Docker packages
        //              the app + its dependencies into a portable image.
        // BUILD_NUMBER : Jenkins auto-increments this (1, 2, 3...)
        //                so each build produces a uniquely tagged image.
        // ============================================================
        stage('Build Docker Image') {
            steps {
                sh '''
                # Build the Docker image from the Dockerfile in the repo
                docker build -t myapp:${BUILD_NUMBER} .

                # Tag the image with the full ECR repository URL
                # so Docker knows where to push it
                docker tag myapp:${BUILD_NUMBER} $ECR_REPO:${BUILD_NUMBER}
                '''
            }
        }

        // ============================================================
        // STAGE 3: LOGIN TO ECR
        // Purpose : Authenticate Docker with AWS ECR
        // What happens : AWS CLI generates a temporary login token,
        //                which is piped into Docker login so Docker
        //                can push/pull images from our private ECR repo.
        // Why needed : ECR is private — Docker needs authentication
        //              before it can push images to it.
        // Credentials : Uses 'aws-creds' stored in Jenkins credentials
        //               (AWS Access Key + Secret Key).
        // ============================================================
        stage('Login to ECR') {
            steps {
                withCredentials([aws(
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        # Get a temporary ECR auth token from AWS and
                        # pipe it directly into docker login
                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin $ECR_REPO
                    '''
                }
            }
        }

        // ============================================================
        // STAGE 4: PUSH TO ECR
        // Purpose : Upload the newly built Docker image to AWS ECR
        // What happens : Docker pushes the image tagged with the build
        //                number to the ECR repository.
        // Why needed : ECS pulls images from ECR when starting tasks.
        //              The image must be in ECR before ECS can use it.
        // Result : Image is now stored as:
        //          356627769740.dkr.ecr.ap-south-1.amazonaws.com/myapp:14
        // ============================================================
        stage('Push to ECR') {
            steps {
                withCredentials([aws(
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        # Push the tagged image to ECR
                        # ECS will pull this image when deploying the new task
                        docker push $ECR_REPO:${BUILD_NUMBER}
                    '''
                }
            }
        }

        // ============================================================
        // STAGE 5: UPDATE TASK DEFINITION
        // Purpose : Tell ECS to use the new Docker image
        // What happens :
        //   Step 1 - Save current task definition ARN for rollback
        //   Step 2 - Download current task definition JSON from AWS
        //   Step 3 - Update the image tag to the new build number
        //   Step 4 - Remove read-only fields AWS won't accept
        //   Step 5 - Register a new task definition revision in ECS
        //   Step 6 - Update the ECS service to use the new revision
        //
        // Why needed : ECS doesn't automatically know about the new image.
        //              We must create a new task definition revision that
        //              points to the new image tag, then tell the service
        //              to use it.
        //
        // Example:
        //   Before: Blue_Green:6  → image: myapp:v3
        //   After : Blue_Green:7  → image: myapp:14
        //
        // Rollback prep : Previous task def ARN is saved to
        //                 /tmp/previous_task_def.txt so if anything
        //                 fails later, we can revert to it.
        // ============================================================
        stage('Update Task Definition') {
            steps {
                withCredentials([aws(
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh """
                    # Step 1: Get the currently running task definition ARN
                    # and save it to a file for rollback use later
                    PREV_TASK_DEF=\$(aws ecs describe-services \
                      --cluster ${CLUSTER} \
                      --services ${SERVICE} \
                      --region ${AWS_REGION} \
                      --query 'services[0].taskDefinition' \
                      --output text)

                    echo "Previous task def (rollback target): \$PREV_TASK_DEF"

                    # Save to file so the rollback block can read it if needed
                    echo "\$PREV_TASK_DEF" > /tmp/previous_task_def.txt

                    # Step 2: Download the full task definition JSON from AWS
                    aws ecs describe-task-definition \
                      --task-definition \$PREV_TASK_DEF \
                      --region ${AWS_REGION} \
                      --query 'taskDefinition' > /tmp/taskdef.json

                    # Step 3 & 4: Use Python to update the image tag
                    # and strip out read-only fields AWS won't accept
                    # on re-registration (like taskDefinitionArn, revision etc.)
                    python3 << 'PYEOF'
import json

# Load the current task definition
with open('/tmp/taskdef.json') as f:
    d = json.load(f)

# Update the container image to the new build number
for c in d.get('containerDefinitions', []):
    if '${ECR_REPO}' in c.get('image', ''):
        c['image'] = '${ECR_REPO}:${BUILD_NUMBER}'
        print(f"Updated image to: {c['image']}")

# Remove fields that AWS does not allow when registering a new task definition
for k in ['taskDefinitionArn','revision','status','requiresAttributes',
          'compatibilities','registeredAt','registeredBy']:
    d.pop(k, None)

# Save the cleaned task definition to a new file
with open('/tmp/taskdef_new.json', 'w') as f:
    json.dump(d, f, indent=2)

print("Task definition updated successfully")
PYEOF

                    # Step 5: Register the updated task definition as a new revision
                    # AWS assigns it the next revision number (e.g. Blue_Green:7)
                    NEW_TASK_DEF=\$(aws ecs register-task-definition \
                      --region ${AWS_REGION} \
                      --cli-input-json file:///tmp/taskdef_new.json \
                      --query 'taskDefinition.taskDefinitionArn' \
                      --output text)

                    echo "New task def registered: \$NEW_TASK_DEF"

                    # Step 6: Update the ECS service to use the new task definition
                    # --force-new-deployment stops the old task and starts a new one
                    aws ecs update-service \
                      --cluster ${CLUSTER} \
                      --service ${SERVICE} \
                      --task-definition \$NEW_TASK_DEF \
                      --force-new-deployment \
                      --region ${AWS_REGION}
                    """
                }
            }
        }

        // ============================================================
        // STAGE 6: HEALTH CHECK
        // Purpose : Verify the new ECS task is running before switching
        //           traffic to it
        // What happens :
        //   Step 1 - Wait 120 seconds for ECS to pull the new image,
        //            start the container, and pass internal health checks
        //   Step 2 - Query ECS for running vs desired task count
        //   Step 3 - If running != desired, exit with code 1 (FAIL)
        //            This triggers automatic rollback via post { failure }
        //
        // Why needed : Without this check, we might switch traffic to
        //              a version that hasn't started yet or is crashing.
        //
        // On failure : Pipeline stops here and rollback is triggered.
        //              The ALB is never switched to the broken version.
        // ============================================================
        stage('Health Check') {
            steps {
                withCredentials([aws(
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh """
                    echo "Waiting 120s for ECS to pull image and start new task..."
                    sleep 120

                    # Check how many tasks are currently running vs desired
                    RUNNING=\$(aws ecs describe-services \
                      --cluster ${CLUSTER} \
                      --services ${SERVICE} \
                      --region ${AWS_REGION} \
                      --query 'services[0].runningCount' \
                      --output text)

                    DESIRED=\$(aws ecs describe-services \
                      --cluster ${CLUSTER} \
                      --services ${SERVICE} \
                      --region ${AWS_REGION} \
                      --query 'services[0].desiredCount' \
                      --output text)

                    echo "Running: \$RUNNING / Desired: \$DESIRED"

                    # If task count doesn't match, deployment failed
                    # exit 1 triggers the post { failure } rollback block
                    if [ "\$RUNNING" != "\$DESIRED" ]; then
                        echo "HEALTH CHECK FAILED: Running \$RUNNING but desired \$DESIRED"
                        exit 1
                    fi

                    echo "Health check passed: \$RUNNING/\$DESIRED tasks running"
                    """
                }
            }
        }

        // ============================================================
        // STAGE 7: SWITCH TRAFFIC
        // Purpose : Point the ALB listener to the new deployment
        // What happens : Updates the ALB listener's default action to
        //                forward traffic to the active target group
        //                where the new ECS task is registered.
        // Why needed : This is the actual "blue-green switch".
        //              Before this step, users still see the old version.
        //              After this step, all traffic goes to the new version.
        // Zero downtime : ALB switches instantly with no dropped connections.
        // ============================================================
        stage('Switch Traffic') {
            steps {
                withCredentials([aws(
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh """
                        # Update the ALB listener to forward traffic to
                        # the target group where our new task is running
                        aws elbv2 modify-listener \
                        --listener-arn ${LISTENER_ARN} \
                        --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_GREEN} \
                        --region ${AWS_REGION}

                        echo "Traffic switched successfully to new deployment"
                    """
                }
            }
        }

        // ============================================================
        // STAGE 8: DEPLOYMENT INFO
        // Purpose : Print a clear summary at the end of a successful build
        // What happens : Echoes build details and the app URL so you
        //                can quickly verify the deployment by clicking
        //                the link in the Jenkins console output.
        // ============================================================
        stage('Deployment Info') {
            steps {
                sh """
                echo "============================================"
                echo "  DEPLOYMENT SUCCESSFUL"
                echo "============================================"
                echo "  Build Number : ${BUILD_NUMBER}"
                echo "  Image        : ${ECR_REPO}:${BUILD_NUMBER}"
                echo "  Cluster      : ${CLUSTER}"
                echo "  Service      : ${SERVICE}"
                echo "--------------------------------------------"
                echo "  ACCESS YOUR APP:"
                echo "  http://${ALB_DNS}:3000"
                echo "============================================"
                """
            }
        }
    }

    // ============================================================
    // POST ACTIONS
    // These run AFTER all stages complete, regardless of outcome.
    //
    // post { failure } — runs only if ANY stage above failed
    //   Purpose : Automatic rollback to the previous working version
    //   What happens :
    //     Step 1 - Reads the previous task definition ARN saved in
    //              Stage 5 from /tmp/previous_task_def.txt
    //     Step 2 - Reverts the ECS service to the previous task def
    //              (which used the last known good Docker image)
    //     Step 3 - Waits 60s for the old task to start
    //     Step 4 - Switches ALB listener back to the previous version
    //
    // post { success } — runs only if all stages passed
    //   Purpose : Print a final confirmation message
    //
    // Why rollback matters : If the new image crashes, has a bug, or
    //   fails the health check, users should never see a broken app.
    //   Rollback reverts everything automatically with no manual steps.
    // ============================================================
    post {
        failure {
            withCredentials([aws(
                credentialsId: 'aws-creds',
                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
            )]) {
                sh """
                echo "============================================"
                echo "  DEPLOYMENT FAILED — INITIATING ROLLBACK"
                echo "============================================"

                # Check if we saved a previous task definition to roll back to
                if [ -f /tmp/previous_task_def.txt ]; then
                    PREV_TASK_DEF=\$(cat /tmp/previous_task_def.txt)
                    echo "Rolling back to: \$PREV_TASK_DEF"

                    # Revert ECS service to the previous task definition
                    # This restarts the old working container
                    aws ecs update-service \
                      --cluster ${CLUSTER} \
                      --service ${SERVICE} \
                      --task-definition \$PREV_TASK_DEF \
                      --force-new-deployment \
                      --region ${AWS_REGION}

                    echo "Waiting 60s for rollback task to start..."
                    sleep 60

                    # Revert the ALB listener back to serve the previous version
                    # Users will now see the last known good version
                    aws elbv2 modify-listener \
                      --listener-arn ${LISTENER_ARN} \
                      --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_GREEN} \
                      --region ${AWS_REGION}

                    echo "============================================"
                    echo "  ROLLBACK COMPLETE"
                    echo "  Reverted to: \$PREV_TASK_DEF"
                    echo "  App URL: http://${ALB_DNS}:3000"
                    echo "============================================"
                else
                    # This happens if Stage 5 never ran (e.g. build failed early)
                    echo "No previous task definition found — skipping rollback"
                fi
                """
            }
        }

        success {
            // Confirm successful deployment in the Jenkins console
            echo "✅ Build #${BUILD_NUMBER} deployed successfully. App: http://${ALB_DNS}:3000"
        }
    }
}
