def ciProject = 'labs-ci-cd'
def testProject = 'labs-test'
def devProject = 'labs-dev'
openshift.withCluster() {
    openshift.withProject() {
        ciProject = openshift.project()
        testProject = ciProject.replaceFirst(/^labs-ci-cd/, 'labs-test')
        devProject = ciProject.replaceFirst(/^labs-ci-cd/, 'labs-dev')
    }
}

def buildConfig = { project, namespace, buildSecret, fromImageStream ->
    if (!fromImageStream) {
        fromImageStream = 'registry.access.redhat.com/rhscl/nginx-112-rhel7'
    }
    def template = """
---
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  labels:
    build: ${project}
  name: ${project}
  namespace: ${namespace}
spec:
  failedBuildsHistoryLimit: 5
  nodeSelector: null
  output:
    to:
      kind: ImageStreamTag
      name: '${project}:latest'
  postCommit: {}
  resources: {}
  runPolicy: Serial
  source:
    binary: {}
    type: Binary
  strategy:
    sourceStrategy:
      from:
        kind: DockerImage
        name: '${fromImageStream}'
    type: Source
  successfulBuildsHistoryLimit: 5
"""
    openshift.withCluster() {
        openshift.apply(template, "--namespace=${namespace}")
    }
}

def deploymentConfig = {project, ciNamespace, targetNamespace ->
    def template = """
---
apiVersion: v1
kind: List
items:
- apiVersion: v1
  data:
    env: |-
      {
          "hostname": "localhost",
          "service_port": 8080
      }
  kind: ConfigMap
  metadata:
    name: insult-config
- apiVersion: v1
  kind: ImageStream
  metadata:
    labels:
      build: '${project}'
    name: '${project}'
    namespace: '${targetNamespace}'
  spec: {}
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: '${project}'
    name: '${project}'
  spec:
    replicas: 1
    selector:
      name: '${project}'
    strategy:
      activeDeadlineSeconds: 21600
      resources: {}
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        creationTimestamp: null
        labels:
          name: '${project}'
      spec:
        containers:
          - image: '${project}'
            imagePullPolicy: Always
            name: '${project}'
            env:
            - name: KUBERNETES_NAMESPACE
              value: ${targetNamespace}
            - name: JAVA_OPTIONS
              value: |-
                -Dvertx.jgroups.config=default-configs/default-jgroups-kubernetes.xml -Djava.net.preferIPv4Stack=true
                -Dorg.slf4j.simpleLogger.log.org.jgroups=WARN -Dorg.slf4j.simpleLogger.log.org.infinispan=WARN
            - name: JAVA_ARGS
              value: '-cluster -cluster-port 5800'
            ports:
              - containerPort: 8080
                protocol: TCP
            livenessProbe:
                failureThreshold: 3
                httpGet:
                  path: /api/v1/health
                  port: 8080
                  scheme: HTTP
                initialDelaySeconds: 5
                periodSeconds: 10
                successThreshold: 1
                timeoutSeconds: 1
            readinessProbe:
              failureThreshold: 3
              httpGet:
                path: /api/v1/health
                port: 8080
                scheme: HTTP
              initialDelaySeconds: 10
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 1
            resources: {}
            terminationMessagePath: /dev/termination-log
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        securityContext: {}
        terminationGracePeriodSeconds: 30
    test: false
    triggers:
      - type: ConfigChange
      - imageChangeParams:
          automatic: true
          containerNames:
            - '${project}'
          from:
            kind: ImageStreamTag
            name: '${project}:latest'
        type: ImageChange
- apiVersion: v1
  kind: Service
  metadata:
    labels:
      name: '${project}'
    name: '${project}'
  spec:
    ports:
      - name: http
        port: 3000
        protocol: TCP
        targetPort: 8080
    selector:
      name: '${project}'
    sessionAffinity: None
    type: ClusterIP
- apiVersion: v1
  kind: Route
  metadata:
    labels:
      name: '${project}'
    name: '${project}'
  spec:
    port:
      targetPort: http
    to:
      kind: Service
      name: '${project}'
      weight: 100
    wildcardPolicy: None
"""
    openshift.withCluster() {
        openshift.apply(template, "--namespace=${targetNamespace}")
    }
}

