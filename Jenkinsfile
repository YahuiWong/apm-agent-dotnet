#!/usr/bin/env groovy

@Library('apm@current') _

pipeline {
  agent { label 'linux && immutable' }
  environment {
    REPO = 'apm-agent-dotnet'
    // keep it short to avoid the 248 characters PATH limit in Windows
    BASE_DIR = "apm-agent-dotnet"
    NOTIFY_TO = credentials('notify-to')
    JOB_GCS_BUCKET = credentials('gcs-bucket')
    CODECOV_SECRET = 'secret/apm-team/ci/apm-agent-dotnet-codecov'
    GITHUB_CHECK_ITS_NAME = 'Integration Tests'
    ITS_PIPELINE = 'apm-integration-tests-selector-mbp/master'
    OPBEANS_REPO = 'opbeans-dotnet'
    BENCHMARK_SECRET  = 'secret/apm-team/ci/benchmark-cloud'
    SLACK_CHANNEL = '#apm-agent-dotnet'
    AZURE_RESOURCE_GROUP_PREFIX = "ci-dotnet-${env.BUILD_ID}"
  }
  options {
    timeout(time: 4, unit: 'HOURS')
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10', daysToKeepStr: '30'))
    timestamps()
    ansiColor('xterm')
    disableResume()
    durabilityHint('PERFORMANCE_OPTIMIZED')
    rateLimitBuilds(throttle: [count: 60, durationName: 'hour', userBoost: true])
    quietPeriod(10)
  }
  triggers {
    issueCommentTrigger('(?i)(.*(?:jenkins\\W+)?run\\W+(?:the\\W+)?(?:benchmark\\W+)?tests(?:\\W+please)?.*|/test)')
  }
  parameters {
    booleanParam(name: 'Run_As_Master_Branch', defaultValue: false, description: 'Allow to run any steps on a PR, some steps normally only run on master branch.')
  }
  stages {
    stage('Initializing'){
      stages{
        stage('Checkout') {
          options { skipDefaultCheckout() }
          steps {
            pipelineManager([ cancelPreviousRunningBuilds: [ when: 'PR' ] ])
            deleteDir()
            gitCheckout(basedir: "${BASE_DIR}", githubNotifyFirstTimeContributor: true)
            stash allowEmpty: true, name: 'source', useDefaultExcludes: false
            script {
              dir("${BASE_DIR}"){
                // Skip all the stages for PRs with changes in the docs only
                env.ONLY_DOCS = isGitRegionMatch(patterns: [ '^docs/.*' ], shouldMatchAll: true)

                 // Look for changes related to the benchmark, if so then set the env variable.
                def patternList = [
                  '^test/Elastic.Apm.Benchmarks/.*'
                ]
                env.BENCHMARK_UPDATED = isGitRegionMatch(patterns: patternList)
              }
            }
          }
        }
        stage('Parallel'){
          when {
            beforeAgent true
            expression { return env.ONLY_DOCS == "false" }
          }
          parallel{
            stage('Linux'){
              options { skipDefaultCheckout() }
              environment {
                MSBUILDDEBUGPATH = "${env.WORKSPACE}"
              }
              /**
              Make sure there are no code style violation in the repo.
              */
              stages{
              //   Disable until https://github.com/elastic/apm-agent-dotnet/issues/563
              //   stage('CodeStyleCheck') {
              //     steps {
              //       withGithubNotify(context: 'CodeStyle check') {
              //         deleteDir()
              //         unstash 'source'
              //         dir("${BASE_DIR}"){
              //           dotnet(){
              //             sh label: 'Install and run dotnet/format', script: '.ci/linux/codestyle.sh'
              //           }
              //         }
              //       }
              //     }
              //   }
                /**
                Build the project from code..
                */
                stage('Build') {
                  steps {
                    withGithubNotify(context: 'Build - Linux') {
                      deleteDir()
                      unstash 'source'
                      dir("${BASE_DIR}"){
                        dotnet(){
                          sh '.ci/linux/build.sh'
                          whenTrue(isPR()) {
                            sh(label: 'Package', script: '.ci/linux/release.sh true')
                          }
                        }
                      }
                    }
                  }
                  post {
                    unsuccessful {
                      archiveArtifacts(allowEmptyArchive: true,
                        artifacts: "${MSBUILDDEBUGPATH}/**/MSBuild_*.failure.txt")
                    }
                    success {
                      whenTrue(isPR()) {
                        archiveArtifacts(allowEmptyArchive: true, artifacts: "${BASE_DIR}/build/output/_packages/*.nupkg,${BASE_DIR}/build/output/*.zip")
                      }
                    }
                  }
                }
                /**
                Execute unit tests.
                */
                stage('Test') {
                  steps {
                    withGithubNotify(context: 'Test - Linux', tab: 'tests') {
                      deleteDir()
                      unstash 'source'
                      filebeat(output: "docker.log"){
                        dir("${BASE_DIR}"){
                          dotnet(){
                            sh label: 'Test & coverage', script: '.ci/linux/test.sh'
                          }
                        }
                      }
                    }
                  }
                  post {
                    always {
                      reportTests()
                      publishCoverage(adapters: [coberturaAdapter("${BASE_DIR}/target/**/*coverage.cobertura.xml")],
                                      sourceFileResolver: sourceFiles('STORE_ALL_BUILD'))
                      codecov(repo: env.REPO, basedir: "${BASE_DIR}", secret: "${CODECOV_SECRET}")
                    }
                    unsuccessful {
                      archiveArtifacts(allowEmptyArchive: true,
                        artifacts: "${MSBUILDDEBUGPATH}/**/MSBuild_*.failure.txt")
                    }
                  }
                }
              }
            }
            stage('Windows .NET Framework'){
              agent { label 'windows-2019-immutable' }
              options { skipDefaultCheckout() }
              environment {
                HOME = "${env.WORKSPACE}"
                DOTNET_ROOT = "${env.WORKSPACE}\\dotnet"
                PATH = "${env.DOTNET_ROOT};${env.DOTNET_ROOT}\\tools;${env.PATH};${env.HOME}\\bin"
                MSBUILDDEBUGPATH = "${env.WORKSPACE}"
              }
              stages{
                /**
                Install the required tools
                */
                stage('Install tools') {
                  steps {
                    cleanDir("${WORKSPACE}/*")
                    unstash 'source'
                    dir("${HOME}"){
                      powershell label: 'Install tools', script: "${BASE_DIR}\\.ci\\windows\\tools.ps1"
                      powershell label: 'Install msbuild tools', script: "${BASE_DIR}\\.ci\\windows\\msbuild-tools.ps1"
                    }
                  }
                }
                /**
                Build the project from code..
                */
                stage('Build - MSBuild') {
                  steps {
                    withGithubNotify(context: 'Build MSBuild - Windows') {
                      cleanDir("${WORKSPACE}/${BASE_DIR}")
                      unstash 'source'
                      dir("${BASE_DIR}"){
                        bat '.ci/windows/msbuild.bat'
                      }
                    }
                  }
                  post {
                    unsuccessful {
                      archiveArtifacts(allowEmptyArchive: true,
                        artifacts: "${MSBUILDDEBUGPATH}/**/MSBuild_*.failure.txt")
                    }
                  }
                }
                /**
                Execute unit tests.
                */
                stage('Test') {
                  steps {
                    withGithubNotify(context: 'Test MSBuild - Windows', tab: 'tests') {
                      cleanDir("${WORKSPACE}/${BASE_DIR}")
                      unstash 'source'
                      dir("${BASE_DIR}"){
                        powershell label: 'Install test tools', script: '.ci\\windows\\test-tools.ps1'
                        bat label: 'Prepare solution', script: '.ci/windows/prepare-test.bat'
                        bat label: 'Build', script: '.ci/windows/msbuild.bat'
                        bat label: 'Test & coverage', script: '.ci/windows/testnet461.bat'
                      }
                    }
                  }
                  post {
                    always {
                      reportTests()
                    }
                    unsuccessful {
                      archiveArtifacts(allowEmptyArchive: true,
                        artifacts: "${MSBUILDDEBUGPATH}/**/MSBuild_*.failure.txt")
                    }
                  }
                }
                /**
                Execute IIS tests.
                */
                stage('IIS Tests') {
                  steps {
                    withGithubNotify(context: 'IIS Tests', tab: 'tests') {
                      cleanDir("${WORKSPACE}/${BASE_DIR}")
                      unstash 'source'
                      dir("${BASE_DIR}"){
                        powershell label: 'Install test tools', script: '.ci\\windows\\test-tools.ps1'
                        bat label: 'Build', script: '.ci/windows/msbuild.bat'
                        bat label: 'Test IIS', script: '.ci/windows/test-iis.bat'
                      }
                    }
                  }
                  post {
                    always {
                      reportTests()
                    }
                  }
                }
              }
              post {
                always {
                  cleanWs(disableDeferredWipeout: true, notFailBuild: true)
                }
              }
            }
            stage('Windows .NET Core'){
              agent { label 'windows-2019-immutable' }
              options { skipDefaultCheckout() }
              environment {
                HOME = "${env.WORKSPACE}"
                DOTNET_ROOT = "C:\\Program Files\\dotnet"
                PATH = "${env.DOTNET_ROOT};${env.DOTNET_ROOT}\\tools;${env.PATH};${env.HOME}\\bin"
                MSBUILDDEBUGPATH = "${env.WORKSPACE}"
              }
              stages{
                /**
                Install the required tools
                */
                stage('Install tools') {
                  steps {
                    cleanDir("${WORKSPACE}/*")
                    unstash 'source'
                    dir("${HOME}"){
                      powershell label: 'Install tools', script: "${BASE_DIR}\\.ci\\windows\\tools.ps1"
                    }
                  }
                }
                /**
                Build the project from code..
                */
                stage('Build - dotnet') {
                  steps {
                    withGithubNotify(context: 'Build dotnet - Windows') {
                      retry(3) {
                        cleanDir("${WORKSPACE}/${BASE_DIR}")
                        unstash 'source'
                        dir("${BASE_DIR}"){
                          bat label: 'Build', script: '.ci/windows/dotnet.bat'
                        }
                      }
                    }
                  }
                  post {
                    unsuccessful {
                      archiveArtifacts(allowEmptyArchive: true,
                        artifacts: "${MSBUILDDEBUGPATH}/**/MSBuild_*.failure.txt")
                    }
                  }
                }
                /**
                Execute unit tests.
                */
                stage('Test') {
                  steps {
                    withGithubNotify(context: 'Test dotnet - Windows', tab: 'tests') {
                      cleanDir("${WORKSPACE}/${BASE_DIR}")
                      unstash 'source'
                      dir("${BASE_DIR}"){
                        powershell label: 'Install test tools', script: '.ci\\windows\\test-tools.ps1'
                        retry(3) {
                          bat label: 'Build', script: '.ci/windows/dotnet.bat'
                        }
                        withAzureCredentials(path: "${HOME}", credentialsFile: '.credentials.json') {
                          bat label: 'Test & coverage', script: '.ci/windows/test.bat'
                        }
                      }
                    }
                  }
                  post {
                    always {
                      reportTests()
                    }
                    unsuccessful {
                      archiveArtifacts(allowEmptyArchive: true, artifacts: "${MSBUILDDEBUGPATH}/**/MSBuild_*.failure.txt")
                    }
                  }
                }
                stage('Startup Hook Tests') {
                  steps {
                    withGithubNotify(context: 'Test startup hooks - Windows', tab: 'tests') {
                      cleanDir("${WORKSPACE}/${BASE_DIR}")
                      unstash 'source'
                      dir("${BASE_DIR}"){
                        powershell label: 'Install test tools', script: '.ci\\windows\\test-tools.ps1'
                        retry(3) {
                          bat label: 'Build', script: '.ci/windows/zip.bat'
                        }
                        bat label: 'Test & coverage', script: '.ci/windows/test-zip.bat'
                      }
                    }
                  }
                  post {
                    always {
                      reportTests()
                    }
                    unsuccessful {
                      archiveArtifacts(allowEmptyArchive: true, artifacts: "${MSBUILDDEBUGPATH}/**/MSBuild_*.failure.txt")
                    }
                  }
                }
              }
              post {
                always {
                  cleanWs(disableDeferredWipeout: true, notFailBuild: true)
                }
              }
            }
            stage('Integration Tests') {
              agent none
              when {
                anyOf {
                  changeRequest()
                  expression { return !params.Run_As_Master_Branch }
                }
              }
              steps {
                build(job: env.ITS_PIPELINE, propagate: false, wait: false,
                      parameters: [string(name: 'INTEGRATION_TEST', value: '.NET'),
                                    string(name: 'BUILD_OPTS', value: "--dotnet-agent-version ${env.GIT_BASE_COMMIT} --opbeans-dotnet-agent-branch ${env.GIT_BASE_COMMIT}"),
                                    string(name: 'GITHUB_CHECK_NAME', value: env.GITHUB_CHECK_ITS_NAME),
                                    string(name: 'GITHUB_CHECK_REPO', value: env.REPO),
                                    string(name: 'GITHUB_CHECK_SHA1', value: env.GIT_BASE_COMMIT)])
                githubNotify(context: "${env.GITHUB_CHECK_ITS_NAME}", description: "${env.GITHUB_CHECK_ITS_NAME} ...", status: 'PENDING', targetUrl: "${env.JENKINS_URL}search/?q=${env.ITS_PIPELINE.replaceAll('/','+')}")
              }
            }
            stage('Benchmarks') {
              agent { label 'metal' }
              environment {
                REPORT_FILE = 'apm-agent-benchmark-results.json'
                HOME = "${env.WORKSPACE}"
              }
              when {
                beforeAgent true
                allOf {
                  anyOf {
                    branch 'master'
                    tag pattern: '\\d+\\.\\d+\\.\\d+.*', comparator: 'REGEXP'
                    expression { return params.Run_As_Master_Branch }
                    expression { return env.BENCHMARK_UPDATED != "false" }
                    expression { return env.GITHUB_COMMENT?.contains('benchmark tests') }
                  }
                  expression { return env.ONLY_DOCS == "false" }
                }
              }
              options {
                warnError('Benchmark failed')
                timeout(time: 1, unit: 'HOURS')
              }
              steps {
                withGithubNotify(context: 'Benchmarks') {
                  deleteDir()
                  unstash 'source'
                  dir("${BASE_DIR}") {
                    script {
                      sendBenchmarks.prepareAndRun(secret: env.BENCHMARK_SECRET, url_var: 'ES_URL',
                                                   user_var: 'ES_USER', pass_var: 'ES_PASS') {
                        sh '.ci/linux/benchmark.sh'
                      }
                    }
                  }
                }
              }
              post {
                always {
                  catchError(message: 'deleteDir failed', buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    deleteDir()
                  }
                }
              }
            }
          }
        }
      }
    }
    stage('Release to feedz.io') {
      options { skipDefaultCheckout() }
      when {
        beforeAgent true
        anyOf {
          branch 'master'
          expression { return params.Run_As_Master_Branch }
        }
      }
      steps {
        deleteDir()
        unstash 'source'
        dir("${BASE_DIR}"){
          release(secret: 'secret/apm-team/ci/elastic-observability-feedz.io', withSuffix: true)
        }
      }
      post{
        success {
          archiveArtifacts(allowEmptyArchive: true,
            artifacts: "${BASE_DIR}/build/output/_packages/*.nupkg")
        }
      }
    }
    stage('Release') {
      options {
        skipDefaultCheckout()
      }
      when {
        beforeInput true
        beforeAgent true
        // Tagged release events ONLY
        tag pattern: '\\d+\\.\\d+\\.\\d+(-(alpha|beta|rc)\\d*)?', comparator: 'REGEXP'
      }
      stages {
        stage('Notify') {
          steps {
            notifyStatus(slackStatus: 'warning', subject: "[${env.REPO}] Release ready to be pushed",
                         body: "Please (<${env.BUILD_URL}input|approve>) it or reject within 12 hours.\n Changes: ${env.TAG_NAME}")
          }
        }
        stage('Release to NuGet') {
          input {
            message 'Should we release a new version?'
            ok 'Yes, we should.'
          }
          environment {
            RELEASE_URL_MESSAGE = "(<https://github.com/elastic/apm-agent-dotnet/releases/tag/${env.TAG_NAME}|${env.TAG_NAME}>)"
          }
          steps {
            deleteDir()
            unstash 'source'
            dir("${BASE_DIR}") {
              release(secret: 'secret/apm-team/ci/elastic-observability-nuget')
            }
          }
          post {
            failure {
              notifyStatus(slackStatus: 'danger', subject: "[${env.REPO}] Release *${env.TAG_NAME}* failed", body: "Build: (<${env.RUN_DISPLAY_URL}|here>)")
            }
            success {
              notifyStatus(slackStatus: 'good', subject: "[${env.REPO}] Release *${env.TAG_NAME}* published", body: "Build: (<${env.RUN_DISPLAY_URL}|here>)\nRelease URL: ${env.RELEASE_URL_MESSAGE}")
            }
          }
        }
      }
    }
    stage('AfterRelease') {
      options {
        skipDefaultCheckout()
      }
      when {
        anyOf {
          tag pattern: '\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
        }
      }
      stages {
        stage('Opbeans') {
          environment {
            REPO_NAME = "${OPBEANS_REPO}"
          }
          steps {
            deleteDir()
            dir("${OPBEANS_REPO}"){
              git credentialsId: 'f6c7695a-671e-4f4f-a331-acdce44ff9ba',
                  url: "git@github.com:elastic/${OPBEANS_REPO}.git"
              sh script: ".ci/bump-version.sh ${env.BRANCH_NAME}", label: 'Bump version'
              // The opbeans pipeline will trigger a release for the master branch
              gitPush()
              // The opbeans pipeline will trigger a release for the release tag with the format v<major>.<minor>.<patch>
              gitCreateTag(tag: "v${env.BRANCH_NAME}")
            }
          }
        }
      }
    }
  }
  post {
    cleanup {
      cleanupAzureResources()
      notifyBuildResult()
    }
  }
}

