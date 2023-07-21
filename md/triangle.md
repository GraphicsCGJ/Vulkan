# Triangle Code 분석

## 1. initVulkan
### 1.1 Instance 생성
```c++
VulkanExampleBase::createInstance(bool enableValidation);
```
#### 1.1.1. applicationInfo 설정
```VkApplicationInfo``` 에 간단한 name 정보 담아서 구조체 생성

#### 1.1.2. Extension 설정
사용하고자 하는 벌칸 익스텐션 이름들의 배열인 ```instanceExtensions``` 에
VK_KHR_SURFACE + OS별 SURFACE_EXTENSION_NAME을 붙인다.

지원가능한 익스텐션 목록을 가져오고
내가 사용하고자 하는 익스텐션(SURFACE관련 2개) 와 맞지 않으면 에러를 뱉어준다.

디버그 유틸이 포함될 경우, 해당 익스텐션이름도 받아와 추가한다.

#### 1.1.3. Layer 설정

디버그 유틸이 포함될 경우 큐에 제출하기 전에 디버그 레이어를 타야 하므로, 레이어도 하나 추가해준다.

#### 1.1.4. Instance 생성
모아놓은 정보 applicationInfo, ExtensionInfos, LayersInfos 를 가지고 인스턴스를 만든다.
+) 생성 직후, 디버그 익스텐션 사용시, 이를 위한 디버그 유틸 매신저 CreationInfo(콜백) 을 달아준다.


### 1.2 Physical Device 설정
#### 1.2.1. 사용할 GPU 선정
별 일 없으면 0번 GPU를 사용한다. (내가 argv로 지정하지 않을 경우)

#### 1.2.2. physicalDevice 가져오기
vkEnumeratePhysicalDevices 를 통해 GPU목록을 가져오고, 사용하고자 하는 GPU의
```vkPhysicalDevice``` 를 저장해놓는다.

디바이스 Properties (속성)
```c++
typedef struct vkGetPhysicalDeviceProperties {
    uint32_t                            apiVersion;
    uint32_t                            driverVersion;
    uint32_t                            vendorID;
    uint32_t                            deviceID;
    VkPhysicalDeviceType                deviceType;
    char                                deviceName[VK_MAX_PHYSICAL_DEVICE_NAME_SIZE];
    uint8_t                             pipelineCacheUUID[VK_UUID_SIZE];
    VkPhysicalDeviceLimits              limits;
    VkPhysicalDeviceSparseProperties    sparseProperties;
} vkGetPhysicalDeviceProperties;
```

디바이스 Features. 내가 당장 이해하는 플래그는 아래 두개 뿐
```c++
typedef struct VkPhysicalDeviceFeatures {
    ...
    VkBool32    geometryShader;
    VkBool32    tessellationShader;
    ...
} VkPhysicalDeviceFeatures;

```

메모리 관련 정보.
```c++
typedef struct VkPhysicalDeviceMemoryProperties {
    uint32_t        memoryTypeCount;
    VkMemoryType    memoryTypes[VK_MAX_MEMORY_TYPES];
    uint32_t        memoryHeapCount;
    VkMemoryHeap    memoryHeaps[VK_MAX_MEMORY_HEAPS];
} VkPhysicalDeviceMemoryProperties;

typedef struct VkMemoryHeap {
    VkDeviceSize         size;
    VkMemoryHeapFlags    flags;
} VkMemoryHeap;

typedef uint64_t VkDeviceSize;
```
로 되어있으며, 디바이스 내의 메모리 정보를 알 수 있다.

#### 1.2.3. VulkanDevice 초기화
예제 코드 내에서 만들어 놓은 Device 추상화의 최종 버전.
```VulkanDevice``` 객체로 PhisicalDevice 및 Logical Device를 추상화하여 이 객체에 Device관련 데이터를 저장한다.

```c++
struct VulkanDevice
{
	VkPhysicalDevice physicalDevice; // Physical
	VkDevice logicalDevice; // Logical

    // Physical
	VkPhysicalDeviceProperties properties;
	VkPhysicalDeviceFeatures features;
	VkPhysicalDeviceFeatures enabledFeatures;
	VkPhysicalDeviceMemoryProperties memoryProperties;
	std::vector<VkQueueFamilyProperties> queueFamilyProperties;
	std::vector<std::string> supportedExtensions;

    // Logical
	VkCommandPool commandPool = VK_NULL_HANDLE;

	struct
	{
		uint32_t graphics;
		uint32_t compute;
		uint32_t transfer;
	} queueFamilyIndices;

    // Methods.. 디바이스 정보들을 가져오거나, pool 생성 및 커맨드 버퍼의 커맨드들 제출 등의 작업 진행
    VkResult createLogicalDevice(
        VkPhysicalDeviceFeatures enabledFeatures, // 본 예에선 empty
        std::vector<const char *> enabledExtensions, // 본 예에선 empty
        void *pNextChain,
        bool useSwapChain = true,
        VkQueueFlags requestedQueueTypes = VK_QUEUE_GRAPHICS_BIT | VK_QUEUE_COMPUTE_BIT
    );
};
```

