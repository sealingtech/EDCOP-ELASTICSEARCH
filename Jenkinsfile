#!/usr/bin/groovy

node {
  def app



  def pwd = pwd()
  def tool_name="elasticsearch"
  def support_tool_name="curator"
  def container_dir = "$pwd/containers/"
  def custom_image = "images.elasticsearch"
  /* #def custom_values_url = "http://repos.sealingtech.com/cisco-c240-m5/elasticsearch/values.yaml" */
  def user_id = ''
  wrap([$class: 'BuildUser']) {
      echo "userId=${BUILD_USER_ID},fullName=${BUILD_USER},email=${BUILD_USER_EMAIL}"
      user_id = "${BUILD_USER_ID}"
  }

  sh "env"

  def container_tag = "gcr.io/edcop-dev/$user_id-$tool_name"
  def support_container_tag = "gcr.io/edcop-dev/$user_id-$support_tool_name"

  stage('Clone repository') {
      /* Let's make sure we have the repository cloned to our workspace */
      checkout scm
  }


  stage('Build images') {
      /* This builds the actual image; synonymous to
       * docker build on the command line */
      println("Building $container_tag:$env.BUILD_ID and $support_container_tag:$env.BUILD_ID")

      app = docker.build("$container_tag:$env.BUILD_ID","./containers/elasticsearch")
      app = docker.build("$support_container_tag:$env.BUILD_ID","./containers/curator")
  }


  stage('Push images') {
      /* Finally, we'll push the images with two tags:
       * First, the incremental build number from Jenkins
       * Second, the 'latest' tag.
       * Pushing multiple tags is cheap, as all the layers are reused. */
      docker.withRegistry('https://gcr.io/edcop-dev/', 'gcr:edcop-dev') {
          app.push("$env.BUILD_ID")
      }
  }

  stage('helm lint') {
      sh "helm lint $tool_name"
  }

  stage('helm deploy') {
      /* Don't actually need the images, the official ones work */
      sh "helm install --name='$user_id-$tool_name-$env.BUILD_ID' $tool_name"
  }

  stage('sleeping 7 minutes') {
    sleep(420)
  }

  stage('Verifying running pods') {
    /* Master */
    def master_number_scheduled=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name-master  -o jsonpath={.status.replicas}").trim()
    def master_number_current=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name-master  -o jsonpath={.status.currentReplicas}").trim()
    def master_number_ready=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name-master  -o jsonpath={.status.readyReplicas}").trim()
 
    /* Workers */
    def worker_number_scheduled=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.replicas}").trim()
    def worker_number_current=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.currentReplicas}").trim()
    def worker_number_ready=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.readyReplicas}").trim()

    /* Printing Result */
    println("[MASTER] Ready pods: $master_number_ready  Current pods: $master_number_current  Scheduled pods: $master_number_scheduled")
    println("[WORKER] Ready pods: $worker_number_ready  Current Pods: $worker_number_current  Scheduled pods: $worker_number_scheduled")

    /* Verifying Result */
    if(master_number_ready==master_number_scheduled) {
      println("Master pods are running")
    } else {
      println("Some or all of the master pods failed")
      error("Some or all of the master pods failed")
    } 
    if(worker_number_ready==worker_number_scheduled) {
      println("Worker pods are running")
    } else {
      println("Some or all of the worker pods failed")
      error("Some or all of the worker pods failed")
    }
  }

  stage('Verifying Elasticsearch started on first pods') {
    /* Master */
    def master_command="kubectl get pods  | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name-master | awk "+'{\'print $1\'}'+"| head -1"
    def first_master_pod=sh(returnStdout: true, script: master_command)
    def master_command2="kubectl logs $first_master_pod | grep started"
    println("Master logs:")
    println(master_command2) 
    sh(master_command)

    /* Worker */
    def worker_command="kubectl get pods  | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name | awk "+'{\'print $1\'}'+"| head -1"
    def first_worker_pod=sh(returnStdout: true, script: worker_command)
    def worker_command2="kubectl logs $first_worker_pod | grep started"
    println("Worker logs:")
    println(worker_command2)
    sh(worker_command)
  }

  stage('Verify cluster health') {
    /* Uses worker pod */
    def worker_command="kubectl get pods  | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name | awk "+'{\'print $1\'}'+"| head -1"
    def first_worker_pod=sh(returnStdout: true, script: worker_command).trim()
    def health_command="kubectl exec -i $first_worker_pod -- bash -c \"curl -X --head data-service" + ':' + "9200/_cluster/health\" | jq --raw-output \'.status\'"
    def health=sh(returnStdout: true, script: health_command).trim()

    /* Health should be green */
    if(health=="green") {
      println("Cluster health is green")
    } else {
      println("Cluster health isn't green, something went wrong.")
      error("Cluster health isn't green, something went wrong.")
    }
  }
}
