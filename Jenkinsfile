def SERVICE_GROUP = "demo"
def SERVICE_NAME = "runtime-protection"
def IMAGE_NAME = "${SERVICE_NAME}"
def REPOSITORY_URL = "https://github.com/SecOpsDemo/prisma-runtime-protection.git"
def REPOSITORY_SECRET = ""
def SLACK_TOKEN_DEV = ""
def SLACK_TOKEN_DQA = ""

@Library("github.com/opsnow-tools/valve-butler")
def butler = new com.opsnow.valve.v7.Butler()
def label = "worker-${UUID.randomUUID().toString()}"

def bucket = 'aws-sam-cli-managed-default-samclisourcebucket-10qrl9wgc1thr'
def functionName = 'prisma-runtime-protection-SubProcessFunction-KEU34X3B4KEL'
def region = 'ap-northeast-2'

properties([
  buildDiscarder(logRotator(daysToKeepStr: "60", numToKeepStr: "30"))
])
podTemplate(label: label, containers: [
  containerTemplate(name: "builder", image: "opsnowtools/valve-builder:v0.2.36", command: "cat", ttyEnabled: true, alwaysPullImage: true),
  containerTemplate(name: "sam", image: "pahud/aws-sam-cli:0.53.0", command: "cat", ttyEnabled: true)
], volumes: [
  hostPathVolume(mountPath: "/var/run/docker.sock", hostPath: "/var/run/docker.sock"),
  hostPathVolume(mountPath: "/home/jenkins/.draft", hostPath: "/home/jenkins/.draft"),
  hostPathVolume(mountPath: "/home/jenkins/.helm", hostPath: "/home/jenkins/.helm")
]) {
  node(label) {
    stage("Prepare") {
      container("builder") {
        butler.prepare(IMAGE_NAME)
      }
    }
    stage("Checkout") {
      container("builder") {
        try {
          if (REPOSITORY_SECRET) {
            git(url: REPOSITORY_URL, branch: BRANCH_NAME, credentialsId: REPOSITORY_SECRET)
          } else {
            git(url: REPOSITORY_URL, branch: BRANCH_NAME)
          }
        } catch (e) {
          butler.failure(SLACK_TOKEN_DEV, "Checkout")
          throw e
        }

        butler.scan("python")
      }
    }
    // stage("Tests") {
    //   container("maven") {
    //     try {
    //       butler.mvn_test()
    //     } catch (e) {
    //       butler.failure(SLACK_TOKEN_DEV, "Tests")
    //       throw e
    //     }
    //   }
    // }
    stage("Build") {
      container("builder") {
        try {
          sh "zip -rj ${commitID()}.zip run_subprocess"
          butler.success(SLACK_TOKEN_DEV, "Build")
        } catch (e) {
          butler.failure(SLACK_TOKEN_DEV, "Build")
          throw e
        }
      }
    }

    if (BRANCH_NAME == "master") {
      stage('Scan Image - Prisma Cloud') {
        // Scan the image
        prismaCloudScanFunction cloudFormationTemplateFile: '', 
          functionName: '', 
          functionPath: "${commitID()}.zip", 
          logLevel: 'info', 
          project: '', 
          resultsFile: 'prisma-cloud-scan-results.json'
        
        prismaCloudPublish resultsFilePattern: 'prisma-cloud-scan-results.json'
      }
      stage('Scan Image - Prisma Cloud') {
        container("builder") {
          try {
            envAWS("here");
            sh "aws s3 cp ${commitID()}.zip s3://${bucket}"
            butler.success(SLACK_TOKEN_DEV, "Build")
          } catch (e) {
            butler.failure(SLACK_TOKEN_DEV, "Build")
            throw e
          }
        }
      }
      stage("Deploy") {
        container("builder") {
          try {
            sh "aws lambda update-function-code --function-name ${functionName} --s3-bucket ${bucket} --s3-key ${commitID()}.zip --region ${region}"
            def lambdaVersion = sh(
              script: "aws lambda publish-version --function-name ${functionName} --region ${region} | jq -r '.Version'",
              returnStdout: true
            )
            sh "aws lambda update-alias --function-name ${functionName} --name production --region ${region} --function-version ${lambdaVersion}"
            butler.success(SLACK_TOKEN_DEV, "Deploy")
          } catch (e) {
            butler.failure(SLACK_TOKEN_DEV, "Deploy")
            throw e
          }
        }
      }
    }
  }
}

def commitID() {
    sh 'git rev-parse HEAD > .git/commitID'
    def commitID = readFile('.git/commitID').trim()
    sh 'rm .git/commitID'
    commitID
}

def envAWS(target = "") {
    if (!target) {
        throw new RuntimeException("env_target:target is null.")
    }

    sh """
        rm -rf ${home}/.aws && mkdir -p ${home}/.aws
    """

    this.target = target

    // check target secret
    count = sh(script: "kubectl get secret -n devops | grep 'aws-config-${target}' | wc -l", returnStdout: true).trim()
    if ("${count}" == "0") {
        echo "env_target:target is null."
        throw new RuntimeException("target is null.")
    }

    sh """
        kubectl get secret aws-config-${target} -n devops -o json | jq -r .data.config | base64 -d > ${home}/aws_config
        kubectl get secret aws-config-${target} -n devops -o json | jq -r .data.credentials | base64 -d > ${home}/aws_credentials
        cp ${home}/aws_config ${home}/.aws/config
        cp ${home}/aws_credentials ${home}/.aws/credentials
    """
}
