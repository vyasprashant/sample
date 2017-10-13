node {
  def project = 'sample'
  def appName = 'sample'
  def feSvcName = "${appName}-frontend"
  def imageTag = "prashantvyas/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"

  checkout scm

  stage 'Build image'
  sh("cd sample-app; sudo docker build -t ${imageTag} -f Dockerfile .")

  stage 'Run Go tests'
  sh("sudo docker run ${imageTag} go test")

  stage 'Push image to registry'
  sh("sudo docker login --username=prashantvyas --password=prashant12 ;sudo docker push ${imageTag}")

  stage "Deploy Application"
  switch (env.BRANCH_NAME) {
    // Roll out to canary environment
    case "canary":
        // Change deployed image in canary to the one we just built
        sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./sample-app/k8s/canary/*.yaml")
        sh("/usr/local/bin/kubectl --namespace=production apply -f sample-app/k8s/services/")
        sh("/usr/local/bin/kubectl --namespace=production apply -f sample-app/k8s/canary/")
        sh("echo http://`/usr/local/bin/kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
        break

    // Roll out to production
    case "master":
        // Change deployed image in canary to the one we just built
        sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./sample-app/k8s/production/*.yaml")
        sh("/usr/local/bin/kubectl --namespace=production apply -f sample-app/k8s/services/")
        sh("/usr/local/bin/kubectl --namespace=production apply -f sample-app/k8s/production/")
        sh("echo http://`/usr/local/bin/kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
        break

    // Roll out a dev environment
    default:
        // Create namespace if it doesn't exist
        sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
        // Don't use public load balancing for development branches
        sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./sample-app/k8s/services/frontend.yaml")
        sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./sample-app/k8s/dev/*.yaml")
        sh("/usr/local/bin/kubectl --namespace=${env.BRANCH_NAME} apply -f sample-app/k8s/services/")
        sh("/usr/local/bin/kubectl --namespace=${env.BRANCH_NAME} apply -f sample-app/k8s/dev/")
        echo 'To access your environment run `kubectl proxy`'
        echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
  }
}
