node('master'){
    stage ('Pull Source Code'){
    checkout scm
     //   git branch: 'jenkins2-course', 
     //       url: 'https://github.com/g0t4/solitaire-systemjs-course'
    }
    // pull dependencies from npm
    // on windows use: bat 'npm install'
    stage('Installing NPM module'){
        sh 'npm install'
    }
    // stash code & dependencies to expedite subsequent testing
    // and ensure same code & dependencies are used throughout the pipeline
    // stash is a temporary archive
    
    stage('Stash everything'){
        stash name: 'everything', 
            excludes: 'test-results/**', 
            includes: '**'
    }    
    // test with PhantomJS for "fast" "generic" results
    // on windows use: bat 'npm run test-single-run -- --browsers PhantomJS'
    
    stage('Running tests'){
        sh 'npm run test-single-run -- --browsers PhantomJS'
    }
    // archive karma test results (karma is configured to export junit xml files)
    stage('Archiving Test Results'){
        step([$class: 'JUnitResultArchiver', 
            testResults: 'test-results/**/test-results.xml'])
    }
    stage('GZIP App folder contents'){
        sh 'tar -cvzf /Users/jas/.jenkins/workspace/solitaire-pipeline/app.tgz /Users/jas/.jenkins/workspace/solitaire-pipeline/app/'
    }
    stage('Uploading Artifact'){
    nexusArtifactUploader artifacts: [[artifactId: 'solitaire_app_folder', 
    classifier: 'java_app_folder', 
    file: '/Users/jas/.jenkins/workspace/solitaire-pipeline/app.tgz', 
    type: 'tgz']], 
    credentialsId: 'nexusadmin', 
    groupId: 'Solitaire_Pipeline', 
    nexusUrl: 'localhost:8081/', 
    nexusVersion: 'nexus3', 
    protocol: 'http', 
    repository: 'jenkins-nexus', 
    version: '${BUILD_NUMBER}'
    }
}
node('mac'){
    stage('Removing folder contents'){
        sh 'ls -la'
        sh 'rm -rf *'
        sh 'ls -la'
    }
    stage('Unstashing Everything'){
        unstash 'everything'
    }
}

// defining parallel integration tests
stage 'Browser Testing'
parallel chrome: {
    runTests("Chrome")
}, firefox: {
    runTests("Firefox")
}, safari: {
    runTests("Safari")
}
def runTests(browser) {
    node {
        sh 'ls -la'
        sh 'rm -rf *'
        sh 'ls -la'
        unstash 'everything'
        sh "npm run test-single-run -- --browsers ${browser}"
        step([$class: 'JUnitResultArchiver', 
            testResults: 'test-results/**/test-results.xml'])
    }
}

def notify(status){
    emailext (
      to: "wesmdemos@gmail.com",
      subject: "${status}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
      body: """<p>${status}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
        <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
    )
}

node {
    notify("Deploy to Prod?")
}

input 'Please approve deployment'

// limit concurrency so we dont perform simultaneous deploys
// and if multiple pipelines are executing,
// newest is only one that will be allowed through, the rest are cancelled

stage name: 'Deploy to Staging', concurrency: 1
node {
    // write build number to the index page so I can see update
    sh "echo '<h1>${env.BUILD_DISPLAY_NAME}' >> app/index.html"
    // deploy to a docker container mapped to port 3000
    sh 'docker-compose up -d --build'

    notify 'Solitaire deployed to a docker container on :3000'
}
