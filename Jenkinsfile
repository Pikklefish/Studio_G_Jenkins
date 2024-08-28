

def NEXUS_LINK = "http://192.168.15.150:8081/repository/createg-snapshot/com/studiog/varodrt/admin/maven-metadata.xml"

def SERVER_LIST = [
    ['192.168.15.170', '/home/peter/deploy'],
    ['192.168.15.170', '/home/peter']
]

node{
    stage('Pull the file off Nexus') {
        withCredentials([usernameColonPassword(credentialsId: '5771e4b4-a87b-4a7b-b536-2a68bdc8caa9', variable: 'NEXUS_CREDENTIALS')]) {
            sh script: """
            curl -u \$NEXUS_CREDENTIALS -o rider.war ${NEXUS_LINK}
            """
        }
    }

    stage('Upload file(s) to server') {
        withCredentials([usernameColonPassword(credentialsId: 'ba0bd89f-af69-40dc-af02-842b82fc02be', variable: 'FTP_CREDENTIALS')]) {

            // Extract the credentials in Groovy
            def ftpUser = "${FTP_CREDENTIALS.split(':')[0]}"
            def ftpPass = "${FTP_CREDENTIALS.split(':')[1]}"

            SERVER_LIST.each { server ->
                sh script: """
                sshpass -p '${ftpPass}' sftp -oBatchMode=no ${ftpUser}@${server[0]} <<EOF
                cd ${server[1]}
                put rider.war
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
