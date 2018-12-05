＃當Kubernetes版本時會發生什麼！

> Translate by Google doc

想像一下，我想將nginx部署到Kubernetes集群。我可能會在我的終端類型是這樣的

```bash
kubectl運行--image = nginx的--replicas =
```

然後按Enter。幾秒鐘後，我會看到三個nginx pod分佈在我的所有工作節點上。它的工作方式就像魔法一樣，那太好了！但真正發生了什麼呢？

Kubernetes的一個令人敬畏的事情是，它通過用戶友好的API處理跨基礎架構的工作負載部署。簡單的抽象隱藏了複雜性。但是為了充分理解它為我們提供的價值，了解它的內部結構也很有用。本指南將引導您完成從客戶端到kubelet的請求的整個生命週期，在必要時鏈接到源代碼以說明正在發生的事情。

這是一份活文件。如果您發現可以改進或重寫的區域，歡迎捐款！

## contents

1. [kubectl]（#kubectl）
   -  [驗證和生成器]（＃validation-and-generators）
   -  [API組和版本協商]（＃api-groups-and-version-negotiation）
   -  [client- side auth]（＃client-auth）
1. [kube-apiserver]（＃kube-apiserver）
   -  [認證]（＃認證）
   -  [授權]（＃授權）
   -  [准入控制]（#cancent-control）
