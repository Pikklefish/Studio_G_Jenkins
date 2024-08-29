

def NEXUS_LINK = "http://192.168.15.150:8081/repository/createg-snapshot/com/studiog/varodrt/admin/maven-metadata.xml"

def SERVER_LIST = [
    ['192.168.15.170', '/home/peter/deploy'],
    ['192.168.15.170', '/home/peter']
]

node{
    stage('Pull the file off Nexus') {
        withCredentials([usernameColonPassword(credentialsId: 'ee15f893-1728-450b-9556-2647a12f856d', variable: 'NEXUS_CREDENTIALS')]) {
            sh script: """
            curl -u \$NEXUS_CREDENTIALS -o TEST_8932.war ${NEXUS_LINK}
            """
        }
    }

    stage('Upload file(s) to server') {
        withCredentials([usernameColonPassword(credentialsId: '0ada22b4-bceb-47b2-91aa-c4d225dc1fe6', variable: 'FTP_CREDENTIALS')]) {

            // Extract the credentials in Groovy
            def ftpUser = "${FTP_CREDENTIALS.split(':')[0]}"
            def ftpPass = "${FTP_CREDENTIALS.split(':')[1]}"

            SERVER_LIST.each { server ->
                sh script: """
                sshpass -p '${ftpPass}' sftp -oBatchMode=no ${ftpUser}@${server[0]} <<EOF
                cd ${server[1]}
                put TEST_8932.war
                EOF
                """
            }
        }
    }
}

// pipeline {
//     agent { 
//         node {
//             label 'docker-agent-python'
//         }
//     }
//     triggers {
//         pollSCM '*/5 * * * * *'
//     }
//     stages {
//         stage('Build') {
//             steps {
//                 echo "Building.."
//                 sh '''
//                 cd myapp
//                 pip install -r requirements.txt
//                 '''
//             }
//         }
//         stage('Test') {
//             steps {
//                 echo "Testing.."
//                 sh '''
//                 cd myapp
//                 python3 hello.py
//                 python3 hello.py --name=Brad
//                 '''
//             }
//         }
//         stage('Deliver') {
//             steps {
//                 echo 'Deliver....'
//                 sh '''
//                 echo "doing delivery stuff.."
//                 '''
//             }
//         }
//     }
// }
