def map = [
    'newsite16' : 'USK@UTbgOlck8yonG-PBz3a0IAr2bo4OANpQIS5l7ujQzDg,Uw4kLUKyu5C~zFnGnYXfB9MsutJCgJBg8Cvf9~0GagI,AQACAAE/newsite16/0/'
    ]

node {
  deleteDir()
  docker.image('python:3').withRun('--network=host') { c ->
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
        sh 'PATH="$PATH:$(pwd)/dgof" git clone ' + "freenet::$entry.value $entry.key"
      }
    }
  }
}
