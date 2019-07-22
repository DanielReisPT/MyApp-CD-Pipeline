pipeline {
  agent any
  parameters {
    // App List Parameters
    string(name: 'ApplicationScope', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy.')
    string(name: 'ApplicationScopeWithTests', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy (including test applications)')
    string(name: 'TriggeredBy', defaultValue: 'N/A', description: 'Name of LifeTime user that triggered the pipeline remotely.')
  }
  options { skipStagesAfterUnstable() }
  environment {
    // Artifacts Folder
    ArtifactsFolder = "Artifacts"

	LifeTimeEnvironmentURL = 'cicd-life.outsystemsonazure.com'
	LifeTimeAPIVersion = 2
	DevelopmentEnvironment = 'Development'
	RegressionEnvironment = 'Regression'
	//AcceptanceEnvironment=<Name of Acceptance environment>
	//PreProductionEnvironment=<Name of Pre-Production environment>
	ProductionEnvironment = 'Production'
	AuthorizationToken=credentials('eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJsaWZldGltZSIsInN1YiI6IllUWTVZakF3WlRVdE5UWXhaQzAwTlRWbExXRmpaVGN0TldNMk5UWTBaREF6T0daaSIsImF1ZCI6ImxpZmV0aW1lIiwiaWF0IjoiMTU2MzU0NzY2NCIsImppdCI6IjdDME9vRTNzRnYifQ==.9XnbZs08Jf7uND9GU/Jm4Fp7JE9kwxnJ7w1sE4ceXjY=')
	ProbeEnvironmentURL = 'https://cicd-test.outsystemsonazure.com'
	BddEnvironmentURL = 'https://cicd-test.outsystemsonazure.com'
  }
  stages {
    stage('Install Python Dependencies') {
      steps {
        echo "Create Artifacts Folder"
        powershell "mkdir ${env.ArtifactsFolder}"
        // Only the virtual environment needs to be installed at the system level
        echo "Install Python Virtual environments"
        powershell 'pip install -q -I virtualenv'
        // Install the rest of the dependencies
        withPythonEnv('python') {
          echo "Install Python requirements"
          powershell 'pip install -U outsystems-pipeline'
        }
      }
    }
    stage('Get Latest Tags') {
      steps {
        withPythonEnv('python') {
          echo "Pipeline run triggered remotely by '${params.TriggeredBy}' for the following applications (including tests): '${params.ApplicationScopeWithTests}'"
          echo 'Retrieving latest application tags from Development environment...'
          powershell "python -m outsystems.pipeline.fetch_lifetime_data --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeEnvironmentURL} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion}"
          echo 'Deploying latest application tags to Regression...'
          powershell "python -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeEnvironmentURL} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.DevelopmentEnvironment}\" --destination_env \"${env.RegressionEnvironment}\" --app_list \"${params.ApplicationScopeWithTests}\""
        }
      }
      post {
        always {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "*_data/*.cache", onlyIfSuccessful: true
          }
        }
        failure {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "DeploymentConflicts"
          }
        }
      }
    }
    stage('Run Regression') {
      steps {
        withPythonEnv('python') {
          echo 'Generating URLs for BDD testing...'
          powershell "python -m outsystems.pipeline.generate_unit_testing_assembly --artifacts \"${env.ArtifactsFolder}\" --app_list \"${params.ApplicationScopeWithTests}\" --cicd_probe_env ${env.ProbeEnvironmentURL} --bdd_framework_env ${env.BddEnvironmentURL}"
          echo "Testing the URLs and generating the JUnit results XML..."
          powershell(script: "python -m outsystems.pipeline.evaluate_test_results --artifacts \"${env.ArtifactsFolder}\"", returnStatus: true)
        }
      }
      post {
        always {
          withPythonEnv('python') {
            echo "Publishing JUnit test results..."
            junit(testResults: "${env.ArtifactsFolder}\\junit-result.xml", allowEmptyResults: true)
            //echo "Sending notifcations to slack..."
            //powershell "python .\\custom_pipeline\\slack\\send_test_results_to_slack.py --artifacts \"${env.ArtifactsFolder}\" --slack_hook ${env.SlackHook} --slack_channel \"${params.SlackChannel}\" --job_name \"${env.JOB_NAME}\" --job_dashboard_url ${env.RUN_DISPLAY_URL}"
          }
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "*_data/*.cache", onlyIfSuccessful: true
          }
        }
      }
    }
    stage('Deploy Production') {
      steps {
        withPythonEnv('python') {
          echo 'Deploying latest application tags to Production...'
          powershell "python -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeEnvironmentURL} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.PreProductionEnvironment}\" --destination_env \"${env.ProductionEnvironment}\" --app_list \"${params.ApplicationScope}\""
        }
      }
      post {
        always {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "*_data/*.cache", onlyIfSuccessful: true
          }
        }
        failure {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "DeploymentConflicts"
          }
        }
      }
    }
  }
  post {
    always { 
      echo 'Deleting artifacts folder content...'
      dir ("${env.ArtifactsFolder}") {
        deleteDir()
      }
    }
  }
}
