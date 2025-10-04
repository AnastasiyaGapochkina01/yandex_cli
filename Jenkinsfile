def yc = '/var/lib/jenkins/yc'

pipeline {
  agent any

  parameters {
    choice(name: 'IMAGE', choices: ['ubuntu-2404-lts-oslogin', 'debian-12'])
    string(name: 'CPU', defaultValue: '2')
    string(name: 'MEM', defaultValue: '4')
    string(name: 'DISK_SIZE', defaultValue: '20')
    string(name: 'VM_NAME', defaultValue: 'test-vm')
  }

  environment {
    YC_FOLDER_ID = 'b1gdge57rslfb323otnm'
    YC_CLOUD_ID = 'b1glh5elut8uibsdt45f'
    SA_KEY = credentials('iam-key')
  }

  stages {
    stage('Setup profile') {
      steps {
        script {
          sh """
            ${yc} config profile create sa-profile || true
            ${yc} config set folder-id ${env.YC_FOLDER_ID}
            ${yc} config set cloud-id ${env.YC_CLOUD_ID}
            ${yc} config set service-account-key ${env.SA_KEY}
            ${yc} config profile activate sa-profile
          """
        }
      }
    }

    stage('Create VM') {
      steps {
        script {
          def vmName = params.VM_NAME
          def image = params.IMAGE
          def cpu = params.CPU
          def mem = params.MEM
          def diskSize = params.DISK_SIZE

          sh """
            ${yc} compute instance create \
            --name $vmName \
            --zone ru-central1-b \
            --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
            --create-boot-disk image-folder-id=standard-images,image-family=$image,size=$diskSize \
            --memory $mem \
            --cores $cpu \
            --metadata-from-file user-data=metadata.yaml
          """
          timeout(time: 5, unit: 'MINUTES') {
            waitUntil {
              def status = sh(script: "${yc} compute instance get --name $vmName --format json | jq -r '.status'", returnStdout: true).trim()
              echo "VM status ${status}"
              return (status == 'RUNNING')
            }
          }
        }
      }
    }

    stage('Run ansible') {
      steps {
        script {
        def vmPubIp = sh(
          script: "${yc} compute instance get --name ${params.VM_NAME} --format json | jq -r '.network_interfaces[0].primary_v4_address.one_to_one_nat.address'", 
          returnStdout: true
        ).trim()
        echo "VM public IP: $vmPubIp"

        timeout(time: 5, unit: 'MINUTES') {
          waitUntil {
            def sshAvailable = sh(
              script: "ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 jenkins@${vmPubIp} echo 1",
              returnStatus: true
            ) == 0
            if(!sshAvailable) {
              echo "SSH not available yet"
            }
            return sshAvailable
          }
        }
        echo "SSH available"

        build(
          job: 'apply-ansible',
          parameters: [
            string(name: 'HOST', value: vmPubIp),
            booleanParam(name: 'DRY_RUN', value: true)
          ],
          wait: true
        )
        }
      }
    }
  }
}
