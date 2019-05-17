@Library('pipeline-library@master')
import com.genesys.jenkins.Service
import groovy.json.JsonSlurperClassic

properties([
        parameters([
                string(name: 'SERVICE_NAME', description: 'The service project to build', defaultValue: "${currentBuild.projectName}", trim: true),
                string(name: 'BRANCH', description: 'The branch to build', defaultValue: 'master', trim: true)
        ]),
])

node('dev_mesos||dev_shared') {
    final String gitBase = 'git@bitbucket.org'
    final String gitOwner = 'inindca'
    final String gitUserName = 'PureCloud Jenkins'
    final String gitUserEmail = 'purecloud-jenkins@ininica.com'
    final String bitbucketCredentials = '7c2a5698-a932-447a-9727-6852d0994ea0'
    final String branch = getBranch()
    final boolean isRelease = isRelease(branch)
    final String release = (isRelease ? env.BUILD_NUMBER : '0-SNAPSHOT')
    final String jdkVersion = 'Oracle JDK 8 (latest)'
    final String mvnVersion = 'Maven 3.6.0'
    final def mvnHome = tool name: mvnVersion, type: 'hudson.tasks.Maven$MavenInstallation'
    final def mvnSettings = '/var/build/settings.xml'
    final def javaHome = tool name: jdkVersion, type: 'hudson.model.JDK'
    final String ecrRepoName = currentBuild.projectName
    final String region = 'us-east-1'
    final serviceBuild = new com.genesys.jenkins.Service()

    stage('clean-workspace') {
        cleanWs()
    }

    def common
    stage('init-common') {
        // check out inin-docker repo to gain access to reusable utilities
        dir('commonlib') {
            git branch: 'master', credentialsId: bitbucketCredentials, url: 'git@bitbucket.org:inindca/inin-docker.git'
            common = load 'common/Jenkinsfile'
        }
    }

    def scmVars
    stage('checkout') {
        dir('src') {
            scmVars = git url:"${gitBase}:${gitOwner}/${params.SERVICE_NAME}",
                    branch: "${branch}",
                    credentialsId: bitbucketCredentials
        }
    }

    stage('prepare-environment') {
        // initialize git parameters, necessary to allow us to create a new tag for the release
        dir('src') {
            sh "git config user.name \"${gitUserName}\""
            sh "git config user.email \"${gitUserEmail}\""
        }
    }

    stage('build-artifact') {
        final String mvnGoals
        if (isRelease) {
            mvnGoals = "-U -B clean deploy scm:tag -Drevision=${release}"
        } else {
            mvnGoals = "-U -B clean verify"
        }
        dir('src') {
            try {
                withEnv(["JAVA_HOME=${javaHome}", "PATH=${mvnHome}/bin:${javaHome}/bin:${env.PATH}"]) {
                    sh "mvn -s ${mvnSettings} ${mvnGoals}"
                }
            } finally {
                junit allowEmptyResults: true, testResults: '**/target/*-reports/*.xml'
            }
        }
    }

    Map ecrRepoInfo
    stage('prepare-repo') {
        final createOrDescribeRepositoryBuild = build job: 'docker-create-or-describe-repository', parameters: [string(name: 'REPOSITORY_NAME', value: ecrRepoName), string(name: 'BRANCH', value: 'master')]
        copyArtifacts filter: 'repositoryInfo.json', projectName: 'docker-create-or-describe-repository', selector: specific("${createOrDescribeRepositoryBuild.getNumber()}")
        ecrRepoInfo = new JsonSlurperClassic().parseText(readFile('repositoryInfo.json'))
        common.ecrLogin(region, [ecrRepoInfo.registryId])
    }

    final String commitHash
    final String localRevisionTag
    final String localReleaseTag
    final String localLatestTag
    stage('prepare-local-docker-tags') {
        commitHash = scmVars.GIT_COMMIT
        localRevisionTag = "${ecrRepoName}:${commitHash}"
        localReleaseTag = "${ecrRepoName}:${release}.RELEASE"
        localLatestTag = "${ecrRepoName}:latest"
    }

    stage('build-container') {
        // build and tag the docker container
        dir('src') {
            sh "docker build -t ${localRevisionTag} ."
            sh "docker tag ${localRevisionTag} ${localReleaseTag}"
            sh "docker tag ${localRevisionTag} ${localLatestTag}"
        }
    }

    stage('describe-build') {
        currentBuild.description = "<li>branch='${branch}'</li><li>release='${release}'</li>"
    }

    stage('prepare-deploy-files') {
        // prepare the deploy files
        sh "cp -R src/deploy ."
        sh "mkdir -p ./deploy/infra/cloudformation.d"
        serviceBuild.preBuild(bitbucketCredentials)
    }

    final String remoteRevisionTag
    final String remoteReleaseTag
    final String remoteLatestTag
    stage('prepare-remote-docker-tags') {
        if ( isRelease) {
            remoteRevisionTag = "${ecrRepoInfo.repositoryUri}:${commitHash}"
            remoteReleaseTag = "${ecrRepoInfo.repositoryUri}:${release}.RELEASE"
            remoteLatestTag = "${ecrRepoInfo.repositoryUri}:latest"
        }
    }

    String imageDigest = null
    stage('push-container') {
        if ( isRelease) {
            // apply various tags to the container and push to aws ecr
            sh "docker tag ${localRevisionTag} ${remoteRevisionTag}"
            sh "docker tag ${localRevisionTag} ${remoteReleaseTag}"
            sh "docker tag ${localRevisionTag} ${remoteLatestTag}"
            sh "docker push ${remoteRevisionTag}"
            sh "docker push ${remoteReleaseTag}"
            sh "docker push ${remoteLatestTag}"
            final Map imageDetail = common.describeImage(region, ecrRepoInfo.repositoryName, "${release}.RELEASE", ecrRepoInfo.registryId)
            imageDigest = imageDetail.imageDigest
        }
    }

    final List<String> buildEnv = [
            "WORKSPACE=${pwd()}",
            "AUTO_DEPLOY_TO_DEV=true",
            "AUTO_DEPLOY_TO_TEST=false",
            "CLUSTER_NAME=${SERVICE_NAME}",
            "SERVICE_ROLE=${SERVICE_NAME}",
            "EMAIL_LIST=devops-cloudapplications@genesys.com",
            "CREDENTIALS_ID=${bitbucketCredentials}",
            "DEPLOY_SCHEDULE=always",
            "TEST_JOB=no-tests",
            "AUTO_SUBMIT_CM=true",
            "SKYNET_DEPLOY_TO_PROD=false",
            "GIT_BRANCH=${BRANCH}",
            "REPO=${SERVICE_NAME}.git",
            "ECR_REPO=${ecrRepoInfo.repositoryName}",
            "IMAGE_DIGEST=${imageDigest}",
            "PACKER_ROOT=/opt/packer/packer-1.3.2/",
            "OWNER=DevOps-CloudApplications@genesys.com"
    ]

    stage('post-build') {
        // build ami and prepare build properties for skynet
        withEnv(buildEnv) {
            serviceBuild.postBuild(true, 'infra', 'dev', 'us-east-1', 2)
        }
    }

    stage('bricks-deploy') {
        if (isRelease) {
            // launch deploy
            withEnv(buildEnv) {
                serviceBuild.bricksPostBuild(SERVICE_NAME)
            }
        }
    }
}
private String getBranch() {
    final String defaultBranch = 'master'
    if (params.BRANCH != null && params.BRANCH.length() > 0) {
        echo "Using 'params.BRANCH'='${params.BRANCH}'"
        return params.BRANCH
    }
    if (env.BRANCH != null && env.BRANCH.length() > 0) {
        echo "Using 'env.BRANCH'='${env.BRANCH}'"
        return env.BRANCH
    }
    echo "Using 'defaultBranch'='${defaultBranch}'"
    return defaultBranch
}

private boolean isRelease(final String branch) {
    final String defaultBranch = 'master'
    boolean isRelease = false
    if (defaultBranch == branch) {
        echo "detected '${defaultBranch}' branch"
        isRelease = true
    } else {
        echo "branch is not '${defaultBranch}'"
    }
    echo "'isRelease'='${isRelease}'"
    return isRelease
}
