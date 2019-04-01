project = "conan-external-libs"

conan_remote = "ess-dmsc-local"
conan_user = "ess-dmsc"

images = [
  'centos7': [
    'name': 'essdmscdm/centos7-build-node:3.6.0',
    'sh': '/usr/bin/scl enable devtoolset-6 -- /bin/bash -e'
  ],
  'debian9': [
    'name': 'essdmscdm/debian9-build-node:2.6.0',
    'sh': 'bash -e'
  ],
  'ubuntu1804': [
    'name': 'essdmscdm/ubuntu18.04-build-node:1.4.0',
    'sh': 'bash -e'
  ],
  'alpine': [
    'name': 'screamingudder/alpine-build-node:1.3.0',
    'sh': 'bash -e'
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
                ${conan_remote} ${local_conan_server}
              conan user \
                --password '${CONAN_PASSWORD}' \
                --remote ${conan_remote} \
                ${conan_user} \
                > /dev/null
            \""""
          }  // withCredentials
        }  // stage

        stage("${image_key}: Package") {

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install ${project}/conanfile.txt \
              --settings build_type=Release \
              --build=outdated
          \""""

          // There is a problem with the boost packages on alpine until we can update
          // to boost 1.67 or above
          if (image_key != "alpine") {
            sh """docker exec ${container_name} ${custom_sh} -c \"
              conan install ${project}/conanfile_boost.txt \
                --settings build_type=Release \
                --build=outdated
            \""""
          }

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install gtest/1.8.0@conan/stable \
              --settings build_type=Release \
              --options gtest:shared=False \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install libcurl/7.56.1@bincrafters/stable \
              --settings build_type=Release \
              --options libcurl:shared=True \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install libcurl/7.56.1@bincrafters/stable \
              --settings build_type=Release \
              --options libcurl:shared=False \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install OpenSSL/1.0.2n@conan/stable \
              --settings build_type=Release \
              --options OpenSSL:shared=False \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install fmt/5.2.0@bincrafters/stable \
              --settings build_type=Release \
              --options fmt:shared=False \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan install zlib/1.2.11@conan/stable \
              --settings build_type=Release \
              --options zlib:shared=False \
              --build=outdated
          \""""

          if (image_key == 'centos7') {
            // There is only one cmake_findboost_modular package.
            sh """docker exec ${container_name} ${custom_sh} -c \"
              conan install cmake_findboost_modular/1.65.1@bincrafters/stable \
                --build=outdated
            \""""

            // There is only one cmake_installer package.
            sh """docker exec ${container_name} ${custom_sh} -c \"
              conan install cmake_installer/3.10.0@conan/stable \
                --build=outdated
            \""""

            // Header-only package.
            sh """docker exec ${container_name} ${custom_sh} -c \"
              conan install jsonformoderncpp/3.1.0@vthiery/stable \
                --build=outdated
            \""""
          } else {
            // boost_log 1.65.1 does not build on CentOS because of boost_python.
            sh """docker exec ${container_name} ${custom_sh} -c \"
              conan install boost_log/1.65.1@bincrafters/stable \
                --options boost_filesystem:shared=True \
                --build=outdated
            \""""

            // Delete duplicate packages, as they can cause upload problems.
            sh """docker exec ${container_name} ${custom_sh} -c \"
              conan remove --force cmake_installer/3.10.0@conan/stable
            \""""
          }
        }  // stage

        stage("${image_key}: Upload") {
          sh """docker exec ${container_name} ${custom_sh} -c \"
            conan search '*'
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
          sh "conan install conanfile.txt \
              --settings build_type=Release \
              --build=outdated"

          sh "conan install libcurl/7.56.1@bincrafters/stable \
              --settings build_type=Release \
              --options libcurl:shared=True \
              --options libcurl:darwin_ssl=False \
              --build=outdated"

          sh "conan install libcurl/7.56.1@bincrafters/stable \
              --settings build_type=Release \
              --options libcurl:shared=False \
              --options libcurl:darwin_ssl=False \
              --build=outdated"

        }  // stage

        stage("macOS: Upload") {
          sh "conan search '*'"
          sh "conan upload --confirm --all --remote ${conan_remote} '*'"
        }
      }
    }  // node
  }  // return
}  // def


node {
  checkout scm

  if (!env.CHANGE_ID) {
    def builders = [:]
    for (x in images.keySet()) {
      def image_key = x
      builders[image_key] = get_pipeline(image_key)
    }
    builders['macOS'] = get_macos_pipeline()
    parallel builders
  }

  // Delete workspace when build is done.
  cleanWs()
}