def cleanDir(path){
  powershell label: "Clean ${path}", script: "Remove-Item -Recurse -Force ${path}"
}

def dotnet(Closure body){

  def homePath = "${env.WORKSPACE}/${env.BASE_DIR}"
  withEnv([
    "HOME=${homePath}",
    "DOTNET_ROOT=${homePath}/.dotnet",
    "PATH+DOTNET=${homePath}/.dotnet/tools:${homePath}/.dotnet"
    ]){
    sh(label: 'Install dotnet SDK', script: """
    mkdir -p \${DOTNET_ROOT}
    # Download .Net SDK installer script
    curl -s -O -L https://dotnet.microsoft.com/download/dotnet-core/scripts/v1/dotnet-install.sh
    chmod ugo+rx dotnet-install.sh

    # Install .Net SDKs
    ./dotnet-install.sh --install-dir "\${DOTNET_ROOT}" -version '2.1.505'
    ./dotnet-install.sh --install-dir "\${DOTNET_ROOT}" -version '3.0.103'
    ./dotnet-install.sh --install-dir "\${DOTNET_ROOT}" -version '3.1.100'
    ./dotnet-install.sh --install-dir "\${DOTNET_ROOT}" -version '5.0.203'
    """)
    withAzureCredentials(path: "${homePath}", credentialsFile: '.credentials.json') {
      withTerraform(){
        body()
      }
    }
  }
}

