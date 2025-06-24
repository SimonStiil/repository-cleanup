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
                        [key: 'repository_short', value: '$.repository.name'],
                        [key: 'pull_request_action', value: '$.action'],
                        [key: 'pull_request_number', value: '$.number']
                    ],
                    genericHeaderVariables: [
                        [key: 'x-github-event']
                    ],
                    causeString: 'Triggered on $branch ($repository)',
                    tokenCredentialId: 'jenkins-webhook-repo-cleanup',
                    printContributedVariables: true,
                    printPostContent: true,
                    silentResponse: false
                ]
            ]),
            parameters([
                string(
                    defaultValue: 'scriptcrunch', 
                    name: 'STRING-PARAMETER', 
                    trim: true
                )
            ])
])
podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: curl
        image: curlimages/curl:8.14.1
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
        currentBuild.description = repository + "\n" + x_github_event+"."+pull_request_action
      } catch(all) {
        currentBuild.description = repository + "\n" + x_github_event
      }
      try{
        if (x_github_event != "delete" && (x_github_event == "pull_request" && pull_request_action != "closed")) {
          echo "Skipping branch: " + branch
          return
        }
      } catch(all) {
      }
    // https://docs.github.com/en/webhooks/webhook-events-and-payloads#delete
        container('curl') {
          Map props
          stage('Get main Docker env'){
            httpRequest outputFile: 'package.env', 
                url: "https://raw.githubusercontent.com/${repository}/main/package.env",
                consoleLogResponseBody: false,
                quiet: true,
                wrapAsMultipart: false
            props = readProperties file: 'package.env'
            try{
              currentBuild.description = props.PACKAGE_DESTINATION.split("/")[0] + "\n" + repository + "\n" + x_github_event+"."+pull_request_action
            } catch(all) {
              try{
                currentBuild.description = props.PACKAGE_DESTINATION.split("/")[0] + "\n" + repository + "\n" + x_github_event
              } catch(any) {
              }
            }
            try{
              props.put("repository", repository)
              props.put("repository_short", repository_short)
            } catch(all) {
              echo "Warning: repository or repository_short missing"
            }
            try{
              if (x_github_event == "pull_request" && pull_request_action == "closed"){
                props.put("branch", "PR-"+pull_request_number)
                props.put("github_event", "delete")
                
              } else {
                try{
                  props.put("branch", branch)
                } catch(all) {
                  echo "Info: branch missing"
                }
                try{
                  props.put("branchType", branchType)
                } catch(all) {
                  echo "Info: branchType missing"
                }
                try{
                  props.put("github_event", x_github_event)
                } catch(all) {
                  echo "Warning: x_github_event missing"
                  return
                }
              }
            } catch(all) {
              echo "Warning: " + all.toString()
            }
            
            echo "Properties: " + props.toString()
          }
          stage('Cleanup Packages'){
            if (props.PACKAGE_DESTINATION.contains("docker.io")){
              echo "docker.io cleanup"
              dockerhubCleanupTags props
            }
            if (props.PACKAGE_DESTINATION.contains("ghcr.io")){
              echo "ghcr.io cleanup"
              githubPackageCleanupVersions props
            }
          }
        }
    } else {
          echo "not started by Webhook"
    }
  }
}