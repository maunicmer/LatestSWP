pipeline {
    agent none
    options { 
        disableConcurrentBuilds() 
        timeout(time: 60, unit: 'MINUTES')
    }
    environment {
        SWP_DIR = '../../assets/swp-debian10'
        SAFEWALKTEMPLATE = '128'
        GATEWAYTEMPLATE = '147'
        SAFEWALKSRVVMID = '8022'
        SAFEWALKGWVMID = '8023'
        SAFEWALKSRVIP = '192.168.140.70'
        SAFEWALKGWIP = '192.168.140.71'
        SAAS_URL = 'https://saas.dev.safewalk.info/swp-debian10'
        RUN_SYSTEM_UPDATE = 'yes'
        RUN_SYSTEM_ROLLBACK = 'no'
        WORDPRESS_URL = 'wordpress.qa.safewalk.info'
        GATEWAY_URL = 'sfgwlatestswp.dev.safewalk.info'
        
    }
    stages {
        stage ('Getting latest SWP version') {
            agent { 
                label 'local'
            }
            steps {
                script {
                    env.LATESTSWP=sh(returnStdout: true, 
                    script: '''  ls -Art $SWP_DIR/ | tail -n 1 | awk -F "-" '{print substr($4, 1, length($4)-4)}' ''').trim()
                    echo "Latest SWP version found ${LATESTSWP}"
                    
                }
            }
        }
        stage ('Building the environment'){
            agent { 
                label 'local'
            }
                steps {
                    script {
                    sh """ #!/bin/bash
                        echo "Cloning Safewalk Server"
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm clone ${SAFEWALKTEMPLATE} ${SAFEWALKSRVVMID} --name SWPLT-QA-STL-Safewalk-Srv --full 1'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm set ${SAFEWALKSRVVMID} --ipconfig0 ip=${SAFEWALKSRVIP}/24,gw=192.168.140.1'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm set ${SAFEWALKSRVVMID} --sshkeys jenkins.pub'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm set ${SAFEWALKSRVVMID} --memory 4096 --balloon 0'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm start ${SAFEWALKSRVVMID}'
                        sleep 5
                        echo "Cloning Safewalk Gateway"
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm clone ${GATEWAYTEMPLATE} ${SAFEWALKGWVMID} --name SWPLT-QA-STL-Safewalk-Gw --full 1'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm set ${SAFEWALKGWVMID} --ipconfig0 ip=${SAFEWALKGWIP}/24,gw=192.168.140.1'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm set ${SAFEWALKGWVMID} --sshkeys jenkins.pub'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm set ${SAFEWALKGWVMID} --memory 1024 --balloon 0'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm start ${SAFEWALKGWVMID}'
                        sleep 60
                        """
                    }
                }
            }
        stage ('Updating packages'){
            agent { 
                label 'local'
            }
                steps {
                    script {
                    sh """ #!/bin/bash
                        ssh -o StrictHostKeyChecking=no root@${SAFEWALKSRVIP} echo > /root/.ssh/known_hosts
                        ssh -o StrictHostKeyChecking=no root@${SAFEWALKSRVIP} rm /etc/apt/sources.list.d/*.bk.*
                        ssh -o StrictHostKeyChecking=no root@${SAFEWALKSRVIP} sed -i '/turnkey/d' /etc/apt/sources.list.d/security.sources.list
                        ssh -o StrictHostKeyChecking=no root@${SAFEWALKSRVIP} 'apt-get update && apt-get -y upgrade'
                        ssh -o StrictHostKeyChecking=no root@${SAFEWALKGWIP} 'apt-get update && apt-get -y upgrade'
                        """
                    }
                }
        }
        stage('Running Safewalk post-upgrade Tests') {
            agent { 
                label 'windows'
            }
            steps {
                echo 'Running..'
                dir ("C:/Users/Altipeak/Desktop/resources"){
                powershell """
                    (Get-Content ./config.properties -Raw)  -replace 'baseUrl=.*','baseUrl=https\\://${SAFEWALKSRVIP}' | Set-Content -Path ./config.properties
                    (Get-Content ./config.properties -Raw)  -replace 'download-site-d10=.*','download-site-d10=${SAAS_URL}' | Set-Content -Path ./config.properties
                    (Get-Content ./config.properties -Raw)  -replace 'run-system-update=.*','run-system-update=${RUN_SYSTEM_UPDATE}' | Set-Content -Path ./config.properties
                    (Get-Content ./config.properties -Raw)  -replace 'run-system-rollback=.*','run-system-rollback=${RUN_SYSTEM_ROLLBACK}' | Set-Content -Path ./config.properties
                    (Get-Content ./config.properties -Raw)  -replace 'gateway_ssh=.*','gateway_ssh=${SAFEWALKGWIP}' | Set-Content -Path ./config.properties
                    (Get-Content ./config.properties -Raw)  -replace 'gateway_url=.*','gateway_url=${GATEWAY_URL}' | Set-Content -Path ./config.properties
                    (Get-Content ./config.properties -Raw)  -replace 'gateway=.*','gateway=${GATEWAY_URL}' | Set-Content -Path ./config.properties
                    (Get-Content ./config.properties -Raw)  -replace 'saml=.*','saml=${WORDPRESS_URL}' | Set-Content -Path ./config.properties
                 """
                }
                dir ("C:/Users/Altipeak/Desktop/"){
                bat "java -jar SafewalkSWPTests.jar ${env.LATESTSWP}"
                }
            }
         }
        stage ('CheckLog') {
            agent { 
                label 'local'
            }
            steps {
                script {
                    sh ''' #!/bin/bash
                    DIR="../../jobs/LatestSWP-debian10/builds/${BUILD_NUMBER}/log"
                    FAILURES=$(cat $DIR | grep Failures: | awk -F [" | ,"] '{print $7}'  | grep -v 0 )
                    TEST=$(cat $DIR | grep Exception | awk -F ':' '{print $1}' )
                    if  [[ -z "$TEST" ]] && [[ -z "$FAILURES" ]];
                        then
                            echo "No failures found - Updating packages list"
                            ssh -o StrictHostKeyChecking=no root@192.168.140.70 "dpkg -l |grep ii" | awk  '{print $2 " " $3}' >> latest_packages.txt
                            scp latest_packages.txt altipeak@192.168.140.12:/home/altipeak/nginx-static/assests/standalone/latest_packages_debian10.txt
                        else
                            echo "Test failure found - Packages list will not be updated"
                        fi
                    '''
                    }
                }
            }
        stage ('Clean Up'){
            agent { 
                label 'local'
            }
                steps {
                    script {
                    sh """ #!/bin/bash
                        echo "Cleaning Up VMs"
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm stop ${SAFEWALKSRVVMID}'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm destroy ${SAFEWALKSRVVMID}'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm stop ${SAFEWALKGWVMID}'
                        ssh -o StrictHostKeyChecking=no root@192.168.140.1 'qm destroy ${SAFEWALKGWVMID}'
                    """
                    }
                }
            }
    }
}
