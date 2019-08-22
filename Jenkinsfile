@Library('ecdc-pipeline')
import ecdcpipeline.ContainerBuildNode
import ecdcpipeline.PipelineBuilder

project = "conan-external-libs"

conan_remote = "ess-dmsc-local"
conan_user = "ess-dmsc"

container_build_nodes = [
  'centos7': ContainerBuildNode.getDefaultContainerBuildNode('centos7'),
  'alpine': ContainerBuildNode.getDefaultContainerBuildNode('alpine'),
  'debian9': ContainerBuildNode.getDefaultContainerBuildNode('debian9'),
  'ubuntu1804': ContainerBuildNode.getDefaultContainerBuildNode('ubuntu1804')
]

pipeline_builder = new PipelineBuilder(this, container_build_nodes)

builders = pipeline_builder.createBuilders { container ->

    pipeline_builder.stage("${container.key}: checkout") {
        dir(pipeline_builder.project) {
            scm_vars = checkout scm
        }
        // Copy source code to container
        container.copyTo(pipeline_builder.project, pipeline_builder.project)
    }  // stage

    pipeline_builder.stage("${container.key}: Conan setup") {
        withCredentials([
            string(
              credentialsId: 'local-conan-server-password',
              variable: 'CONAN_PASSWORD'
            )
        ]) {
            container.sh """
              set +x
              conan remote add \
                --insert 0 \
                ${conan_remote} ${local_conan_server}
              conan user \
                --password '${CONAN_PASSWORD}' \
                --remote ${conan_remote} \
                ${conan_user} \
                > /dev/null
            """
        }  // withCredentials
    }  // stage

    pipeline_builder.stage("${container.key}: package") {
        container.sh """
            conan install ${project}/conanfile.txt \
              --settings build_type=Release \
              --build=outdated
          """

        container.sh """
          conan install libcurl/7.56.1@bincrafters/stable \
            --settings build_type=Release \
            --options libcurl:shared=True \
            --build=outdated
        """

        container.sh """
          conan install libcurl/7.56.1@bincrafters/stable \
            --settings build_type=Release \
            --options libcurl:shared=False \
            --build=outdated
        """

        container.sh """
          conan install OpenSSL/1.0.2n@conan/stable \
            --settings build_type=Release \
            --options OpenSSL:shared=False \
            --build=outdated
        """

        container.sh """
          conan install zlib/1.2.11@conan/stable \
            --settings build_type=Release \
            --options zlib:shared=False \
            --build=outdated
        """

        if (container.key == 'centos7') {
          // There is only one cmake_findboost_modular package.
          container.sh """
            conan install cmake_findboost_modular/1.69.0@bincrafters/stable \
              --build=outdated
          """

          // Header-only package.
          container.sh """
            conan install jsonformoderncpp/3.1.0@vthiery/stable \
              --build=outdated
          """
        } else {
          container.sh """
            conan install boost_log/1.69.0@bincrafters/stable \
              --options shared=True \
              --build=outdated
          """
        }
    }  // stage

    pipeline_builder.stage("${container.key}: Upload") {
        container.sh """
            conan search '*'
            conan upload --confirm --all --remote ${conan_remote} '*'
        """
    }  // stage
}

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

node('docker') {
    cleanWs()

    stage('Checkout') {
        dir("${project}") {
            try {
                scm_vars = checkout scm
            } catch (e) {
                failure_function(e, 'Checkout failed')
            }
        }
    }

    if (!env.CHANGE_ID) {
        builders['macOS'] = get_macos_pipeline()

        try {
            parallel builders
        } catch (e) {
            pipeline_builder.handleFailureMessages()
            throw e
        }
    }
}
