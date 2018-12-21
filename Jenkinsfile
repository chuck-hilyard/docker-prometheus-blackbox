
node('common')  {
	PROJECT_NAME = 'prometheus-blackbox'
  def CONSUL_URL = "http://consul:8500/v1/kv/${PROJECT_NAME}/config?keys"
  def response = httpRequest(contentType: 'APPLICATION_JSON', url: "${CONSUL_URL}")
  def consul_key_list = response.content.tokenize(",")
  consul_keys = [:]
  for (key in consul_key_list) {
    key = key.toString().replace("[","").replace("]","").replace("\"", "")
    response = httpRequest(contentType: 'APPLICATION_JSON', url: "http://consul:8500/v1/kv/${key}?raw")
    value = response.content
    key = key.toString().replace("${PROJECT_NAME}/config/", "")
    consul_keys[key] = value
  }

  stage('Code Checkout') {
    git branch: "${consul_keys["branch"]}", url: "${consul_keys["github_repo"]}"
    checkout scm
    //stash includes: '**', name: 'everything'
  }

  stage('App Build') {
		//unstash 'everything'
    sh "GOROOT=/usr/local/go GOPATH="$HOME/go" go get github.com/prometheus/blackbox_exporter"
    sh "GOOS=linux make build" 
    sh "mv docker-prometheus-blackbox blackbox_exporter"
    stash includes: '**', name: 'everything'
  }
}

node('docker-builds') {

  FQDN_HYPHENATED = "${consul_keys["FQDN"]}".toString().replace(".","-")

  stage('Docker Build') {
		unstash 'everything'
    sh "docker build -t ${PROJECT_NAME}:${consul_keys["branch"]} ."
    sh "docker tag ${PROJECT_NAME}:${consul_keys["branch"]} ${consul_keys["AWS_ACCOUNT_NUMBER"]}.dkr.ecr.us-west-2.amazonaws.com/${PROJECT_NAME}-${FQDN_HYPHENATED}:${consul_keys["branch"]}"
  }

  stage('Docker Deploy') {
    AWS_LOGIN = sh(script: "aws ecr get-login --region ${consul_keys["REGION"]} --profile ${consul_keys["ENVIRONMENT"]}-${consul_keys["PLATFORM"]} --no-include-email", returnStdout: true).trim()
    sh(script: "echo $AWS_LOGIN |/bin/bash -; docker push ${consul_keys["AWS_ACCOUNT_NUMBER"]}.dkr.ecr.us-west-2.amazonaws.com/${PROJECT_NAME}-${FQDN_HYPHENATED}:${consul_keys["branch"]}", returnStdout: true)
  }
}

// groovy ONLY executes on master nodes and must be included in scriptApproval.xml
// README: https://github.com/jenkinsci/pipeline-plugin/blob/master/TUTORIAL.md#serializing-local-variables
import groovy.text.StreamingTemplateEngine

@NonCPS
def sortBindings(vars) {
  def template = new StreamingTemplateEngine().createTemplate(text);
  String stuff = template.make(vars);
	return stuff;
}
