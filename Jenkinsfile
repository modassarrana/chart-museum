def createNamespace(namespace) {
    echo " Create namespace if needed "
    sh "[ ! -z \"\$(kubectl get ns ${namespace} -o name 2>/dev/null)\" ] || kubectl create ns ${namespace}"
}

def checkSecretExist(namespace,USERNAME,PASSWORD) {
    echo " Check secret exist in namespace ${namespace} "
    sh "[ ! -z \"\$(kubectl get secret chartmuseum-auth -n ${namespace} -o name 2>/dev/null)\" ] || kubectl create secret generic chartmuseum-auth --from-literal=\"basic-auth-user=$USERNAME\" --from-literal=\"basic-auth-pass=$PASSWORD\" --namespace ${namespace}"
}

def getServiceUrl(namespace)
{
    def url
    def serviceIP
    def servicePort
    // Fetch service type to generate url
	def serviceType = sh(script: "kubectl get svc new-chartmuseum --namespace ${namespace} -o jsonpath='{.spec.type}'", returnStdout: true)
    echo "$serviceType"
    // Set URL based on service type
    if ("$serviceType" == 'LoadBalancer') {
        echo 'ServiceType is Loadbalncer'
        serviceIP = sh(script: "kubectl get svc new-chartmuseum --namespace ${namespace}  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'", returnStdout: true )
        servicePort = sh(script: "kubectl get svc new-chartmuseum --namespace ${namespace}  -o jsonpath='{.spec.ports[0].port}'", returnStdout: true)
    } else {
        echo "service Type is NodePort"
		//  podName = sh(script: "kubectl get pod -n chart-museum -o name | grep new-chartmuseum | awk -F\"/\" '{print \$2}'", returnStdout: true).trim()
		serviceIP = sh(script: "kubectl get nodes --namespace ${namespace} -o jsonpath='{.items[0].status.addresses[0].address}'", returnStdout: true )
		servicePort = sh(script: "kubectl get svc new-chartmuseum --namespace ${namespace}  -o jsonpath='{.spec.ports[0].nodePort}'", returnStdout: true)
    }
    url = "http://$serviceIP:$servicePort"
    return url
}

def curlTest(url,USERNAME,PASSWORD) {
    echo " Running curl for chart-museum on $url "
    def out = "http_code"
    def result = sh (
                returnStdout: true,
                script: "curl --output /dev/null --silent --connect-timeout 5 --max-time 5 --retry 5 --retry-delay 5 --retry-max-time 30 -u ${USERNAME}:${PASSWORD} --write-out \"%{${out}}\" ${url}"
        )
    // Check status code is 200 for succefull access
    if ("$result" == "200") {
        echo " Chart museum url is accessible with Result (${out}): $result"
    }
    else {
        echo " Chart museum is not accessible with ERROR (${out}):  $result"
    }
}
podTemplate(
    label: 'mypod', 
    inheritFrom: 'default',
    serviceAccount: 'test',
    containers: [
        containerTemplate(
            name: 'helm', 
            image: 'lachlanevenson/k8s-helm:v2.13.0',
            ttyEnabled: true,
            command: 'cat'
        ),
        containerTemplate(
            name: 'kubectl',
            image: 'lachlanevenson/k8s-kubectl:v1.15.6',
            ttyEnabled: true,
            command: 'cat'
        )
    ],
    volumes:[
        hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
    ]
) {
    node('mypod') {
        def commitId
        def namespace = 'chart-museum'
        def baseurl
        stage ('Extract') {
            checkout scm
            commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }
        stage ('Deploy') {
            container ('kubectl') {
                sh "kubectl get nodes"
		// create namespace if needed
                createNamespace(namespace)
                withCredentials([usernamePassword(credentialsId: 'chart-museum-cred', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    //check secret exist in chart-musuem namespace
                    checkSecretExist(namespace,USERNAME, PASSWORD)
		}
            }
            container ('helm') {
                sh "helm init --client-only --skip-refresh"
		// checkout helm version
                sh "helm version"   
                echo "Deploying chart-museum helm chart ..."
                sh "helm upgrade --install --namespace ${namespace} --wait -f chart-museum/values.yaml new chart-museum"
            }
        }
        stage ('Test pod on k8s') {
            container ('kubectl') {
                // Install curl binary on kubectl container. since image is inherited from alpine
		sh "apk add curl"
		// Check chart-museum pod is in running state
		podStatus = sh(script: "kubectl get pods -n ${namespace} | grep new |  awk -F \" \" '{print \$3}'",returnStdout: true).trim()
		if ("$podStatus" == 'Running') {
			echo " chart museum pod is in running status"
		}
		else {
			echo "chart museum pod is in $podStatus status"
		}
		// Determine chart-museum service url 
		baseurl = getServiceUrl(namespace)
		// test curl command to confirm chart musuem is installed properly
		withCredentials([usernamePassword(credentialsId: 'chart-museum-cred', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    curlTest(baseurl , USERNAME , PASSWORD)
                }
            }
        }
    }
}