### 1.3 Logical Device 설정
#### 1.3.1. 큐패밀리 설정
이제 사용할 큐패밀리들을 설정해야 한다.
위 ```reateLogicalDevice``` 를 보면 팔수 파라미터로
* device의 Feature정보
* 사용할 수 있는 extensions
* pNextChain 정보

를 받고있음을 알 수 있다.
위 함수가 호출되면 각 큐 타입에 해당하는 큐패밀리 idx를 받아오도록 되어있고,
 이 정보들이 각각 queueCreateInfos 벡터에 저장되도록 되어있다.
 비트 설정을 안할 경우 **graphics** 와 동일한 큐패밀리 인덱스를 사용한다.


#### 1.3.2. Extensions 설정
enabledExtensions 라는 벡터 (이 예제에선 빈벡터) 에 추가로 swapchain extension을 추가한다.

#### 1.3.3. pNextChain
이 예제에선 사용하지 않는다.
사용하고자 하는 feature도 따로 정의가 되지 않는다.
device에서 우리가 긁어온 fearture말고, 추가적으로 익스텐션에 의해 정의된 feature들을 추가하고 싶다면 아래와 같이
 구조체 타입을 달아준다.

```c++
if (pNextChain) {
  physicalDeviceFeatures2.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2;
  physicalDeviceFeatures2.features = enabledFeatures;
  physicalDeviceFeatures2.pNext = pNextChain;
  deviceCreateInfo.pEnabledFeatures = nullptr;
  deviceCreateInfo.pNext = &physicalDeviceFeatures2;
}
```

#### 1.3.4. deviceCreateInfo 생성 및 deviec생성
이제 실제 논리 device를 만든다.
* queueCreateInfos
* enabledFeatures
* deviceExtensions // swapchain 추가된 enabledExtensions

#### 1.3.5. graphics queue를 위한 commandPool도 만들어준다.
```c++
commandPool = createCommandPool(queueFamilyIndices.graphics);
```

### 1.4. 자잘한 설정들
#### 1.4.1. swapchain에 만든 객체 저장

swapChain.connect(instance, physicalDevice, device);
swapChain은 ```VulkanSwapChain``` 의 유저타입
이후 initSurface 시에 실제 동작이 시작된다.

#### 1.4.2. 세마포어 생성
```c++
VK_CHECK_RESULT(vkCreateSemaphore(device,
                                  &semaphoreCreateInfo,
                                  nullptr,
                                  &semaphores.presentComplete));

VK_CHECK_RESULT(vkCreateSemaphore(device,
                                  &semaphoreCreateInfo,
                                  nullptr,
                                  &semaphores.renderComplete));
```
로 세마포어 2개를 만든다.

```semaphores.presentComplete``` 와 ```semaphores.renderComplete``` 는 VkSemaphore

#### 1.4.3. submit Info 초기화
큐 제출에 쓰이는 VkSubmitInfo를 여기서 초기화 해준다.
present와 render에 대한 세마포어를 디바이스에서 대기해야한다는 걸 명시한다.


## 2. setWindow
윈도우 생성은 스킵한다. (OS별로 너무 다름)

## 3. prepare
렌더링에 쓰이는 자원들이 할당되는 곳이다.
```VulkanExampleBase::prepare()``` 와 application각각이 설정해놓은 ```prepare()``` 함수들을 호출한다.

```VulkanExampleBase::prepare()``` 이후에 동기화가 ```prepare()``` 가 수행된다.

### 3.1 VulkanExampleBase::prepare()
#### 3.1.1. swapchain 초기화
컬러 포맷 / 컬러스페이스 / device와 OS별 surface 등을 고려하여 swapchain을 생성한다.
세부 로직은 ```initSurface``` 함수를 보면 되는데, 앞에 Device부분 초기화 하는 (큐패밀리 관련) 부분이 많이 겹친다.

#### 3.1.2. Command pool 생성
커맨드 풀 생성. (위에도 만들긴 했어서 중복이다. 최종적으로 사용하는 Pool은 1개)

#### 3.1.3. setupSwapchain



알아야 하는 것들 목록
렌더패스가 정확하게 뭘 하는건지 다시 파악
컬러 어태치먼트가 정확하게 뭘 하는건지 다시 파악
디스크립터셋과 각 버퍼간의 관계 확인


