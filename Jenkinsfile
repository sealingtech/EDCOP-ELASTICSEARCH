#!/usr/bin/groovy

node {
  def app



  def pwd = pwd()
  def tool_name="elasticsearch"
  def support_tool_name="curator"
  def container_dir = "$pwd/containers/"
  def custom_image = "images.elasticsearch"
  #def custom_values_url = "http://repos.sealingtech.com/cisco-c240-m5/elasticsearch/values.yaml"
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
}