pipeline {
    agent {
        label 'jenkins-slave-npm'
    }
    environment {
        PROJECT_NAME = 'cr18-22'
        KUBERNETES_NAMESPACE = "${ciProject}"
    }
    stages {
        stage('Prep Sonar Scanner') {
          steps {
            sh "curl -L -o sonar-scanner.zip 'https://akamai.bintray.com/07/07a50ec270a36cb83f26fe93233819c53c145248c638f4591880f1bd36e331d6?__gda__=exp=1534781610~hmac=a6bd78d8433afd6173c4bd8917eff3089b1e0ada162090b8102790cf1e821528&response-content-disposition=attachment%3Bfilename%3D%22sonar-scanner-cli-3.2.0.1227-linux.zip%22&response-content-type=application%2Fzip&requestInfo=U2FsdGVkX18ABhchMOQyLZ-TkJi4cKa2gAn34yDqPKmbn8EZO3lnQ_8dJxd7LpBSOelPp16Cw9SFSf-2lDNbJ96ZUln81t8QBbtMQE_jEsqnr_bhKCCBnxeV74v3QHXi2NhkCCMN168ZjDxqLZUNydgRwyKFU9krXGGxILrfMn0u8urKSkmZXp2TSgD2jeoW3Za2t1LHYGFqDcP3j1K-kWrjxh-8B4QBFs7qeGOrpbA&response-X-Checksum-Sha1=0c3074dc06491fdd060c1039ee8d5dae95a2bddc&response-X-Checksum-Sha2=07a50ec270a36cb83f26fe93233819c53c145248c638f4591880f1bd36e331d6'"
            sh "unzip -x sonar-scanner.zip"
            sh 'mv sonar-scanner-*-linux sonar-scanner'
          }
        }
        stage('Quality And Security') {
            parallel {
                stage('Compile & Test') {
                    steps {
                        sh 'npm install'
                        sh 'npm run build'
                        sh 'npm run test:unit'
                    }
                }
                stage('Ensure SonarQube Webhook is configured') {
                    when {
                        expression {
                            withSonarQubeEnv('sonar') {
                                def retVal = sh(returnStatus: true, script: "curl -u \"${SONAR_AUTH_TOKEN}:\" http://sonarqube:9000/api/webhooks/list | grep Jenkins")
                                echo "CURL COMMAND: ${retVal}"
                                return (retVal > 0)
                            }
                        }
                    }
                    steps {
                        withSonarQubeEnv('sonar') {
                            sh "curl -X POST -u \"${SONAR_AUTH_TOKEN}:\" -F \"name=Jenkins\" -F \"url=http://jenkins/sonarqube-webhook/\" http://sonarqube:9000/api/webhooks/create"
                        }
                    }
                }
            }
        }
        stage('Wait for SonarQube Quality Gate') {
            steps {
                script {
                    withSonarQubeEnv('sonar') {
                        sh './sonar-scanner/bin/sonar-scanner '
                    }
                    def qualitygate = waitForQualityGate()
                    if (qualitygate.status != "OK") {
                        error "Pipeline aborted due to quality gate failure: ${qualitygate.status}"
                    }
                }
            }
        }
        stage('OpenShift Deployments') {
            parallel {
                stage('Create Binary BuildConfig') {
                    steps {
                        script {
                            buildConfig(PROJECT_NAME, ciProject, UUID.randomUUID().toString())
                        }
                    }
                }
                stage('Create Test Deployment') {
                    steps {
                        script {
                            deploymentConfig(PROJECT_NAME, ciProject, testProject)
                        }
                    }
                }
                stage('Create Dev Deployment') {
                    steps {
                        script {
                            deploymentConfig(PROJECT_NAME, ciProject, devProject)
                        }
                    }
                }
            }
        }
        stage('Build Image') {
            steps {
                script {
                    openshift.selector('bc', PROJECT_NAME).startBuild("--from-dir=dist", '--wait')
                }
            }
        }
        stage('Promote to TEST') {
            steps {
                script {
                    openshift.tag("${PROJECT_NAME}:latest", "${testProject}/${PROJECT_NAME}:latest")
                }
            }
        }
        stage('Promote to DEMO') {
            input {
                message "Promote service to DEMO environment?"
                ok "PROMOTE"
            }
            steps {
                script {
                    openshift.tag("${PROJECT_NAME}:latest", "${devProject}/${PROJECT_NAME}:latest")
                }
            }
        }
    }
}
