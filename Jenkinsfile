@Library('ecdc-pipeline')
import ecdcpipeline.ContainerBuildNode
import ecdcpipeline.PipelineBuilder

container_build_nodes = [
  'centos7': ContainerBuildNode.getDefaultContainerBuildNode('centos7-gcc8'),
  'debian10': ContainerBuildNode.getDefaultContainerBuildNode('debian10'),
  'debian11': ContainerBuildNode.getDefaultContainerBuildNode('debian11'),
  'ubuntu2004': ContainerBuildNode.getDefaultContainerBuildNode('ubuntu2004'),
  'ubuntu2204': ContainerBuildNode.getDefaultContainerBuildNode('ubuntu2204')
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

  pipeline_builder.stage("${container.key}: build") {
    container.sh """
      conan install boost_build/1.69.0@bincrafters/stable \
        --build=boost_build \
        --build=outdated
    """

    container.sh """
      conan install boost_filesystem/1.69.0@bincrafters/stable \
        --build=boost_filesystem \
        --build=outdated
    """

   container.sh """
      conan install boost_system/1.69.0@bincrafters/stable \
        --build=boost_system \
        --build=outdated
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
