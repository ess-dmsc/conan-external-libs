project = "conan-external-libs"

conan_remote = "ess-dmsc-local"
conan_user = "ess-dmsc"

images = [
    'centos': [
        'name': 'essdmscdm/centos-build-node:0.8.0',
        'sh': 'sh'
    ],
    'centos-gcc6': [
        'name': 'essdmscdm/centos-gcc6-build-node:0.2.0',
        'sh': '/usr/bin/scl enable rh-python35 devtoolset-6 -- /bin/bash'
    ],
    'fedora': [
        'name': 'essdmscdm/fedora-build-node:0.4.1',
        'sh': 'sh'
    ]
]

base_container_name = "${project}-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

def get_pipeline(image_key) {
    return {
        def container_name = "${base_container_name}-${image_key}"
        try {
            def image = docker.image(images[image_key]['name'])
            def custom_sh = images[image_key]['sh']
            def container = image.run("\
                --name ${container_name} \
                --tty \
                --network=host \
                --env http_proxy=${env.http_proxy} \
                --env https_proxy=${env.https_proxy} \
                --env local_conan_server=${env.local_conan_server} \
            ")

            // Copy sources to container and change owner and group.
            sh "docker cp ${project} ${container_name}:/home/jenkins/${project}"
            sh """docker exec --user root ${container_name} ${custom_sh} -c \"
                chown -R jenkins.jenkins /home/jenkins/${project}
            \""""

            stage("${image_key}: Conan setup") {
                    withCredentials([string(
                        credentialsId: 'local-conan-server-password',
                        variable: 'CONAN_PASSWORD'
                    )])
                {
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
                }
            }

            stage("${image_key}: Package") {
                sh """docker exec ${container_name} ${custom_sh} -c \"
                    conan install zlib/1.2.11@conan/stable \
                        --settings build_type=Release \
                        --options zlib:shared=False \
                        --build=missing
                    conan install zlib/1.2.11@conan/stable \
                        --settings build_type=Release \
                        --options zlib:shared=True \
                        --build=missing
                    conan install gtest/1.8.0@conan/stable \
                        --settings build_type=Release \
                        --options gtest:shared=False \
                        --build=missing
                    conan install gtest/1.8.0@conan/stable \
                        --settings build_type=Release \
                        --options gtest:shared=True \
                        --build=missing
                    conan install cmake_installer/1.0@conan/stable \
                        --options cmake_installer:version="3.9.0" \
                        --build=missing
                \""""

                // Boost only builds on CentOS with GCC 6.3.
                if (image_key == 'centos-gcc6') {
                    sh """docker exec ${container_name} ${custom_sh} -c \"
                        conan install Boost/1.64.0@conan/stable \
                            --settings build_type=Release \
                            --options Boost:shared=False \
                            --build=missing
                        conan install Boost/1.64.0@conan/stable \
                            --settings build_type=Release \
                            --options Boost:shared=True \
                            --build=missing
                    \""""
                }
            }

            stage("${image_key}: Upload") {
                sh """docker exec ${container_name} ${custom_sh} -c \"
                    conan upload --confirm --all --remote ${conan_remote} '*'
                \""""
            }
        } finally {
            sh "docker stop ${container_name}"
            sh "docker rm -f ${container_name}"
        }
    }
}

// Temporarily hard code build node because of disk space limitations.
node('docker && dmbuild03.dm.esss.dk') {
    def builders = [:]

    // Delete workspace when build is done.
    cleanWs()

    dir("${project}") {
        stage('Checkout') {
            scm_vars = checkout scm
        }
    }

    for (x in images.keySet()) {
        def image_key = x
        builders[image_key] = get_pipeline(image_key)
    }
    parallel builders
}
