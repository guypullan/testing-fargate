#!/usr/bin/env groovy

def dockerregistry = "329802642264.dkr.ecr.eu-west-1.amazonaws.com"
def certsprep = "/scripts/infrastructurebuild/certsprep.sh"
def clean = "git clean -ffde certs"
def GitBranchName = scm.branches[0].name
def cronstring = ""
//if ( GitBranchName == 'master' ) { cronstring = "itworked" } else { cronstring = "definedbutwrong" }
if (GitBranchName == 'master') { cronstring = "05 15 * * 1-5 % BUILDTASK=infrastructuredeployment;FUNCTION=stackupdate;STACKSCALING=standard;ENVIRONMENT=test;STACKLIST=main\n10 15 * * 1-5 % BUILDTASK=infrastructuredeployment;FUNCTION=stackupdate;STACKSCALING=standard;ENVIRONMENT=test;STACKLIST=main"}

pipeline {
  agent any
  options {
    timeout(time: 1, unit: 'HOURS')
    timestamps ()
  }
  environment {
    COSMOSREPOCONFIG = 'repoconfig.json'
    CONFIGLOCATION = 'infrastructure'
    COMPONENTNAME = 'testing-fargate'
    CONFIGFILE = "${env.ENVIRONMENT}-${COMPONENTNAME}.json"
    COSMOS_CERT = "/workspace/certs/cert.pem"
    KEY = "/workspace/certs/client.key"
    CERT = "/workspace/certs/client.crt"
  }
  parameters {
    choice(name: 'BUILDTASK', choices: 'infrastructuredeployment\napplicationdeployment\n\nboth', description: 'Are you working with Infrastructure, Software or both?')
    choice(name: 'FUNCTION', choices: 'stackcreation\nstackteardown\nstackupdate', description: 'What task are you wanting to perform?')
    choice(name: 'ENVIRONMENT', choices: 'int\ntest\nlive', description: 'Which environment are you building?')
    string(name: 'PROJECTNAME', defaultValue: 'news-devops-toolchain', description: 'What is the project name?')
    string(name: 'STACKLIST', defaultValue: 'all', description: 'which stacks are you working with?  if you want to perform work on all stacks put \"all\" in this section')
    choice(name: 'DEPLOYAPP', choices: 'no\nyes', description: 'Do you want to deploy your application?')
  }
  triggers {
    parameterizedCron(cronstring)
  }
  stages {
    stage ('Initialize') {
      steps {
        echo "performing workspace preparation tasks"
      }
    }
    stage ('cosmos component creation') {
      when {
        expression { (params.BUILDTASK == 'infrastructuredeployment' || params.BUILDTASK == 'both') && params.FUNCTION != 'stackteardown' }
      }
      agent {
        docker {
          image "${dockerregistry}/bbc-news/infrastructurebuild-tools:0.0.10"
          args "-u root -v /etc/pki/tls:/certs"
        }
      }
      steps {
          sh "${clean}"
          checkout scm
          sh "${certsprep}"
          sh "/scripts/infrastructurebuild/cosmoscomponent.sh"
      }
    }
    stage ('stack create/delete/update') {
      when {
        expression { params.BUILDTASK == 'infrastructuredeployment' || params.BUILDTASK == 'both' }
      }
      agent {
        docker {
          image "${dockerregistry}/bbc-news/infrastructurebuild-tools:0.0.10"
          args "-u root -v /etc/pki/tls:/certs"
        }
      }
      steps {
        sh "${clean}"
        checkout scm
        sh "${certsprep}"
        sh "/scripts/infrastructurebuild/${env.FUNCTION}.sh"
      }
    }
    stage ('cosmos configuration') {
      when {
        expression { (params.BUILDTASK == 'infrastructuredeployment' || params.BUILDTASK == 'both') && params.FUNCTION != 'stackteardown'}
      }
      agent {
        docker {
          image "${dockerregistry}/bbc-news/infrastructurebuild-tools:0.0.6"
          args "-u root -v /etc/pki/tls:/certs"
        }
      }
      steps {
        parallel(
          static: {
            sh "${clean}"
            checkout scm
            sh "${certsprep}"
            sh "/scripts/infrastructurebuild/cosmosconfig.sh"
          },
          stackoutputs: {
            sh "${clean}"
            checkout scm
            sh "${certsprep}"
            sh "/scripts/infrastructurebuild/cloudstackoutputstocosmos.sh"
          },
          repositories: {
            sh "${clean}"
            checkout scm
            sh "${certsprep}"
            sh "/scripts/infrastructurebuild/repoconfig.sh"
          }
        )
      }
    }
//    stage ('Application Dependencies') {
//      when {
//        expression { params.BUILDTASK == 'applicationdeployment' || params.BUILDTASK == 'both'}
//      }
//      agent {
//        docker {
//          image '329802642264.dkr.ecr.eu-west-1.amazonaws.com/bbc_news/ruby-23:latest'
//          args "-u root"
//        }
//      }
//      steps {
//        sh "${clean}"
//        checkout scm
//        sh 'make dependencies'
//        sh 'make lint'
//        sh 'make test'
//      }
//    }
//    stage ('Application Assets') {
//      when {
//        expression { params.BUILDTASK == 'applicationdeployment' || params.BUILDTASK == 'both'}
//      }
//      agent {
//        docker {
//          image 'tarqwyn/task-runner'
//          args "-u root"
//        }
//      }
//      steps {
//        sh "${clean}"
//        checkout scm
//        sh 'make assets'
//        sh 'make upload_assets'
//      }
//    }
//    stage ('Application Tests') {
//      when {
//        expression { params.BUILDTASK == 'applicationdeployment' || params.BUILDTASK == 'both'}
//      }
//      agent {
//        docker {
//          image '329802642264.dkr.ecr.eu-west-1.amazonaws.com/bbc_news/ruby-23:latest'
//          args "-u root"
//        }
//      }
//      steps {
//        sh "${clean}"
//        checkout scm
//        sh 'make lint'
//        sh 'make test'
//      }
//    }
//    stage ('Cosmos Build') {
//      when {
//        expression { params.BUILDTASK == 'applicationdeployment' || params.BUILDTASK == 'both'}
//      }
//      agent any
//      steps {
//        sh 'make rpm'
//        sh 'make release'
//      }
//    }
//    stage ('Application Deployment') {
//      when {
//        expression { params.DEPLOYAPP == 'yes' && params.FUNCTION != 'stackteardown' }
//      }
//      agent {
//        docker {
//          image "${dockerregistry}/bbc-news/infrastructurebuild-tools:0.0.4"
//          args "-u root -v /etc/pki/tls:/certs"
//        }
//      }
//      steps {
//        sh "${clean}"
//        checkout scm
//        sh "${certsprep}"
//        sh "/scripts/infrastructurebuild/applicationdeployment.sh"
//        echo "completed application deployment processes"
//      }
//    }
  }
}
