@Library('ecdc-pipeline')
import ecdcpipeline.ContainerBuildNode
import ecdcpipeline.PipelineBuilder

container_build_nodes = [
  'centos7': ContainerBuildNode.getDefaultContainerBuildNode('centos7-gcc8'),
  'debian10': ContainerBuildNode.getDefaultContainerBuildNode('debian10'),
  'ubuntu1804': ContainerBuildNode.getDefaultContainerBuildNode('ubuntu1804-gcc8'),
  'ubuntu2004': ContainerBuildNode.getDefaultContainerBuildNode('ubuntu2004')
]

pipeline_builder = new PipelineBuilder(this, container_build_nodes)

builders = pipeline_builder.createBuilders { container ->
  pipeline_builder.stage("${container.key}: checkout") {
    dir(pipeline_builder.project) {
      scm_vars = checkout scm
    }
    // Copy source code to container
    container.copyTo(pipeline_builder.project, pipeline_builder.project)
   // stage

  pipeline_builder.stage("${container.key}: Conan setup") {
    container.sh """
      conan remote add \
        --insert 0 \
        ess-dmsc-local ${local_conan_server}
    """
  }  // stage

  pipeline_builder.stage("${container.key}: build") {
    container.sh """
      conan install boost_filesystem/1.69.0@bincrafters/stable --build=outdated
      conan install boost_system/1.69.0@bincrafters/stable --build=outdated
    """
  }  // stage
}

node('docker') {
  cleanWs()

  stage('Checkout') {
    try {
      scm_vars = checkout scm
    } catch (e) {
      failure_function(e, 'Checkout failed')
    }
  }

  try {
    parallel builders
  } catch (e) {
    pipeline_builder.handleFailureMessages()
    throw e
  }
}
