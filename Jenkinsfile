#!/usr/bin/groovy

@Library('github.com/lachie83/jenkins-pipeline@dev')
def pipeline = new io.estrado.Pipeline()


node {
  def app
  def pwd = pwd()
  def tool_name="elasticsearch"
  def support_tool_name="curator"
  def container_dir = "$pwd/containers/"
  def custom_image = "images.elasticsearch"
  def breakout_roles = true
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

  stage('helm lint') {
      sh "helm lint $tool_name"
  }

  stage('helm deploy') {
      sh "helm install --name='$user_id-$tool_name-$env.BUILD_ID' $tool_name"
  }

  stage('sleeping 3 minutes') {
    sleep(180)
  }

  stage('Verifying running pods') {
    if (!breakout_roles) {
      /* Default Nodes */
      def number_scheduled=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.replicas}").trim()
      def number_current=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.currentReplicas}").trim()
      def number_ready=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.readyReplicas}").trim()
    
      /* Printing Result */
      println("Scheduled Pods: $number_scheduled | Ready pods: $number_ready | Current pods: $number_current")

      /* Verifying Result */
      if(number_ready==number_scheduled) {
        println("All pods are running")
      } else {
        println("Some or all of the pods failed")
        error("Some or all of the pods failed")
      }
    } 
    else {
      /* Breakout Nodes */
      def master_number_scheduled=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name-master  -o jsonpath={.status.replicas}").trim()
      def master_number_current=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name-master  -o jsonpath={.status.currentReplicas}").trim()
      def master_number_ready=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name-master  -o jsonpath={.status.readyReplicas}").trim()
      def client_number_scheduled=sh(returnStdout: true, script: "kubectl get deployment $user_id-$tool_name-$env.BUILD_ID-$tool_name-client  -o jsonpath={.status.availableReplicas}").trim()
      def client_number_current=sh(returnStdout: true, script: "kubectl get deployment $user_id-$tool_name-$env.BUILD_ID-$tool_name-client  -o jsonpath={.status.availableReplicas}").trim()
      def client_number_ready=sh(returnStdout: true, script: "kubectl get deployment $user_id-$tool_name-$env.BUILD_ID-$tool_name-client  -o jsonpath={.status.readyReplicas}").trim()
      def data_number_scheduled=sh(returnStdout: true, script: "kubectl get daemonset $user_id-$tool_name-$env.BUILD_ID-$tool_name-data  -o jsonpath={.status.desiredNumberScheduled}").trim()
      def data_number_current=sh(returnStdout: true, script: "kubectl get daemonset $user_id-$tool_name-$env.BUILD_ID-$tool_name-data  -o jsonpath={.status.currentNumberScheduled}").trim()
      def data_number_ready=sh(returnStdout: true, script: "kubectl get daemonset $user_id-$tool_name-$env.BUILD_ID-$tool_name-data  -o jsonpath={.status.numberReady}").trim()
    
      /* Printing Result */
      println("[MASTER NODES] Scheduled Pods: $master_number_scheduled | Ready pods: $master_number_ready | Current pods: $master_number_current")
      println("[CLIENT NODES] Scheduled Pods: $client_number_scheduled | Ready pods: $client_number_ready | Current pods: $client_number_current")
      println("[DATA NODES]   Scheduled Pods: $data_number_scheduled | Ready pods: $data_number_ready | Current pods: $data_number_current")
      
      /* Verifying Result */
      if (master_number_ready==master_number_scheduled) {
        println("All master pods are running")
      } else {
        println("Some or all of the master pods failed")
        error("Some or all of the master pods failed")
      }
      if (client_number_ready==client_number_scheduled) {
        println("All client pods are running")
      } else {
        println("Some or all of the client pods failed")
        error("Some or all of the client pods failed")
      }
      if (data_number_ready==data_number_scheduled) {
        println("All data pods are running")
      } else {
        println("Some or all of the data pods failed")
        error("Some or all of the data pods failed")
      }
    }
  }

  stage('Verify cluster health') {
    def command="kubectl get pods | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name | awk "+'{\'print $1\'}'+"| head -1"
    if (breakout_roles) {
      command="kubectl get pods | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name-master | awk "+'{\'print $1\'}'+"| head -1"
    }
    def first_pod=sh(returnStdout: true, script: command).trim()
    /* You MUST have jq installed on Jenkins' filesystem or container */
    def health_command="kubectl exec -i " + "$first_pod" + " -- bash -c \"curl --silent -X --head data-service" + ':' + "9200/_cluster/health\" | jq --raw-output \'.status\'"
    def health=sh(returnStdout: true, script: health_command).trim()

    /* Health should be green */
    if(health=="green") {
      println("Cluster health is green")
    } else if (health=="yellow") {
      println("WARNING: Cluster health is yellow")
    } else {
      println("ERROR: Cluster health is red, something went wrong")
      error("ERROR: Cluster health is red, something went wrong")
    }
  }
  
  stage('Verify init scripts completed') {
    /* Get elasticsearch template init logs */
    def init_job_command="kubectl get pods | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name-post-installs-job | awk "+'{\'print $1\'}'+"| head -1"
    def init_job_pod=sh(returnStdout: true, script: init_job_command).trim()
    def init_job_status=sh(returnStdout: true, script: "kubectl get pod $init_job_pod -o jsonpath={.status.phase}").trim()
    
    if(init_job_status=="Succeeded") {
      println("Initialization jobs completed successfully.")
    } else {
      println("ERROR: Initialization jobs did not complete sucessfully")
      error("ERROR: Initialization jobs did not complete sucessfully")
    }
  }
}