def cleanupAzureResources(){
    def props = getVaultSecret(secret: 'secret/apm-team/ci/apm-agent-dotnet-azure')
    def authObj = props?.data
    def dockerCmd = "docker run --rm -i -v \$(pwd)/.azure:/root/.azure mcr.microsoft.com/azure-cli:latest"
    withEnvMask(vars: [
        [var: 'AZ_CLIENT_ID', password: "${authObj.client_id}"],
        [var: 'AZ_CLIENT_SECRET', password: "${authObj.client_secret}"],
        [var: 'AZ_SUBSCRIPTION_ID', password: "${authObj.subscription_id}"],
        [var: 'AZ_TENANT_ID', password: "${authObj.tenant_id}"]
    ]) {
        dir("${BASE_DIR}"){
            cmd label: "Create storage for Azure authentication",
                script: "mkdir -p ${BASE_DIR}/.azure || true"
            cmd label: "Logging into Azure",
                script: "${dockerCmd} az login --service-principal --username ${AZ_CLIENT_ID} --password ${AZ_CLIENT_SECRET} --tenant ${AZ_TENANT_ID}"
            cmd label: "Setting Azure subscription",
                script: "${dockerCmd} az account set --subscription ${AZ_SUBSCRIPTION_ID}"
            cmd label: "Checking and removing any Azure related resource groups",
                script: "for group in `${dockerCmd} az group list --query \"[?name | starts_with(@,'${AZURE_RESOURCE_GROUP_PREFIX}')]\" --out json|jq .[].name --raw-output`;do ${dockerCmd} az group delete --name \$group --no-wait --yes;done"
        }
    }
}

