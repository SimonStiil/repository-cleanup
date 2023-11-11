@Library('pipeline-library')
import static dk.stiil.pipeline.Constants.*

properties([buildDiscarder(
                logRotator(artifactDaysToKeepStr: '5', artifactNumToKeepStr: '5', daysToKeepStr: '', numToKeepStr: '10')
            ), disableConcurrentBuilds(),
            [$class: 'GithubProjectProperty', displayName: '',
              projectUrlStr: 'https://github.com/SimonStiil/repository-cleanup/'],
            durabilityHint('MAX_SURVIVABILITY'),
            pipelineTriggers([
                [$class: 'GenericTrigger',
                    genericVariables: [
                        [key: 'branch', value: '$.ref'],
                        [key: 'branchType', value: '$.ref_type'],
                        [key: 'repository', value: '$.repository.full_name'],
                        [key: 'repository_short', value: '$.repository.name']
                    ],
                    genericHeaderVariables: [
                        [key: 'x-github-event']
                    ],
                    causeString: 'Triggered on $branch ($repository)',
                    tokenCredentialId: 'generic-webhook-token',
                    printContributedVariables: true,
                    printPostContent: true,
                    silentResponse: false
                ]
            ])
])
podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: curl
        image: curlimages/curl:8.4.0
        command:
        - sleep
        args: 
        - 99d
      restartPolicy: Never
''') {
  node(POD_LABEL) {
    stage('setup') {  
      checkout scm
      setupKubernetesSA
    }
    if ( currentBuild.getBuildCauses("org.jenkinsci.plugins.gwt.GenericCause").size() > 0) {
      try{
        currentBuild.description = x_github_event
        if (x_github_event != "push" && x_github_event != "delete") {
          return
        }
      } catch(all) {
      }
    // https://docs.github.com/en/webhooks/webhook-events-and-payloads#delete
        container('curl') {
          Map props
          stage('Get main Docker env'){
            httpRequest outputFile: 'package.env', url: "https://raw.githubusercontent.com/${repository}/main/package.env"
            props = readProperties file: 'package.env'
            
            try{
              props.put("github_event", x_github_event)
            } catch(all) {
              println "Warning: x_github_event missing"
            }
            
            try{
              props.put("repository", repository)
              props.put("repository_short", repository_short)
            } catch(all) {
              println "Warning: repository or repository_short missing"
            }
            
            try{
              props.put("branch", branch)
            } catch(all) {
              println "Warning: branch missing"
            }
            
            try{
              props.put("branchType", branchType)
            } catch(all) {
              println "Info: branchType missing"
            }
          }
          stage('Cleanup Packages'){
            githubPackageCleanupVersions props
          }
        }
    } else {
          println "not started by Webhook"
    }
  }
}