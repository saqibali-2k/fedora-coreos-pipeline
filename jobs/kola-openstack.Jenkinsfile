def pipeutils, streams, official
node {
    checkout scm
    pipeutils = load("utils.groovy")
    streams = load("streams.groovy")
    official = pipeutils.isOfficial()
}

properties([
    pipelineTriggers([]),
    parameters([
      choice(name: 'STREAM',
             // list devel first so that it's the default choice
             choices: (streams.development + streams.production + streams.mechanical),
             description: 'Fedora CoreOS stream to test'),
      string(name: 'VERSION',
             description: 'Fedora CoreOS Build ID to test',
             defaultValue: '',
             trim: true),
      string(name: 'ARCH',
             description: 'Target architecture',
             defaultValue: 'x86_64',
             trim: true),
      string(name: 'S3_STREAM_DIR',
             description: 'Override the Fedora CoreOS S3 stream directory',
             defaultValue: '',
             trim: true),
      string(name: 'KOLA_TESTS',
             description: 'Override tests to run',
             defaultValue: "",
             trim: true),
      string(name: 'COREOS_ASSEMBLER_IMAGE',
             description: 'Override the coreos-assembler image to use',
             defaultValue: "coreos-assembler:main",
             trim: true),
      string(name: 'FCOS_CONFIG_COMMIT',
             description: 'The exact config repo git commit to run tests against',
             defaultValue: '',
             trim: true),
    ]),
    buildDiscarder(logRotator(
        numToKeepStr: '100',
        artifactNumToKeepStr: '100'
    )),
    durabilityHint('PERFORMANCE_OPTIMIZED')
])

// Locking so we don't run multiple times on the same region. We are speeding up our testing by dividing
// the load to different regions depending on the architecture. This allows us not to exceed our quota in 
// a single region while still being able to execute two test runs in parallel.
lock(resource: "kola-openstack-${params.ARCH}") {

    currentBuild.description = "[${params.STREAM}][${params.ARCH}] - ${params.VERSION}"

    // Use ca-ymq-1 for everything right now since we're having trouble with
    // image uploads to ams1.
    def region = "ca-ymq-1"

    def s3_stream_dir = params.S3_STREAM_DIR
    if (s3_stream_dir == "") {
        s3_stream_dir = "fcos-builds/prod/streams/${params.STREAM}"
    }

    try { timeout(time: 90, unit: 'MINUTES') {
        cosaPod(image: params.COREOS_ASSEMBLER_IMAGE,
                memory: "256Mi", kvm: false,
                secrets: ["aws-fcos-builds-bot-config", "openstack-kola-tests-config"]) {

            def openstack_image_name, openstack_image_filepath
            stage('Fetch Metadata/Image') {
                def commitopt = ''
                if (params.FCOS_CONFIG_COMMIT != '') {
                    commitopt = "--commit=${params.FCOS_CONFIG_COMMIT}"
                }
                // Grab the metadata. Also grab the image so we can upload it.
                shwrap("""
                export AWS_CONFIG_FILE=\${AWS_FCOS_BUILDS_BOT_CONFIG}/config
                cosa init --branch ${params.STREAM} ${commitopt} https://github.com/coreos/fedora-coreos-config
                cosa buildfetch --build=${params.VERSION} --arch=${params.ARCH} \
                    --url=s3://${s3_stream_dir}/builds --artifact=openstack
                cosa decompress --build=${params.VERSION} --artifact=openstack
                """)
                openstack_image_filepath = shwrapCapture("""
                cosa meta --build=${params.VERSION} --arch=${params.ARCH} --image-path openstack
                """)

                // Use a consistent image name for this stream in case it gets left behind
                openstack_image_name = "kola-fedora-coreos-${params.STREAM}-${params.ARCH}"

            }

            stage('Upload/Create Image') {
                // Create the image in OpenStack
                shwrap("""
                # First delete it if it currently exists, then create it.
                ore openstack --config-file=\${OPENSTACK_KOLA_TESTS_CONFIG}/config \
                    --region=${region} \
                    delete-image --id=${openstack_image_name} || true
                ore openstack --config-file=\${OPENSTACK_KOLA_TESTS_CONFIG}/config \
                    --region=${region} \
                    create-image --file=${openstack_image_filepath} \
                    --name=${openstack_image_name} --arch=${params.ARCH}
                """)
            }
            
            // In VexxHost we'll use the network called "private" for the
            // instance NIC, attach a floating IP from the "public" network and
            // use the v3-starter-4 instance (ram=8GiB, CPUs=4).
            // The clouds.yaml config should be located at ${OPENSTACK_KOLA_TESTS_CONFIG}/config.
            //
            // Since we don't have permanent images uploaded to VexxHost we'll
            // skip the upgrade test.
            try {
                fcosKola(cosaDir: env.WORKSPACE, parallel: 5,
                        build: params.VERSION, arch: params.ARCH,
                        extraArgs: params.KOLA_TESTS,
                        skipBasicScenarios: true,
                        skipUpgrade: true,
                        platformArgs: """-p=openstack                               \
                            --allow-rerun-success                                   \
                            --openstack-config-file=\${OPENSTACK_KOLA_TESTS_CONFIG}/config \
                            --openstack-flavor=v3-starter-4                          \
                            --openstack-network=private                              \
                            --openstack-region=${region}                             \
                            --openstack-floating-ip-network=public                   \
                            --openstack-image=${openstack_image_name}""")
            } finally {
                stage('Delete Image') {
                    // Delete the image in OpenStack
                    shwrap("""
                    ore openstack --config-file=\${OPENSTACK_KOLA_TESTS_CONFIG}/config \
                        --region=${region} \
                        delete-image --id=${openstack_image_name}
                    """)
                }
            }

            currentBuild.result = 'SUCCESS'
        }
    }} catch (e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (official && currentBuild.result != 'SUCCESS') {
            slackSend(color: 'danger', message: ":fcos: :openstack: :trashfire: kola-openstack <${env.BUILD_URL}|#${env.BUILD_NUMBER}> [${params.STREAM}][${params.ARCH}] (${params.VERSION})")
        }
    }
}
