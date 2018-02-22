project = "conan-external-libs"

conan_remote = "ess-dmsc-local"
conan_user = "ess-dmsc"

images = [
  'centos7': [
    'name': 'essdmscdm/centos7-build-node:1.0.1',
    'sh': 'sh'
  ],
  'centos7-gcc6': [
    'name': 'essdmscdm/centos7-gcc6-build-node:1.0.0',
    'sh': '/usr/bin/scl enable rh-python35 devtoolset-6 -- /bin/bash'
  ],
  'debian9': [
  'name': 'essdmscdm/debian9-build-node:1.0.0',
  'sh': 'sh'
  ],
  'fedora25': [
    'name': 'essdmscdm/fedora25-build-node:1.0.0',
    'sh': 'sh'
  ],
  'ubuntu1604': [
    'name': 'essdmscdm/ubuntu16.04-build-node:2.0.0',
    'sh': 'sh'
  ],
  'ubuntu1710': [
    'name': 'essdmscdm/ubuntu17.10-build-node:1.0.0',
    'sh': 'sh'
  ]
]

base_container_name = "${project}-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

def get_pipeline(image_key) {
  return {
    node('docker') {
      def container_name = "${base_container_name}-${image_key}"
      try {
        def image = docker.image(images[image_key]['name'])
        def custom_sh = images[image_key]['sh']
        def container = image.run("\
          --name ${container_name} \
          --tty \
          --cpus=2 \
          --memory=4GB \
          --network=host \
          --env http_proxy=${env.http_proxy} \
          --env https_proxy=${env.https_proxy} \
          --env local_conan_server=${env.local_conan_server} \
        ")

        stage("${image_key}: Checkout") {
          sh """docker exec ${container_name} ${custom_sh} -c \"
            git clone \
              --branch ${env.BRANCH_NAME} \
              https://github.com/ess-dmsc/${project}.git
          \""""
        }  // stage

        stage("${image_key}: Conan setup") {
          withCredentials([
            string(
              credentialsId: 'local-conan-server-password',
              variable: 'CONAN_PASSWORD'
            )
          ]) {
            sh """docker exec ${container_name} ${custom_sh} -c \"
              set +x
              conan remote add \
                --insert 0 \
                ${conan_remote} ${local_conan_server} && \
              conan user \
                --password '${CONAN_PASSWORD}' \
                --remote ${conan_remote} \
                ${conan_user} \
                > /dev/null
            \""""
            
            // temporary, until remote is added to container images
            sh """docker exec ${container_name} ${custom_sh} -c \"
              conan remote add \
                vthiery https://api.bintray.com/conan/vthiery/conan-packages
            \""""
          }  // withCredentials
        }  // stage

        stage("${image_key}: Package") {
          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install zlib/1.2.11@conan/stable \
              --settings build_type=Release \
              --options zlib:shared=False \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install zlib/1.2.11@conan/stable \
              --settings build_type=Release \
              --options zlib:shared=True \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install asio/1.11.0@bincrafters/stable \
              --build=outdated
          \""""
          
          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install jsonformoderncpp/3.1.0@vthiery/stable \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install gtest/1.8.0@conan/stable \
              --settings build_type=Release \
              --options gtest:shared=False \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install gtest/1.8.0@conan/stable \
              --settings build_type=Release \
              --options gtest:shared=True \
              --build=outdated
          \""""

          // There is only one cmake_installer package.
          if (image_key == 'centos7') {
            sh """docker exec ${container_name} ${custom_sh} -c \"
              conan install cmake_installer/3.10.0@conan/stable \
                --build=outdated
            \""""
          }
        }  // stage

        stage("${image_key}: Upload") {
          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan upload --confirm --all --remote ${conan_remote} '*'
          \""""
        }  // stage
      } finally {
        sh "docker stop ${container_name}"
        sh "docker rm -f ${container_name}"
      }  // finally
    }  // node
  }  // return
}  // def

def get_macos_pipeline() {
  return {
    node('macos') {
      cleanWs()
      dir("${project}") {
        stage("macOS: Checkout") {
          checkout scm
        }  // stage

        stage("macOS: Conan setup") {
          withCredentials([
            string(
              credentialsId: 'local-conan-server-password',
              variable: 'CONAN_PASSWORD'
            )
          ]) {
            sh "conan user \
                --password '${CONAN_PASSWORD}' \
                --remote ${conan_remote} \
                ${conan_user} \
                > /dev/null"
          }  // withCredentials
        }  // stage

        stage("macOS: Package") {
          sh "conan install zlib/1.2.11@conan/stable \
              --settings build_type=Release \
              --options zlib:shared=False \
              --build=outdated"

          sh "conan install zlib/1.2.11@conan/stable \
              --settings build_type=Release \
              --options zlib:shared=True \
              --build=outdated"

          sh "conan install gtest/1.8.0@conan/stable \
              --settings build_type=Release \
              --options gtest:shared=False \
              --build=outdated"

          sh "conan install gtest/1.8.0@conan/stable \
              --settings build_type=Release \
              --options gtest:shared=True \
              --build=outdated"
        }  // stage

        stage("macOS: Upload") {
          sh "conan upload --confirm --all --remote ${conan_remote} \
            zlib/1.2.11@conan/stable"

          sh "conan upload --confirm --all --remote ${conan_remote} \
            gtest/1.8.0@conan/stable"
        }
      }
    }  // node
  }  // return
}  // def


node {
  checkout scm

  def builders = [:]
  for (x in images.keySet()) {
    def image_key = x
    builders[image_key] = get_pipeline(image_key)
  }
  builders['macOS'] = get_macos_pipeline()
  parallel builders

  // Delete workspace when build is done.
  cleanWs()
}
