도저히 할당량 요청이 안되서 다음과 같이 지원 요청 보냈습니다

![image](https://github.com/user-attachments/assets/0957055f-6ff6-479a-b330-b67998ba7846)


그래도 안되면 다른 오픈소스 관심있는거 있는데 그거라도 빠르게 PR 올려보겠습니다

https://github.com/modelcontextprotocol/inspector


19:54 요청 성공했습니다 이제 GPU 서버 접근 가능  
![image](https://github.com/user-attachments/assets/1da402ef-5b63-4e3d-a524-7522fa49f639)



gpu cluster node 생성 완료했고, gpu 인식된거 확인했습니다
### 코드
```hcl
resource "azurerm_kubernetes_cluster_node_pool" "gpu_node_pool" {
  name = "gpupool"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.k8s.id
  vm_size = "standard_nc4as_t4_v3"
  node_count = 1

  node_taints = ["sku=gpu:NoSchedule"]

  node_labels = {
    "sku"       = "gpu"
  }
}
```

### 로그
```
☁  aks-infrastructure [feat/terraform-tutorial-azure] ⚡  k logs -f job/samples-tf-mnist-demo         
2025-06-22 12:34:37.717954: I tensorflow/core/platform/cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: SSE4.1 SSE4.2 AVX AVX2 FMA
2025-06-22 12:34:38.047110: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 0 with properties: 
name: Tesla T4 major: 7 minor: 5 memoryClockRate(GHz): 1.59
pciBusID: 0001:00:00.0
totalMemory: 15.56GiB freeMemory: 15.46GiB
2025-06-22 12:34:38.047144: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:0) -> (device: 0, name: Tesla T4, pci bus id: 0001:00:00.0, compute capability: 7.5)
2025-06-22 12:37:50.590121: I tensorflow/stream_executor/dso_loader.cc:139] successfully opened CUDA library libcupti.so.8.0 locally
Successfully downloaded train-images-idx3-ubyte.gz 9912422 bytes.
Extracting /tmp/tensorflow/input_data/train-images-idx3-ubyte.gz
Successfully downloaded train-labels-idx1-ubyte.gz 28881 bytes.
Extracting /tmp/tensorflow/input_data/train-labels-idx1-ubyte.gz
Successfully downloaded t10k-images-idx3-ubyte.gz 1648877 bytes.
Extracting /tmp/tensorflow/input_data/t10k-images-idx3-ubyte.gz
Successfully downloaded t10k-labels-idx1-ubyte.gz 4542 bytes.
Extracting /tmp/tensorflow/input_data/t10k-labels-idx1-ubyte.gz
```
