pipeline {
    agent any

    stages {
        stage('Create Files and Same Tar file') {
            steps {
                script {
                    writeFile file: 'model_weights.json', text: '{"weights": "initial_weights"}'
                    writeFile file: 'control_output.json', text: '{"output": "initial_output"}'
                    
                    sh '''
                    sha256sum model_weights.json > checksum.txt
                    sha256sum control_output.json >> checksum.txt
                    '''
                    
                    sh 'tar -cvf same_values.tar model_weights.json control_output.json checksum.txt'
                }
            }
        }
        stage('Modify Files and Create Mismatch Tar') {
            steps {
                script {
                    sh 'echo \'"additional_line": "new_data"\' >> control_output.json'
                    
                    sh 'tar -cvf mismatch_values.tar model_weights.json control_output.json checksum.txt'
                }
            }
        }
        stage('Remove Sample Files') {
            steps {
                script {
                    // Remove all files created
                    sh 'rm *.json checksum.txt'
                }
            }
        }

        stage('Validate Hash') {
            steps {
                script {
                    checkout scm
                    def tarball_name = "mismatch_values.tar"
                    def PWD = sh(script: "echo \$(pwd)", returnStdout: true).trim()
                    
                    // Untar the tarball.
                    sh """
                        tar -xvf ${PWD}/${tarball_name} -C ${PWD}
                    """
                    
                    def checksumFile = "${PWD}/checksum.txt"
                    
                    // Check if checksum.txt exists.
                    if (!fileExists(checksumFile)) {
                        error "Missing checksum file: ${checksumFile}"
                    }
                    
                    // Read checksum file contents.
                    def checksumContent
                    try {
                        checksumContent = readFile(checksumFile)
                    } catch (Exception e) {
                        error "Failed to read checksum file: ${checksumFile}"
                    }
                    
                    // Validate checksum file format and hash values.
                    def calculatedHashes = [:]
                    def missingHashes = []
                    
                    checksumContent.readLines().each { checksumLine ->
                        try {
                            def parts = checksumLine.split()
                            if (parts.size() >= 2) {
                                def expectedHash = parts[0].trim()
                                def filename = parts[1].trim()
                                
                                // Validate hash format.
                                if (!expectedHash.matches(/[a-fA-F0-9]{64}/)) {
                                    error "Corrupted checksum file: ${checksumFile}"
                                }
                                
                                // Calculate hash for each file except "checksum.txt".
                                if (!filename.endsWith("checksum.txt")) {
                                    def fileToHash = "${PWD}/${filename}"
                                    if (!fileExists(fileToHash)) {
                                        error "Missing file: ${filename} expected in checksum file."
                                    }
                                    def calculatedHash = sh(script: "sha256sum ${fileToHash} | awk '{print \$1}'", returnStdout: true).trim()
                                    calculatedHashes[filename] = calculatedHash
                                    
                                    // Comparing the hashes.
                                    if (expectedHash == calculatedHash) {
                                        echo "Hash verification successful for ${filename}!"
                                    } else {
                                        error "Verification failed: Hash mismatch for ${filename}. Expected: ${expectedHash}, Calculated: ${calculatedHash}"
                                    }
                                }
                            } else {
                                if (parts.size() == 0) {
                                    missingHashes.add("Missing hash in line: ${checksumLine}")
                                } else if (parts.size() == 1) {
                                    missingHashes.add("Missing hash for file in line: ${checksumLine}")
                                }
                            }
                        } catch (Exception ex) {
                            echo "Error processing line in checksum file: ${checksumLine}"
                            ex.printStackTrace()
                        }
                    }
                    
                    if (missingHashes) {
                        error "Missing hashes in checksum file: \n${missingHashes.join('\n')}"
                    }
                    
                    // Check for missing files in "checksumfile.txt"
                    def filesInChecksum = checksumContent.readLines().collect { line ->
                        line.split()[1].trim()
                    }.findAll { filename ->
                        !filename.endsWith("checksum.txt")
                    }
                    
                    def filesInWorkspace = sh(script: "ls ${PWD}", returnStdout: true).trim().split()

                    def missingFiles = filesInChecksum.findAll { filename ->
                        !filesInWorkspace.contains(filename)
                    }
                    
                    if (missingFiles) {
                        error "Missing files in workspace: ${missingFiles.join(', ')}"
                    }
                }
            }
        }
    }

    post {
        always{
            // Clean up workspace
            deleteDir()
        }
    }
}
