## kubernetes二次开发

kubernetes本身对很多场景已经能够达到直接使用，但是kubernetes也提供了能够扩展的能力，可以基于kubernetes进行二次开发。

* client-go：k8s的官方开发库，可以直接调用k8s的接口，常用于开发一些运维平台或者操作k8s的资源
* virtual-kubelet：虚拟kubelet，可以伪装成kubelet，常用于实现Serverless、edge IOT等
* cloud-provider：云端负载均衡器，
* CRD & Operator & Controller：定义用户自己的资源，然后基于client-go开发自己的Controller控制自己资源的运行
