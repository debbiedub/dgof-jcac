// The common preparation
// 1. Install pyFreenet3 (could be prepared in the docker image used)
// 2. "install" dgof     (could be prepared in the docker image used)
//
// The logic for every repo
// 1. clone and throw the clone away
// 2. pull from github
// 3. Push to freenet
// 4. If the clone didn't work, reinsert. The reinsert will abort if pushing.
//
// Fetch-URI in the script. Private-URI in the freesitemgr config.
//
// The Organization plugin requires the Jenkinsfile to be in the repo.
// I will work with stages instead.

def map = [
    'plugin-Spider' : 'USK@UTbgOlck8yonG-PBz3a0IAr2bo4OANpQIS5l7ujQzDg,Uw4kLUKyu5C~zFnGnYXfB9MsutJCgJBg8Cvf9~0GagI,AQACAAE/newsite16/1/',
    ]

node ('debbies') {
  deleteDir()
  docker.image('python:3').inside("--network=host --env HOME='${env.WORKSPACE}'") {
    withPythonEnv('python3') {
      stage('Get pyFreenet3') {
        sh 'pip3 install pyFreenet3'
      }
      stage('Get dgof') {
        sh '''
          if git clone http://localhost:8888/freenet:USK@nrDOd1piehaN7z7s~~IYwH-2eK7gcQ9wAtPMxD8xPEs,y61pkcoRy-ccB7BHvLCzt3RUjeMILf8ox26NKvPZ-jk,AQACAAE/dgof/22/ dgof 2> gitclone.out
          then
            cat gitclone.out 1>&2
          else
            cp gitclone.out newusk
            sed -i '$s/.*USK@/USK@/p;d' newusk
            sed -i 's,\\(/dgof/[0-9]*/\\).*,\\1,' newusk
            git clone http://localhost:8888/freenet:$(cat newusk) dgof
          fi
          '''
      }

      map.each { entry ->
        stage(entry.key) {
          catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {      
            def cloneFailed = sh returnStatus: true, script: 'PATH="$PATH:$(pwd)/dgof" git clone ' + "freenet::$entry.value newclone$entry.key && rm -rf newclone$entry.key"
            sh "if [ -d $entry.key ]; then ( cd $entry.key && git fetch origin && git push freenet ); else git clone https://gitlab.com/freenet/$entry.key $entry.key --mirror; done"
            if (cloneFailed != 0) {
              sh "freesitemgr reinsert $entry.key"
              sh 'exit 1' // The clone failed
            }
          }
        }
      }
    }
  }
}