1. [etcd]（＃etcd）]（#controls
1. [初始化程序]（#initializers）
1. [控制循環loops）
   -  [部署控制器]（#pripments-controller）
   -  [replicasets controller]（＃replicasets-controller） ）
   -  [）
   -  [scheduler]（＃scheduler）
informers]（#interformers1。[kubelet]（#kubelet）
   -  [pod sync]（#pod-sync）
   -  [CRI和暫停容器]（＃cri-和 -暫停 - 容器）
   -  [CNI和pod網絡]（#cni-and-pod-networking）
   -  [主機間網絡]（#inter-host-networking）
   -  [容器啟動]（＃container-startup）
1. [總結]（＃wrap-up）

## kubectl

###驗證和生成器

好的，讓我們開始吧。我們剛進入我們的終端。現在怎麼辦？

kubectl將要做的第一件事是執行一些客戶端驗證。這可確保_always_失敗的請求（例如，創建不受支持的資源或使用[格式錯誤的圖像名稱]（https://github.com/kubernetes/kubernetes/blob/9a480667493f6275c22cc9cd0f69fb0c75ef3579/pkg/kubectl/cmd/run.go ＃L251））將快速失敗，不會被發送到kube-apiserver。這通過減少不必要的負載來提高系統性能

驗證後，kubectl開始彙編它將發送給kube-apiserver的HTTP請求。在Kubernetes系統中訪問或更改狀態的所有嘗試都通過API服務器進行，後者又與etcd進行通信。 kubectl客戶端也不例外。為了構造HTTP請求，kubectl使用了一個名為[generators]（https://kubernetes.io/docs/user-guide/kubectl-conventions/#generators）的東西，這是一個負責序列化的抽象。

可能不明顯的是，您實際上可以使用`kubectl run`指定多種資源類型，而不僅僅是部署。為了做到這一點，如果生成器名稱為'n'，kubectl將[推斷]（https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go#L231-L257）資源類型t使用`--generator`標誌明確指定。 

例如，具有`--restart-policy = Always`的資源被視為Deployments，而具有`--restart-policy = Never`的資源被視為Pods。 kubectl還將確定是否需要觸發其他操作，例如記錄命令（用於轉出或審計），或者此命令是否只是乾運行（由`--dry-run`標誌指示）。 

在意識到我們想要創建部署之後，它將使用`DeploymentV1Beta1`生成器生成[運行時對象]（https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/run.go# L59）來自我們提供的參數。 “運行時對象”是資源的通用術語。 

### API組和版本協商

在我們繼續之前，值得指出的是Kubernetes使用_versioned_ API，該API被歸類為“API組”。 API組旨在對類似資源進行分類，以便更容易推理。它還為單個單片API提供了更好的替代方案。部署的API組名為`apps`，其最新版本為`v1`。這就是您需要在部署清單頂部鍵入“apiVersion：apps / v1”的原因。

無論如何......在kubectl生成運行時對象之後，它開始[找到合適的API組和版本]（https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go# L580-L597）然後[組裝版本化客戶端]（https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go#L598）知道各種REST資源的語義。此發現階段稱為版本協商，涉及kubectl掃描遠程API上的`/ apis`路徑以檢索所有可能的API組。由於kube-apiserver在此路徑中公開其架構文檔（採用OpenAPI格式），因此客戶端可以輕鬆執行自己的發現。 

為了提高性能，還kubectl（https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/util/factory_client_access.go#L117）所述的OpenAPI架構緩存]到`〜/ .kube / cache / discovery`目錄。如果要查看此API發現的實際操作，請嘗試刪除該目錄並運行`-v`標誌變為最大值的命令。您將看到嘗試查找這些API版本的所有HTTP請求。有很多！

最後一步是實際[發送]（https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go#L628）HTTP請求。一旦它這樣做並獲得成功的響應，kubectl將打印出一條成功消息[基於所需的輸出格式]（https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/ run.go＃L403-L407）。

###客戶端身份驗證

我們在上一步中未提及的一件事是客戶端身份驗證（這是在發送HTTP請求之前處理的），所以我們現在來看看。

為了成功發送請求，kubectl需要能夠進行身份驗證。用戶憑據幾乎總是存儲在磁盤上的`kubeconfig`文件中，但該文件可以存儲在不同的位置。要找到它，kubectl會執行以下操作：

- 如果提供了`--kubeconfig`標誌，請使用它。
- 如果定義了`$ KUBECONFIG`環境變量，請使用它。
- 否則查看[可預測的主目錄]（https://github.com/kubernetes/client-go/blob/master/tools/clientcmd/loader.go#L52），如`〜/ .kube`，並使用找到第一個文件。

解析文件後，它將確定要使用的當前上下文，要指向的當前集群以及與當前用戶關聯的任何身份驗證信息。如果用戶提供了特定於標誌的值（例如`--username`），則這些值優先，並將覆蓋kubeconfig中指定的值。一旦有了這些信息，kubectl就會填充客戶端的配置，以便它能夠適當地修飾HTTP請求：

- 使用[`tls.TLSConfig`]發送x509證書（https://github.com/kubernetes/client-go /blob/82aa063804cf055e16e8911250f888bc216e8b61/rest/transport.go#L80-L89)(這也包括根CA）
- 承載令牌[發送]（https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport /round_trippers.go#L314）中的“授權”HTTP標頭
-用戶名和密碼[發送]（https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L223）經由HTTP基本身份驗證
-  OpenID身份驗證過程由用戶事先手動處理，生成一個像持有者令牌一樣發送的令牌

## kube-apiserver

###身份驗證

所以我們的請求已被發送，萬歲！接下來是什麼？這是kube-apiserver進入圖片的地方。正如我們已經提到的，kube-apiserver是客戶端和系統組件用來持久化和檢索集群狀態的主要接口。要執行其功能，它需要能夠驗證請求者是否是他們所說的人。此過程稱為身份驗證。

apiserver如何驗證請求？當服務器首次啟動時，它會查看用戶提供的所有[CLI標誌]（https://kubernetes.io/docs/admin/kube-apiserver/）並組裝合適的驗證器列表。讓我們舉個例子：如果傳入`--client-ca-file`，它會附加x509身份驗證器;如果它看到提供了`--token-auth-file`，它會將令牌認證者附加到列表中。每次收到請求時，都會[通過驗證器鏈運行直到成功]（https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/union/union.go#L54） ： 

-  [x509處理程序]（https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/x509/x509.go#L60）將驗證HTTP請求是否使用TLS密鑰進行編碼由CA根證書籤名
-  [bearer token handler]（https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/bearertoken/bearertoken.go#L38）將驗證提供的令牌（在HTTP授權報頭中指定）存在於由`--token-AUTH-file`指定磁盤文件中
-的[基本驗證處理]（https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/plugin/ pkg / authenticator / request / basicauth / basicauth.go＃L37）將同樣確保HTTP請求t的基本身份驗證憑據與其自身的本地狀態匹配。

如果_every_ authenticator失敗，[請求失敗]（https://github.com/kubernetes/apiserver/blob/20bfbdf738a0643fe77ffd527b88034dcde1b8e3/pkg/authentication/request/union/union.go#L71）並返回聚合錯誤。如果驗證成功，則會從請求中刪除“Authorization”標頭，並添加[用戶信息]（https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authentication.go#L71 -L75）其背景。這為將來的步驟（例如授權和准入控制器）提供了訪問先前建立的用戶身份的能力。 

###授權

好的，請求已經發送，kube-apiserver已成功驗證我們是誰。終於解脫了！但是，我們還沒有完成。我們可能是我們所說的人，但我們是否有權執行此操作？畢竟，身份和許可並不是一回事。為了讓我們繼續，kube-apiserver需要授權我們。

kube-apiserver處理授權的方式與身份驗證非常相似：基於標誌輸入，它將匯集一系列授權者，這些授權者將針對每個傳入請求運行。如果_all_ authorizers拒絕請求，則該請求將導致“Forbidden”響應並且[不再進一步]（https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authorization.go#L60 ）。如果單個授權人批准，則請求繼續。

與V1.8出貨授權人的一些例子是：

- [網絡掛接]（https://github.com/kubernetes/apiserver/blob/d299c880c4e33854f8c45bdd7ab599fb54cbe575/plugin/pkg/authorizer/webhook/webhook.go#L143），該交互使用群集外HTTP（S）服務;
-  [ABAC]（https://github.com/kubernetes/kubernetes/blob/77b83e446b4e655a71c315ad3f3890dc2a220ccf/pkg/auth/authorizer/abac/abac.go#L223），它強制執行靜態文件中定義的策略;
- [RBAC]（https://github.com/kubernetes/kubernetes/blob/8db5ca1fbb280035b126faf0cd7f0420cec5b2b6/plugin/pkg/auth/authorizer/rbac/rbac.go#L43），其強制其由管理員作為K8S加入RBAC角色資源
- [節點（https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/plugin/pkg/auth/authorizer/node/node_authorizer.go#L67），這確保了節點的客戶端，即kubelet，可以僅訪問託管在其自身上的資源。

查看每個人的“授權”方法，看看它們是如何工作的！

###准入控制

好的，所以到目前為止我們已經過認證並已獲得kube-apiserver的授權。那剩下什麼了？從kube-apiserver的角度來看，它相信我們是誰並允許我們繼續，但對於Kubernetes，系統的其他部分對應該和不應該允許發生什麼有強烈的意見。這是[准入控制器]（https://kubernetes.io/docs/admin/admission-controllers/#what-are-they）進入圖片的地方。

雖然授權的重點是回答用戶是否具有權限，但是接納控制器會攔截該請求，以確保它符合群集的更廣泛期望和規則。它們是對象持久化到etcd之前的最後一個控制堡壘，因此它們封裝了剩餘的系統檢查以確保操作不會產生意外或負面結果。

准入控制器的工作方式類似於認證者和授權者的工作方式，但有一個區別：與認證者和授權者鏈不同，如果單個准入控制器失敗，整個鏈斷開，請求將失敗。 

准入控制器設計的真正優勢在於它致力於提升可擴展性。每個控制器都作為插件存儲在[`plugin / pkg / admission`目錄]（https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission）中，並且滿足於一個小的接口。然後將每個編譯成主要的kubernetes二進製文件。 

准入控制器通常分為資源管理，安全性，默認和參照一致性。以下是一些只關注資源管理的准入控制器示例：

- “InitialResources”根據過去的使用情況為容器的資源設置默認資源限制; 
- `LimitRanger`設置容器請求和限制的默認值，或強制某些資源的上限（不超過2GB的內存，默認為512MB）; 
- `ResourceQuota`計算並拒絕命名空間中的許多對象（pods，rc，服務負載均衡器）或總消耗資源（cpu，內存，磁盤）。

## etcd

到目前為止，Kubernetes已經完全審查了傳入的請求，並允許它出現並繁榮。在下一步中，kube-apiserver反序列化HTTP請求，從它們構造運行時對象（有點像kubectl的生成器的逆過程），並將它們持久化到數據存儲區。讓我們稍微打破一下。

kube-apiserver如何知道在接受我們的請求時該怎麼做？好吧，在提供任何請求之前，會發生一系列非常複雜的步驟。讓我們從第一次運行二進製文件開始：運行

1. `kube-apiserver`二進製文件時，它[創建一個服務器鏈]（https://github.com/kubernetes/kubernetes/blob/master /cmd/kube-apiserver/app/server.go#L119），允許使用apiserver聚合。這基本上是一種支持多個apiservers的方式（我們不需要擔心這個）。
1. 當發生這種情況時，創建[通用apiserver]（https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L149）作為默認實現。 
1. 生成的OpenAPI模式填充[apiserver的配置]（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/config.go#L149）。
1. 然後kube-apiserver迭代架構中指定的所有API組並配置[存儲提供程序]（https://github.com/kubernetes/kubernetes/blob/c7a1a061c3dc5acabcc0c35b3b96a6935dccf546/pkg/master/master.go#L410）對於每個用作通用存儲抽象的。這就是kube-apiserver在訪問或改變資源狀態時所說的內容。
1. 對於每一個API組也通過每個組版本的迭代並[安裝REST映射（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/groupversion.go#L92），用於每條HTTP路由。這允許kube-apiserver映射請求，並且一旦找到匹配就能夠委託給正確的邏輯。
1. 對於我們的特定使用情況下，[POST處理程序（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/installer.go#L710）被註冊時，這反過來將委託給[創建資源處理程序]（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37）。

到目前為止，kube-apiserver完全了解存在哪些路由，並且具有內部映射，以便在請求匹配時調用哪些處理程序和存儲提供程序。多麼聰明的餅乾。現在讓我們假設我們的HTTP請求已經飛入：

1. 如果處理程序鏈可以將請求與設置模式（即我們註冊的路由）匹配，它將[調度專用處理程序]（https://github.com/為路線註冊的kubernetes / apiserver / blob / 7001bc4df8883d4a0ec84cd4b2117655a0009b6c / pkg / server / handler.go＃L143）。否則它會回到[基於路徑的處理程序]（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L248）（當你打電話時會發生這種情況） / apis`）。如果沒有為該路徑註冊處理程序，則調用[未找到處理程序]（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L254），這將導致404.
1. 幸運的是，我們有一個名為[`createHandler`]的註冊路線（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37）！它有什麼作用？那麼它將首先解碼HTTP請求並執行基本驗證，例如確保它們提供的JSON與我們對版本化API資源的期望相關。
1. 審核和最終入場[將發生（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L93-L104）。 
1. 資源將被[保存到ETCD]（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L111）由[委託給存儲提供者（HTTPS ：//github.com/kubernetes/apiserver/blob/19667a1afc13cc13930c40a20f2c12bbdcaaa246/pkg/registry/generic/registry/store.go#L327）。通常，etcd鍵將是`<namespace> / <name>`的形式，但這是可配置的。
1. 捕獲任何創建錯誤，最後，存儲提供程序執行“get”調用以確保實際創建對象。然後，如果需要額外的最終化，它將調用任何後創建處理程序和裝飾器。
1. HTTP響應[構造]（https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L131-L142）並送回。

那是很多步驟！在兔子洞中跟踪碎屑是非常了不起的，因為我們意識到了施藥者實際做了多少工作。總結一下：我們的部署資源現在存在於etcd中。但只是在工作中拋出一個扳手，你將無法看到它......

##初始化器

將一個對象持久保存到數據存儲區後，它不會被apiserver完全看到或者被安排到一系列[intializers]（https://kubernetes.io/docs/admin/extensible-admission-controllers/#initializers）已經運行。初始化程序是與資源類型相關聯的控制器，並在資源可用於外部世界之前對資源執行邏輯。如果資源類型註冊了零初始值設定項，則跳過此初始化步驟，並立即使資源可見。 

正如[許多偉大的博客文章]（https://ahmet.im/blog/initializers/）所指出的，這是一個強大的功能，因為它允許我們執行通用引導操作。示例可以是：

- 將代理邊車容器注入到暴露端口80的Pod中，或者使用特定註釋。
- 將具有測試證書的捲注入特定命名空間中的所有Pod。
- 如果密鑰短於20個字符（例如密碼），請阻止其創建。

`initializerConfiguration`對象允許您聲明哪些初始值設定項應針對某些資源類型運行。想像一下，我們希望每次創建Pod時都運行自定義初始化程序，我們會這樣做：

```
apiVersion：admissionregistration.k8s.io/v1alpha1
kind：InitializerConfiguration
metadata：
  name：custom-pod-initializer
initializers：
  -名稱：podimage.example.com
    規則：
      - apiGroups：
          -
        “”apiVersions：
          - V1
        資源：
          -莢
```

創建此配置後，將定制吊艙initializer`追加`每莢的`metadata.initializers.pending `領域。初始化程序控制器已經部署，並將定期掃描新的Pod。當初始化程序在Pod的掛起字段中檢測到其名稱時，它將執行其邏輯。完成其過程後，它會從掛起列表中刪除其名稱。只有名稱在列表中第一個的初始化程序才可以在資源上運行。當所有初始化程序都完成且“pending”字段為空時，該對象將被視為已初始化。

你眼中的老鷹可能會發現一個潛在的問題。如果kube-apiserver不能看到這些資源，那麼userland控制器如何處理資源？為了解決這個問題，kube-apiserver公開了一個`？includeUninitialized`查詢參數，該參數返回_all_對象，甚至是單元化對象。 

## Control loops

### Deployments controller

在此階段，我們的部署記錄存在於etcd中，並且所有初始化邏輯都已完成。接下來的階段涉及我們設置Kubernetes所依賴的資源拓撲。當我們考慮它時，Deployment實際上只是ReplicaSet的集合，而ReplicaSet是Pod的集合。那麼Kubernetes如何從一個HTTP請求創建這個層次結構呢？這是Kubernetes的內置控制器接管的地方。

Kubernetes在整個系統中充分利用“控制器”。控制器是一個異步腳本，用於的進行協調 
將Kubernetes系統當前狀態與所需狀態。每個控制器都有一個小的責任，由`kube-controller-manager`組件並行運行。讓我們自己介紹一下接管的第一個部署控制器。

將部署記錄存儲到etcd並初始化後，通過kube-apiserver可以看到它。當這個新資源可用時，Deployment控制器會檢測到它，它的工作是監聽部署記錄的更改。在我們的例子中，控制器[註冊一個特定的回調]（https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L122）通過線人創建事件（見下文）有關這是什麼的更多信息）。 

當我們的部署首次可用時，將執行此處理程序，並將[將對象添加到內部工作隊列]開始執行（https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller。去＃L170）。當它處理我們的對象時，控制器將[檢查我們的部署]（https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L572）和[實現]（https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L633）沒有與之關聯的ReplicaSet或Pod記錄。它通過使用標籤選擇器查詢kube-apiserver來實現此目的。有趣的是，這個同步過程是狀態不可知的：它以與現有記錄相同的方式協調新記錄。

在意識到不存在之後，它將開始[縮放過程]（https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L385）以開始解析狀態。它通過推出（例如創建）ReplicaSet資源，為其分配標籤選擇器，並為其提供修訂號1來完成此操作.ReplicaSet的PodSpec是從Deployment的清單以及其他相關元數據中復制的。有時，部署記錄也需要在此之後更新（例如，如果設置了進度截止日期）。 

[狀態隨後更新]（https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L70）然後重新進入相同的協調循環，等待部署匹配所需的完成狀態。由於Deployment控制器只關心創建ReplicaSet，因此需要由下一個控制器ReplicaSet控制器繼續進行此協調階段。 

### ReplicaSets控制器

在上一步中，Deployments控制器創建了Deployment的第一個ReplicaSet，但我們仍然沒有Pod。這是ReplicaSet控制器發揮作用的地方！它的工作是監視ReplicaSet及其相關資源（Pod）的生命週期。與大多數其他控制器一樣，它通過觸發某些事件的處理程序來實現。

我們感興趣的事件是創造。創建ReplicaSet時（由部署控制器提供）RS控制器[檢查狀態]（https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L583）新的ReplicaSet並意識到存在與需要之間存在偏差。然後，它試圖通過[凸起pods的數量]（https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L460）來協調這個狀態，這些狀態屬於ReplicaSet。它開始小心地創建它們，確保ReplicaSet的突發計數（它從其父部署繼承）始終匹配。 

創建Pod的操作[也是批處理]（https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L487），從“SlowStartInitialBatchSize”開始，每次成功迭代加倍在一種“慢啟動”操作。這旨在減少當存在大量pod啟動失敗時（例如，由於資源配額）而使用不必要的HTTP請求淹沒kube-apiserver的風險。如果我們要失敗，我們也可能優雅地失敗，對其他系統組件的影響最小！ 

Kubernetes通過Owner References（子資源中的一個字段，它引用其父項的ID）強制執行對象層次結構。一旦控制器管理的資源被刪除（級聯刪除），這不僅可以確保子資源被垃圾收集，還可以為父資源提供一種有效的方式來防止他們的孩子（想像兩個潛在父母認為的情景）他們擁有同一個孩子！）。 

所有者參考設計的另一個微妙好處是它是有狀態的：如果任何控制器要重啟，那麼停機時間不會影響更寬的系統，因為資源拓撲獨立於控制器。這種對隔離的關注也滲透到控制器本身的設計中：它們不應該在它們沒有明確擁有的資源上運行。相反，控制器應該選擇其所有權斷言，非干擾和非共享。 

無論如何，回到所有者參考！有時在系統中存在“孤立”資源，通常在以下情況下發生：

1. 刪除父項但不刪除子項
2. 垃圾收集策略禁止刪除子項

發生這種情況時，控制器將確保新父項採用孤兒。多個父母可以競賽收養孩子，但只有一個會成功（其他人會收到驗證錯誤）。 

### Informers

正如您可能已經註意到的，某些控制器（如RBAC授權程序或Deployment控制器）需要檢索群集狀態才能運行。為了返回RBAC授權器的示例，我們知道當請求進入時，驗證器將保存用戶狀態的初始表示以供以後使用。然後，RBAC授權程序將使用它來檢索與etcd中的用戶關聯的所有角色和角色綁定。控制器如何訪問和修改這些資源？事實證明這是一個常見的用例，並在Kubernetes中通過告密者解決。 

informer是一種模式，允許控制器訂閱存儲事件並輕鬆列出他們感興趣的資源。除了提供一個很好用的抽象，它還需要處理許多諸如緩存之類的細節（緩存很重要，因為它減少了不必要的kube-apiserver連接，並減少了服務器端和控制端端的重複序列化成本）。通過使用這種設計，它還允許控制器以線程安全的方式進行交互，而不必擔心踩到任何其他人的腳趾。 

有關告密者如何與控制器相關工作的更多信息，請查看[此博客文章]（http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores）。

### Scheduler

在所有控制器運行之後，我們有一個Deployment，一個ReplicaSet和三個存儲在etcd中並通過kube-apiserver提供的Pod。但是，我們的pod仍然處於“Pending”狀態，因為它們還沒有安排到Node。解決此問題的最終控制器是調度程序。

調度程序作為控制平面的獨立組件運行，並以與其他控制器相同的方式運行：它偵聽事件並嘗試協調狀態。在這種情況下，它的[過濾器盒]（https://github.com/kubernetes/kubernetes/blob/master/plugin/pkg/scheduler/factory/factory.go#L190）中有一個空的`NodeName`字段PodSpec並嘗試找到pod可以駐留的合適節點。 

為了找到合適的容器，使用特定的調度算法。默認調度算法的工作方式如下：

1. 當調度程序啟動時，[註冊默認謂詞鏈] [https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/ algorithmprovider /默認/ defaults.go＃L65-L81）。這些謂詞是有效的是，當進行評價，根據其是否適合於[過濾器節點（https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L117）函數託管一個吊艙。例如，如果PodSpec顯式請求CPU或RAM資源，並且節點由於容量不足而無法滿足這些請求，則將取消選擇Pod（資源容量計算為_total capacity_減去_sum資源請求數_sum目前正在運行容器）。



1. 一旦選擇了適當的節點，就會運行一系列[優先級功能]（https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L354-L360 ）對剩餘的節點進行排序以確定其適用性。例如，為了在整個系統中分散工作負載，它將支持資源請求少於其他節點的節點（因為這表明運行的工作負載較少）。當它運行這些函數時，它為每個節點分配一個數字排名。然後選擇排名最高的節點進行調度。

一旦算法找到一個節點，然後調度程序[創建一個綁定]（https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/scheduler.go#L336-L342）對象的名稱和UID與Pod匹配，其ObjectReference字段包含所選節點的名稱。然後通過POST請求[發送到apiserver]（https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/factory/factory.go#L1095）。

當kube-apiserver收到此Binding對象時，註冊表會反序列化該對象並更新Pod對像上的以下字段：它[設置NodeName]（https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry /core/pod/storage/storage.go#L170）到ObjectReference中的那個，它[添加任何相關的註釋]（https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/ pod / storage / storage.go＃L174-L176）和[sets]（https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L177- L180）其“PodScheduled”狀態條件為“True”。 

一旦調度程序將Pod調度到節點，該節點上的kubelet就可以接管開始部署。精彩！

**旁注：自定義調度程序：**有趣的是，謂詞和優先級函數都是可擴展的，可以使用`--policy-config-file`標誌來定義。這引入了一定程度的靈活性。管理員還可以在獨立部署中運行自定義調度程序（具有自定義處理邏輯的控制器）。如果PodSpec包含`schedulerName`，Kubernetes會將該pod的調度移交給在該名稱下註冊的任何調度程序。

## kubelet同上 

### Pod

好了，主控制器循環已經完成，p！讓我們總結一下：HTTP請求通過了身份驗證，授權和准入控制階段;部署，ReplicaSet和三個Pod資源被持久化到etcd;一系列初始化程序運行;最後，每個Pod被安排到一個合適的節點。然而，到目前為止，我們一直在推理的狀態純粹存在於etcd中。接下來的步驟涉及在工作節點之間分配狀態，這是像Kubernetes這樣的分佈式系統的全部要點！發生這種情況的方法是通過一個名為kubelet的組件。讓我們開始！

kubelet是一個在Kubernetes集群中的每個節點上運行的代理，負責管理Pods的生命週期。這意味著它處理“Pod”（實際上只是一個Kubernetes概念）的抽象與其構建塊，容器之間的所有轉換邏輯。它還處理有關安裝卷，容器日誌記錄，垃圾收集以及許多其他重要事項的所有相關邏輯。

一個有用的方法來考慮kubelet再次像一個控制器！它每隔20秒從kube-apiserver查詢Pods（這是可配置的），過濾那些`NodeName` [匹配名稱]的那些（https://github.com/kubernetes/kubernetes/blob/3b66adb8bc6929e1205bcb2bc32f380c39be8381/pkg/kubelet/config運行kubelet的節點的/apiserver.go#L34）。一旦它具有該列表，它就會通過與其自己的內部緩存進行比較來檢測新添加，並在存在任何差異時開始同步狀態。讓我們來看看同步過程是什麼樣的：

1. 如果正在創建pod（我們的是！），它[註冊一些啟動指標]（https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/ pkg / kubelet / kubelet.go＃L1519）在Prometheus中用於跟踪pod延遲。
1. 然後[生成PodStatus]（https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1287）對象，該對象表示Pod當前階段的狀態。 Pod的階段是Pod在其生命週期中的位置的高級摘要。示例包括`Pending`，`Running`，`Succeeded`，`Failed`和`Unknown`。生成這種狀態非常複雜，所以讓我們深入了解會發生什麼：
    - 首先，順序執行一系列`PodSyncHandlers`。每個處理程序檢查Pod是否仍應駐留在節點上。如果他們中的任何人決定Pod不再屬於那裡，Pod的階段[將改變]（https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1293-L1297） `PodFailed`，它最終會從Node中逐出。這些例子包括在超過`activeDeadlineSeconds`之後驅逐一個Pod（在Jobs期間使用）。
    - 接下來，Pod的階段由其init和真實容器的狀態決定。由於我們的容器尚未啟動，因此容器被歸類為[等待]（https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1244）。任何具有等待容器的Pod都具有[`Pending`]階段（https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1258-L1261）。
    - 最後，Pod條件由其容器的條件決定。由於我們的容器尚未由容器運行時創建，因此它將[將PodReady`條件設置為False]（https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/status/generate 。去＃L70-L81）。
1. 生成PodStatus後，它將被發送到Pod的狀態管理器，該管理器的任務是通過apiserver異步更新etcd記錄。
1. 接下來，運行一系列准入處理程序以確保pod具有正確的安全權限。這些包括強制執行[AppArmor配置文件和`NO_NEW_PRIVS`]（https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L883-L884）。在此階段被拒絕的Pod將無限期地保持在“待定”狀態。
1. 如果指定了`cgroups-per-qos`運行時標誌，則kubelet將為pod創建cgroup並應用資源參數。這是為了實現更好的服務質量（QoS）處理。
1. 數據目錄[為pod創建]（https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L772）。這些包括pod目錄（通常是`/ var / run / kubelet / pods / <podID>`），其卷目錄（`<podDir> / volumes`）及其插件目錄（`<podDir> / plugins`）。
1. 捲管理器將[附加並等待]（https://github.com/kubernetes/kubernetes/blob/2723e06a251a4ec3ef241397217e73fa782b0b98/pkg/kubelet/volumemanager/volume_manager.go#L330）查看`Spec.Volumes中定義的任何相關卷`。根據要安裝的捲的類型，某些pod需要等待更長時間（例如雲或NFS卷）。
1. 在`Spec.ImagePullSecrets`中定義的所有秘密都是[從apiserver中檢索]（https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L788），以便以後可以被注入容器中。
1. 容器運行時然後運行容器（下面更詳細地描述）。

### CRI和暫停容器

我們現在處於完成大部分設置並且容器已準備好啟動的位置。執行此啟動的軟件稱為Container Runtime（`docker`或`rkt`是示例）。

為了更加可擴展，自v1.5.0以來的kubelet一直使用一個名為CRI（容器運行時接口）的概念來與具體的容器運行時進行交互。簡而言之，CRI提供了kubelet和特定運行時實現之間的抽象。通過[協議緩衝區]（https://github.com/google/protobuf）（它就像一個更快的JSON）和[gRPC API]（https://grpc.io/）（一種類型的API）進行通信。適合執行Kubernetes操作）。這是一個非常酷的想法，因為通過在kubelet和運行時之間使用已定義的契約，容器如何編排的實際實現細節變得非常無關緊要。重要的是合同。這允許以最小的開銷添加新的運行時，因為沒有核心Kubernetes代碼需要更改！

足夠的離題，讓我們回到部署我們的容器......當一個pod首次啟動時，kubelet [調用`RunPodSandbox`]（https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/kubelet/kuberuntime /kuberuntime_sandbox.go#L51）遠程過程命令（RPC）。 “沙箱”是描述一組容器的CRI術語，在Kubernetes的說法中，你猜對了，就是一個容器。這個術語是故意模糊的，因此它不會失去其他可能實際上不使用容器的運行時的意義（想像一下基於虛擬機管理程序的運行時，沙箱可能是一個VM）。 

在我們的例子中，我們使用的是Docker。在此運行時中，創建沙箱涉及創建“暫停”容器。暫停容器像Pod的所有其他容器的父級一樣，因為它承載了工作負載容器最終將使用的許多pod級資源。這些“資源”是Linux命名空間（IPC，網絡，PID）。如果你不熟悉容器在Linux中的工作方式，那就讓我們快速回顧一下。 Linux內核具有命名空間的概念，它允許主機操作系統分割出一組專用資源（例如CPU或內存）並將其提供給一個進程，就好像它是世界上唯一使用它們的東西一樣。 Cgroup在這裡也很重要，因為它們是Linux管理資源分配的方式（它有點像警察資源使用的警察）。 Docker使用這兩種內核功能來託管一個保證資源和強制隔離的進程。有關更多信息，請查看b0rk的精彩帖子：[甚至是什麼容器？]（https://jvns.ca/blog/2016/10/10/what-even-is-a-container/）。 

“pause”容器提供了一種託管所有這些命名空間的方法，並允許子容器共享它們。通過成為同一網絡命名空間的一部分，一個好處是同一個pod中的容器可以使用`localhost`相互引用。暫停容器的_second_角色與PID命名空間的工作方式有關。在這些類型的命名空間中，進程形成一個分層樹，頂部的“init”進程負責“收穫”死進程。有關其工作原理的更多信息，請查看[偉大的博客文章]（https://www.ianlewis.org/en/almighty-pause-container）。創建暫停容器後，將其檢查點到磁盤，然後啟動。

### CNI和pod網絡

我們的Pod現在有它的骨架：一個暫停容器，它託管所有命名空間以允許pod間通信。但網絡如何運作以及如何建立？ 

當kubelet為pod設置網絡時，它將任務委託給“CNI”插件。 CNI代表Container Network Interface，其運行方式與Container Runtime Interface類似。簡而言之，CNI是一種抽象，允許不同的網絡提供商對容器使用不同的網絡實現。插件已註冊，並且kubelet通過將JSON數據（配置文件位於`/ etc / cni / net.d`中）通過stdin流式傳輸到相關的CNI二進製文件（位於`/ opt / cni / bin`）來與它們進行交互。這是JSON配置的一個例子：

```yaml
{
    “cniVersion”：“0.3.1”，
    “name”：“bridge”，
    “type”：“bridge”，
    “bridge”：“cnio0”，
    “isGateway “：true，
    ”ipMasq“：true，
    ”ipam“：{
        ”type“：”host-local“，
        ”range“：[
          [{”subnet“：”$ {POD_CIDR}“}]
        ]，
        ”routes“： [{“dst”：“0.0.0.0/0”}]
    }
}
```

它還通過`CNI_ARGS`環境變量為pod指定其他元數據，例如其名稱和命名空間。

接下來發生的事情取決於CNI插件，但讓我們來看看`bridge` CNI插件：

1. 插件將首先在根網絡命名空間中設置一個本地Linux橋，以服務該主機上的所有容器
1. 然後將一個接口（一個veth對的一端）插入暫停容器的網絡命名空間，並將另一端連接到網橋。考慮一個veth對的最好方法就像一個大管：一邊連接到容器，另一邊是根網絡命名空間，允許數據包在中間傳輸。 
1.然後應該為暫停容器的界面分配IP並設置路由。這將導致Pod擁有自己的IP地址。 IP分配委託給為JSON配置指定的IPAM提供程序。 
    -  IPAM插件類似於主網絡插件：它們通過二進製文件調用並具有標準化接口。每個必須確定容器接口的IP /子網以及網關和路由，並將此信息返回給主插件。最常見的IPAM插件稱為“host-local”，並從預定義的一組地址範圍中分配IP地址。它將狀態本地存儲在主機文件系統上，從而確保單個主機上IP地址的唯一性。
1. 對於DNS，kubelet將為CNI插件指定內部DNS服務器IP地址，這將確保正確設置容器的`resolv.conf`文件。 

一旦完成該過程，插件將把JSON數據返回到kubelet，指示操作的結果。 

###主機間網絡

到目前為止，我們已經描述了容器如何連接到主機，但主機如何通信？如果不同機器上的兩個Pod想要通信，這顯然會發生。

這通常使用稱為覆蓋網絡的概念來實現，這是一種跨多個主機動態同步路由的方法。一個流行的覆蓋網絡提供商是法蘭絨。安裝後，其核心職責是在群集中的多個節點之間提供第3層IPv4網絡。法蘭絨不控制容器如何與主機聯網（這是CNI記住的工作），而是如何在主機之間傳輸流量。為此，它為主機選擇一個子網並將其註冊到etcd。然後，它保留集群路由的本地表示，並將傳出的數據包封裝在UDP數據報中，確保它到達正確的主機。有關更多信息，請查看[CoreOS的文檔]（https://github.com/coreos/flannel）。

###容器啟動

所有網絡惡作劇都已完成並且已經完成。還剩什麼？我們需要實際啟動工作負載容器。 

沙箱完成初始化並處於活動狀態後，kubelet可以開始為其創建容器。它首先[啟動任何初始容器]（https://github.com/kubernetes/kubernetes/blob/5adfb24f8f46a0d57eb9a15b158db46f9f46f0d80/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L690），如PodSpec中所定義，然後啟動主容器他們自己。這樣做的過程如下： 

1. [拉圖像]（https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55646459/pkg/kubelet/kuberuntime/kuberuntime_container.go#L90）為容器。 PodSpec中定義的任何秘密都用於私有註冊表。
1. 通過CRI創建容器（https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L115）。它通過從父PodSpec填充`ContainerConfig`結構（其中定義了命令，圖像，標籤，安裝，設備，環境變量等），然後通過protobufs將其發送到CRI插件來實現。對於Docker，它反序列化有效負載並填充其自己的配置結構以發送到Daemon API。在此過程中，它將一些元數據標籤（例如容器類型，日誌路徑，沙箱ID）應用於容器。
1. 然後它使用CPU管理器註冊容器，這是1.8中的一個新的alpha功能，它通過使用`UpdateContainerResources` CRI方法將容器分配給本地節點上的CPU組。
1. 然後[啟動]容器（https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L135）。
1. 如果註冊了任何啟動後容器生命週期掛鉤，則它們是[運行]（https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L156-L170）。鉤子可以是`Exec`類型（在容器內執行特定命令）或`HTTP`（對容器端點執行HTTP請求）。如果PostStart掛鉤運行，掛起或失敗的時間太長，容器將永遠不會達到“running”狀態。

##總結

好吧，phew。完成。 Finito。

畢竟，我們應該在一個或多個工作節點上運行3個容器。所有的網絡，捲和秘密都由kubelet填充，並通過CRI插件製作成容器。
