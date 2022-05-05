pipeline {
    agent none
    
    environment {
        SWP_DIR = '../../swp-debian10'

    }
    stages {
        stage ('Latest SWP') {
            agent { 
                label 'local'
            }
            steps {
                script {
                    sh ''' #!/bin/bash
                    ls -Art $SWP_DIR/ | tail -n 1 | awk -F "-" '{print substr($4, 1, length($4)-4)}'
                    '''
                    }
            }
        }
    }
}
