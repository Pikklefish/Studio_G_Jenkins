// node {
//     Nexus_REPO_LINK = "http://192.168.15.150:8081/repository/createg-snapshot/com/doub/drt3/driver-api/1.2.05-SNAPSHOT/driver-api-1.2.05-20240819.142925-1.war"
//     FILE_NAME = "test2.war"
//     stage('Pull the file off Nexus') {
//         withCredentials([usernameColonPassword(credentialsId: '5771e4b4-a87b-4a7b-b536-2a68bdc8caa9', variable: 'NEXUS_CREDENTIALS')]) {
//             sh script: '''
//             curl -u ${NEXUS_CREDENTIALS} -o ${FILE_NAME}  ${NEXUS_REPO_LINK}
//             '''
//         }
//     }
//     stage('Upload file(s) to server') {
//         withCredentials([usernameColonPassword(credentialsId: 'ba0bd89f-af69-40dc-af02-842b82fc02be', variable: 'FTP_CREDENTIALS')]) {
//             sh script: '''
//             # Extract username and password
//             USERNAME=$(echo $FTP_CREDENTIALS | cut -d: -f1)
//             PASSWORD=$(echo $FTP_CREDENTIALS | cut -d: -f2)

//             # Use sshpass with sftp to connect and perform actions
//             sshpass -p "$PASSWORD" sftp -oBatchMode=no $USERNAME@192.168.15.170 <<EOF
//             cd /home/peter/deploy
//             put ${FILE_NAME}
//             EOF
//             '''
//         }
//     }
// }

def NEXUS_LINK = "http://192.168.15.150:8081/repository/createg-snapshot/com/studiog/varodrt/admin/maven-metadata.xml"
def SAVE_FILE_NAME = "test_4.xml"
def SFTP_SERVERS = ['192.168.15.170'] 

node {
    
    stage('Pull the file off Nexus') {
        withCredentials([usernameColonPassword(credentialsId: '5771e4b4-a87b-4a7b-b536-2a68bdc8caa9', variable: 'NEXUS_CREDENTIALS')]) {
            sh script:"""
            curl -u ${NEXUS_CREDENTIALS} -o ${SAVE_FILE_NAME} ${NEXUS_LINK}
            """
        }
    }
    stage('Upload file(s) to server') {
            withCredentials([usernameColonPassword(credentialsId: 'ba0bd89f-af69-40dc-af02-842b82fc02be', variable: 'FTP_CREDENTIALS')]) {

            // Extract the credentials in Groovy
            def ftpUser = "${FTP_CREDENTIALS.split(':')[0]}"
            def ftpPass = "${FTP_CREDENTIALS.split(':')[1]}"

            // Upload the file using sshpass with sftp
            sh script: """
            sshpass -p ${ftpPass} sftp -oBatchMode=no ${ftpUser}@192.168.15.170 <<EOF
            cd /home/peter/deploy
            put ${SAVE_FILE_NAME}
            EOF
            """
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
