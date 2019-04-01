node {
    checkout scm
    def commonlib = load("pipeline-scripts/commonlib.groovy")
    def releaseDate = new Date()
    def fullDate = releaseDate.format("yyyyMMddHHmmss", TimeZone.getTimeZone('UTC'))

    // Expose properties for a parameterized build
    properties(
        [
            buildDiscarder(
                logRotator(
                    artifactDaysToKeepStr: '',
                    artifactNumToKeepStr: '',
                    daysToKeepStr: '',
                    numToKeepStr: '1000')),
            [
                $class: 'ParametersDefinitionProperty',
                parameterDefinitions: [
                    commonlib.ocpVersionParam('BUILD_VERSION', 'azure'),
                    [
                        name: 'VERSION',
                        description: 'The major and minor number for this build (e.g 4.0, 4.1, etc, etc)',
                        $class: 'hudson.model.StringParameterDefinition',
                        defaultValue: ""
                    ],
                    [
                        name: 'RELEASE',
                        description: 'Release string for build',
                        $class: 'hudson.model.StringParameterDefinition',
                        defaultValue: ""
                    ],
                    [
                        name: 'IMAGES',
                        description: 'CSV list of images to build. Empty for all. Enter "NONE" to not build any.',
                        $class: 'hudson.model.StringParameterDefinition',
                        defaultValue: ""
                    ],
                    [
                        name: 'EXCLUDE_IMAGES',
                        description: 'CSV list of images to skip building (IMAGES value is ignored)',
                        $class: 'hudson.model.StringParameterDefinition',
                        defaultValue: ""
                    ],
                    [
                        name: 'IMAGE_MODE',
                        description: 'How to update image dist-gits: with a source rebase, just dockerfile updates, or not at all (no version/release update)',
                        $class: 'hudson.model.ChoiceParameterDefinition',
                        choices: ['rebase', 'update-dockerfile', 'nothing'].join('\n'),
                        defaultValue: 'images:rebase',
                    ],
                    [
                        name: 'SIGNED',
                        description: 'Build images against signed RPMs?',
                        $class: 'hudson.model.BooleanParameterDefinition',
                        defaultValue: false
                    ],
                    commonlib.suppressEmailParam(),
                    [
                        name: 'MAIL_LIST_SUCCESS',
                        description: 'Success Mailing List',
                        $class: 'hudson.model.StringParameterDefinition',
                        defaultValue: [
                            'aos-azure@redhat.com',
                        ].join(',')
                    ],
                    commonlib.mockParam(),
                ]
            ],
        ]
    )

    def buildlib = load("pipeline-scripts/buildlib.groovy")
    buildlib.initialize(false)
    GITHUB_BASE = "git@github.com:openshift" // buildlib uses this global var

    majorVersion = params.BUILD_VERSION.split('\\.')[0]
    minorVersion = params.BUILD_VERSION.split('\\.')[1]
    master_ver = commonlib.ocpDefaultVersion
    version = commonlib.standardVersion(params.VERSION)
    release = params.RELEASE.trim()
    repo_type = params.SIGNED ? "signed" : "unsigned"
    images = commonlib.cleanCommaList(params.IMAGES)
    exclude_images = commonlib.cleanCommaList(params.EXCLUDE_IMAGES)

      // doozer_working must be in WORKSPACE in order to have artifacts archived
    doozer_working = "${WORKSPACE}/doozer_working"
    //Clear out previous workspace
    sh "rm -rf ${doozer_working}"
    sh "mkdir -p ${doozer_working}"

    currentBuild.displayName = "#${currentBuild.number} - ${version}-${release}"

    try {
        sshagent(["openshift-bot"]) {
            // To work on real repos, buildlib operations must run with the permissions of openshift-bot

            // Some images require OSE as a source.
            // Instead of trying to figure out which do, always clone

            currentBuild.description = ""

            echo "ping 1"
            // determine which images, if any, should be built, and how to tell doozer that
            include_exclude = ""
            any_images_to_build = true
            if (exclude_images) {
                include_exclude = "-x ${exclude_images}"
                currentBuild.displayName += " [images]"
            } else if (images.toUpperCase() == "NONE") {
                any_images_to_build = false
            } else if (images) {
                include_exclude = "-i ${images}"
                currentBuild.displayName += images.contains(",") ? " [images]" : " [${images} image]"
            }

            echo "ping 2"
            stage("update dist-git") {
                if (!any_images_to_build) { return }
                currentBuild.description += "building image(s): ${include_exclude ?: 'all'}"
                echo "Build Description: ${currentBuild.description}"
                if (params.IMAGE_MODE == "nothing") { return }

                command = "--working-dir ${doozer_working} --group '${BUILD_VERSION}' "
                if (!params.SKIP_OSE) { command += "--source ose ${OSE_DIR} " }
                command += "--latest-parent-version ${include_exclude} "
                command += "images:${params.IMAGE_MODE} --version ${version} --release ${release} "
                command += "--repo-type ${repo_type} "
                command += "--message 'Updating Dockerfile version and release ${version}-${release}' --push "
                buildlib.doozer command
            }

            stage("build images") {
                return
                if (!any_images_to_build) { return }
                command = "--working-dir ${doozer_working} --group 'openshift-${params.BUILD_VERSION}' "
                command += "${include_exclude} images:build --push-to-defaults --repo-type ${repo_type} "
                try {
                    buildlib.doozer command
                } catch (err) {
                    def record_log = buildlib.parse_record_log(doozer_working)
                    def failed_map = buildlib.get_failed_builds(record_log, true)
                    if (failed_map) {
                        def r = buildlib.determine_build_failure_ratio(record_log)
                        if (r.total > 10 && r.ratio > 0.25 || r.total > 1 && r.failed == r.total) {
                            echo "${r.failed} of ${r.total} image builds failed; probably not the owners' fault, will not spam"
                        } else {
                            buildlib.mail_build_failure_owners(failed_map, "aos-team-art@redhat.com", params.MAIL_LIST_FAILURE)
                        }
                    }
                    throw err  // build is considered failed if anything failed
                }
            }

// Comment out temporarily
//            stage('sync images') {
//                if (majorVersion == "4") {
//                    buildlib.sync_images(
//                        majorVersion,
//                        minorVersion,
//                        "aos-team-art@redhat.com",
//                        currentBuild.number
//                    )
//                }
//            }

//            commonlib.email(
//                to: "${params.MAIL_LIST_SUCCESS}",
//                from: "aos-team-art@redhat.com",
//                subject: "Successful custom OCP build: ${currentBuild.displayName}",
//                body: "Jenkins job: ${env.BUILD_URL}\n${currentBuild.description}",
 //           )
        }
    } catch (err) {
        currentBuild.description = "failed with error: ${err}\n${currentBuild.description}"
        commonlib.email(
            to: "${params.MAIL_LIST_FAILURE}",
            from: "aos-team-art@redhat.com",
            subject: "Error building custom OCP: ${currentBuild.displayName}",
            body: """Encountered an error while running OCP pipeline:

${currentBuild.description}

Jenkins job: ${env.BUILD_URL}
Job console: ${env.BUILD_URL}/console
    """)

        currentBuild.result = "FAILURE"
        throw err
    } finally {
        commonlib.safeArchiveArtifacts([
                "doozer_working/*.log",
                "doozer_working/*.yaml",
                "doozer_working/brew-logs/**",
            ])
    }
}