def withTerraform(Closure body){
  def binDir = "${HOME}/bin"
  withEnv([
    "PATH+TERRAFORM=${binDir}"
    ]){
      sh(label:'Install Terraform', script: """
        mkdir -p ${binDir}
        cd ${binDir}
        curl -sSL -o terraform.zip https://releases.hashicorp.com/terraform/0.15.3/terraform_0.15.3_linux_amd64.zip
        unzip terraform.zip
      """)
      body()
  }
}

def release(Map args = [:]){
  def secret = args.secret
  def withSuffix = args.get('withSuffix', false)
  dotnet(){
    sh(label: 'Release', script: ".ci/linux/release.sh ${withSuffix}")
    def repo = getVaultSecret(secret: secret)
    wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [
      [var: 'REPO_API_KEY', password: repo.data.apiKey],
      [var: 'REPO_API_URL', password: repo.data.url],
      ]]) {
      withEnv(["REPO_API_KEY=${repo.data.apiKey}", "REPO_API_URL=${repo.data.url}"]) {
        sh(label: 'Deploy', script: '.ci/linux/deploy.sh ${REPO_API_KEY} ${REPO_API_URL}')
      }
    }
  }
}

def reportTests() {
  dir("${BASE_DIR}"){
    archiveArtifacts(allowEmptyArchive: true, artifacts: 'target/diag-*.log,test/**/junit-*.xml,target/**/Sequence_*.xml,target/**/testhost*.dmp')
    junit(allowEmptyResults: true, keepLongStdio: true, testResults: 'test/**/junit-*.xml')
  }
}

def notifyStatus(def args = [:]) {
  releaseNotification(slackChannel: "${env.SLACK_CHANNEL}",
                      slackColor: args.slackStatus,
                      slackCredentialsId: 'jenkins-slack-integration-token',
                      to: "${env.NOTIFY_TO}",
                      subject: args.subject,
                      body: args.body)
}
