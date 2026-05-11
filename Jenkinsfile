pipeline {
    agent any

    parameters {
        string(name: 'password', defaultValue: '', description: '远程服务器SSH密码（所有OS使用相同密码）')
        string(name: 'driver_version', defaultValue: '', description: '默认驱动版本号（所有驱动通用）')
        text(
            name: 'DRIVER_VERSION_MAP',
            defaultValue: '',
            description: '''可选：差异化驱动版本映射，格式：驱动名:版本号，每行一个。
未列出的驱动使用默认 driver_version。
示例：
mcepf:1.1.14
mrdma:2.0.1
rnpgbe:1.2.0'''
        )
        text(
            name: 'OS_CONFIG',
            defaultValue: '''AnolisOS-8.6-QU1:10.84.10.188
AnolisOS-8.8:10.84.10.232
AnolisOS-8.9:10.84.10.111
centos6.5_i386:10.84.10.239
centos6.5:10.84.10.209
centos7.2:10.84.10.177
centos7.5:10.84.10.37
centos7.6:10.84.10.172
centos7.7:10.84.10.183
centos7.8:10.84.10.231
centos7.9:10.84.10.196
centos8.0:10.84.10.161
centos8.1:10.84.10.140
centos8.2:10.84.10.107
centos8.3:10.84.10.84
centos8.5:10.84.10.36
centos stream8.9:10.84.10.250
centos9.4:10.84.10.173
ubuntu16.04:10.84.10.164
ubuntu18.04:10.84.10.252
ubuntu20.04:10.84.10.141
ubuntu22.04.1:10.84.10.243
ubuntu22.04.4:10.84.10.120
ubuntu23.04:10.84.10.248
rhel-9.3:10.84.10.251
rhel-9.2:10.84.10.40
rhel_8.9:10.84.10.111
rhel_8.8:10.84.10.112
rhel-8.7:10.84.10.113
rhel-8.2:10.84.10.253
rhel-8.1:10.84.10.203
rhel-7.6:10.84.10.179
rhel-6.6:10.84.10.247
uos-server-20-1070a:10.84.10.187
uos-server-20-1070e:10.84.10.228
Rocky-9.0:10.84.10.225
Rocky-9.3:10.84.10.254
openEuler-20.03-LTS-SP3:10.84.10.233
openEuler-20.03-LTS-SP4:10.84.10.246
openEuler-20.03-LTS-SP1:10.84.10.122
openEuler-20.03-LTS:10.84.10.220
kylinsec-3.5.2:10.84.10.189
''',
            description: 'OS配置列表，格式：OS名称:IP地址，每行一个。支持注释行（以#开头）。'
        )
        booleanParam(name: 'BUILD_ALL', defaultValue: false, description: '一键编译所有驱动（等价于勾选所有驱动）')
        booleanParam(name: 'BUILD_MCEPF', defaultValue: false, description: '编译 mcepf')
        booleanParam(name: 'BUILD_MCEVF', defaultValue: false, description: '编译 mcevf')
        booleanParam(name: 'BUILD_RNP', defaultValue: false, description: '编译 rnp')
        booleanParam(name: 'BUILD_RNPVF', defaultValue: false, description: '编译 rnpvf')
        booleanParam(name: 'BUILD_RNPM', defaultValue: false, description: '编译 rnpm')
        booleanParam(name: 'BUILD_RNPGBE', defaultValue: false, description: '编译 rnpgbe')
        booleanParam(name: 'BUILD_RNPGBEVF', defaultValue: false, description: '编译 rnpgbevf')
        booleanParam(name: 'BUILD_MRDMA', defaultValue: false, description: '编译 mrdma（依赖mcepf，会自动先编译mcepf）')
        booleanParam(name: 'FAIL_FAST', defaultValue: false, description: '是否在一个OS编译失败时立即终止其他OS的编译（默认否，即尽可能完成所有OS的编译）')
    }

    stages {
        stage('多OS多驱动并行编译') {
            steps {
                script {
                    // ====== 解析OS配置 ======
                    def osConfigList = []
                    def lines = params.OS_CONFIG.trim().split('\n')

                    lines.each { line ->
                        def trimmedLine = line.trim()
                        if (trimmedLine && !trimmedLine.startsWith('#')) {
                            def parts = trimmedLine.split(':')
                            if (parts.length >= 2) {
                                def osName = parts[0].trim()
                                def osIp = parts[1].trim()
                                def osPort = parts.length >= 3 ? parts[2].trim() : '22'

                                if (osName && osIp) {
                                    osConfigList.add([
                                        name: osName,
                                        ip: osIp,
                                        port: osPort
                                    ])
                                }
                            }
                        }
                    }

                    if (osConfigList.isEmpty()) {
                        error "OS配置为空或格式错误，请检查OS_CONFIG参数。格式要求：OS名称:IP地址，每行一个"
                    }

                    echo "将要并行编译的OS列表（共 ${osConfigList.size()} 个）："
                    osConfigList.each { os ->
                        echo "  - ${os.name}: ${os.ip}:${os.port}"
                    }

                    // ====== 构建驱动列表 ======
                    def allDrivers = [
                        [name: 'mcepf', buildFlag: params.BUILD_MCEPF],
                        [name: 'mcevf', buildFlag: params.BUILD_MCEVF],
                        [name: 'rnp', buildFlag: params.BUILD_RNP],
                        [name: 'rnpvf', buildFlag: params.BUILD_RNPVF],
                        [name: 'rnpm', buildFlag: params.BUILD_RNPM],
                        [name: 'rnpgbe', buildFlag: params.BUILD_RNPGBE],
                        [name: 'rnpgbevf', buildFlag: params.BUILD_RNPGBEVF],
                        [name: 'mrdma', buildFlag: params.BUILD_MRDMA]
                    ]

                    def selectedDrivers = []
                    allDrivers.each { driver ->
                        if (params.BUILD_ALL || driver.buildFlag) {
                            selectedDrivers.add(driver.name)
                        }
                    }

                    if (selectedDrivers.isEmpty()) {
                        error "未选择任何驱动进行编译，请勾选至少一个驱动或启用 BUILD_ALL"
                    }

                    // 去重并保持顺序
                    selectedDrivers = selectedDrivers.unique()

                    echo "将要编译的驱动列表（共 ${selectedDrivers.size()} 个）："
                    selectedDrivers.each { drv ->
                        echo "  - ${drv}"
                    }

                    // ====== 解析驱动版本映射 ======
                    def driverVersionMap = [:]
                    if (params.DRIVER_VERSION_MAP?.trim()) {
                        def mapLines = params.DRIVER_VERSION_MAP.trim().split('\n')
                        mapLines.each { mapLine ->
                            def trimmedMapLine = mapLine.trim()
                            if (trimmedMapLine && !trimmedMapLine.startsWith('#')) {
                                def mapParts = trimmedMapLine.split(':')
                                if (mapParts.length >= 2) {
                                    def mapDriverName = mapParts[0].trim()
                                    def mapDriverVersion = mapParts[1].trim()
                                    if (mapDriverName && mapDriverVersion) {
                                        driverVersionMap[mapDriverName] = mapDriverVersion
                                    }
                                }
                            }
                        }
                    }

                    def getDriverVersion = { driverName ->
                        return driverVersionMap.get(driverName, params.driver_version)
                    }

                    // ====== 构建并行任务 ======
                    def parallelTasks = [:]

                    osConfigList.each { osInfo ->
                        def osName = osInfo.name
                        def osIp = osInfo.ip
                        def osPort = osInfo.port

                        parallelTasks[osName] = {
                            stage("编译-${osName}") {
                                script {
                                    echo "============================"
                                    echo "开始在 ${osName}(${osIp}:${osPort}) 上编译..."
                                    echo "============================"

                                    try {
                                        // 每个 VM 上构建驱动执行列表（考虑依赖）
                                        def vmDrivers = []
                                        selectedDrivers.each { drv ->
                                            vmDrivers.add(drv)
                                        }

                                        // mrdma 依赖 mcepf：如果选了 mrdma 但没选 mcepf，自动插入
                                        def hasMrdma = vmDrivers.contains('mrdma')
                                        def hasMcepf = vmDrivers.contains('mcepf')
                                        def mcepfInjected = false

                                        if (hasMrdma && !hasMcepf) {
                                            vmDrivers = ['mcepf'] + vmDrivers
                                            mcepfInjected = true
                                            echo "[${osName}] 自动注入 mcepf 依赖（mrdma 需要 mcepf）"
                                        }

                                        def mcepfFailed = false

                                        vmDrivers.each { driverName ->
                                            // 如果 mcepf 失败且当前是 mrdma，则跳过
                                            if (mcepfFailed && driverName == 'mrdma') {
                                                echo "[${osName}] ⚠️ 跳过 ${driverName}，因为前置依赖 mcepf 编译失败"
                                                currentBuild.result = 'UNSTABLE'
                                                return
                                            }

                                            def drvVersion = getDriverVersion(driverName)
                                            echo "[${osName}] >>> 开始编译驱动: ${driverName} (版本: ${drvVersion})"

                                            try {
                                                // 1. 清理旧文件
                                                sh """
                                                    sshpass -p '${params.password}' ssh -o StrictHostKeyChecking=no -p ${osPort} root@${osIp} "rm -rf /home/${driverName}*"
                                                """
                                                sleep 5

                                                // 2. 传输驱动包
                                                sh """
                                                    sshpass -p '${params.password}' scp -o StrictHostKeyChecking=no -P ${osPort} /home/kernel_driver/${driverName}-${drvVersion}.tar.gz root@${osIp}:/home
                                                """
                                                sleep 5

                                                // 3. 解压
                                                sh """
                                                    sshpass -p '${params.password}' ssh -o StrictHostKeyChecking=no -p ${osPort} root@${osIp} "cd /home/; tar xvf ${driverName}-${drvVersion}.tar.gz"
                                                """
                                                sleep 2

                                                // 4. 编译（mrdma 特殊处理）
                                                if (driverName == 'mrdma') {
                                                    sh """
                                                        sshpass -p '${params.password}' ssh -o StrictHostKeyChecking=no -p ${osPort} root@${osIp} "cd /home/${driverName}-${drvVersion}; ./mrdma_install.sh"
                                                    """
                                                } else {
                                                    sh """
                                                        sshpass -p '${params.password}' ssh -o StrictHostKeyChecking=no -p ${osPort} root@${osIp} "cd /home/${driverName}-${drvVersion}/src; make"
                                                    """
                                                }

                                                echo "[${osName}] ✅ 驱动 ${driverName} 编译完成！"

                                            } catch (Exception e) {
                                                echo "[${osName}] ❌ 驱动 ${driverName} 编译失败: ${e.getMessage()}"

                                                // 标记 mcepf 失败，影响后续 mrdma
                                                if (driverName == 'mcepf') {
                                                    mcepfFailed = true
                                                }

                                                currentBuild.result = 'UNSTABLE'
                                                throw e
                                            }
                                        }

                                        echo "============================"
                                        echo "✅ ${osName} 所有驱动编译完成！"
                                        echo "============================"

                                    } catch (Exception e) {
                                        echo "============================"
                                        echo "❌ ${osName} 编译流程失败: ${e.getMessage()}"
                                        echo "============================"
                                        currentBuild.result = 'UNSTABLE'
                                        throw e
                                    }
                                }
                            }
                        }
                    }

                    // 执行并行任务
                    parallelTasks['failFast'] = params.FAIL_FAST
                    parallel parallelTasks
                }
            }
        }
    }

    post {
        always {
            script {
                echo "构建结束 - 状态: ${currentBuild.result}"
                echo "========================================="
            }
        }
        success {
            echo "✅ 所有OS编译成功！"
        }
        unstable {
            echo "⚠️ 部分OS编译失败，请检查日志！"
        }
        failure {
            echo "❌ 编译流程出现严重错误！"
        }
    }
}
