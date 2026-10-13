# Jenkins-Plugin | Copy-Artifact |

    This plugin adds a build step to copy artifacts from another Jenkins project.
    allowing you to easily share files between different builds.

# Prerequisites - Source Job [JOB2]

> Before you can copy artifacts, the **source job** must explicitly archive the files and grant permission to the destination job.

    pipeline {
        agent any
        options {
            // Grant explicit permission to destination jobs
            copyArtifactPermission('JOB1')
        }
        stages {
            stage('Archive Output') {
                steps {
                    // Generate your file here, then archive it
                    archiveArtifacts artifacts: 'result.txt', fingerprint: true
                }
            }
        }
    }

# Usage Example - Jenkins Pipeline [JOB1]

    pipeline {
        agent any
        stages {
            stage('CopyArtifacts') {
                steps {
                    copyArtifacts projectName: 'JOB2',
                    filter: 'result.txt'
                    --- Additional Parameters ---
                }
            }
        }
    }

# Parameters

> You can customize the `copyArtifacts` step by adding the following optional parameters:

    | Parameter              | Type      | Description                                                                                   | Default               |

    | projectName            | String    | **(Required)** The name of the source Jenkins project.                                        | None                  |
    | selector               | Object    | Determines *which* build to copy from (see Build Selectors below).                            | Latest stable build   |
    | filter                 | String    | Ant-style glob pattern to filter files (e.g., `**/*.jar`).                                    | None                  |
    | excludes               | String    | Files to explicitly exclude from copying (e.g., `**/*.log`).                                  | None                  |
    | target                 | String    | Destination directory inside the current workspace.                                           | Root workspace        |
    | flatten                | Boolean   | If `true`, ignores the source directory structure and dumps all files into the target folder. | false                 |
    | optional               | Boolean   | If `true`, the build won't fail even if no source build or artifacts are found.               | false                 |
    | fingerprintArtifacts   | Boolean   | Automatically fingerprints the copied files to track dependencies.                            | true                  |
    | parameters             | String    | Filter the source build by its specific job parameters.                                       | None                  |
                                         | Uses comma-separated key-value pairs (e.g., `PARAM1=value1,PARAM2=value2`).                   |                       |

# Selector

    Use one of these options inside `selector: ...` to control which build is targeted

    | Selector             | Code Syntax Example                      | Description                                                                  |

    | lastSuccessful       | selector: lastSuccessful()               | Copies from the latest completed build that did not fail.                    |
    | specific             | selector: specific('123')                | Copies from a specific, hardcoded build number.                              |
    | permalink            | selector: permalink('lastStableBuild')   | Copies from a specific permalink configured on the server.                   |
    | lastCompleted        | selector: lastCompleted()                | Copies from the last completed build, ignoring its status (success/failure). |
    | latestSavedBuild     | selector: latestSavedBuild()             | Copies from the latest build marked as "keep forever".                       |
    | buildParameter       | selector: buildParameter('SRC_NUM')      | Dynamically uses a build number provided by a job parameter.                 |
    | upstream             | selector: upstream()                     | Copies from the upstream build that triggered the current job.               |

# Example 1 - Retrieving Build Artifacts Using Last Successful Build

> This step copies the `result.txt` file from the last successful build of `JOB2` into the current workspace and reads its content.

> [!WARNING]
> `lastSuccessful()` does not always return the latest run. If the current run fails, it will pull outdated results from a past successful build.

    pipeline {
        agent any

        stages {
            stage('Get Data Back from JOB2') {
                steps {
                    copyArtifacts projectName: 'JOB2', 
                                filter: 'result.txt', 
                                selector: lastSuccessful()
                    
                    script {
                        def returnedData = readFile('result.txt').trim()
                        echo "SUCCESS! Data returned from JOB2 is: ${returnedData}"
                    }
                }
            }
        }
    }

# Example 2 - Passing Data Between Jenkins Jobs

> [!NOTE]
> Example 2 uses `specific(runNumber)` to capture the exact build ID dynamically. 
> This guarantees that `JOB1` only processes the current run's data, eliminating the risk of pulling outdated history.

> • [JOB1](JOB1.yaml): Generates two random numbers, passes them to JOB2, and waits. 
                                                 If JOB2 succeeds, it copies the specific result.txt artifact, reads it, and prints the result.
>
> • [JOB2](JOB2.yaml): Receives the two numbers, multiplies them, saves the output to result.txt, and archives it as an artifact

