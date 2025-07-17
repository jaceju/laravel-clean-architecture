
# **為大型模組化 Laravel 應用程式打造務實的架構**

## **執行摘要**

本報告旨在深入探討為大型、模組化 Laravel 專案設計架構時所面臨的核心挑戰：如何在嚴謹的軟體設計模式與 Laravel 框架的務實特性之間取得平衡。您所規劃的架構，包含控制器（Controller）、服務（Service）、倉儲（Repository）與資料傳輸物件（Data Transfer Object, DTO），展現了對分層架構的深刻理解。然而，您也敏銳地察覺到，在真實世界的開發場景中，教條式地應用這些模式可能會與框架的慣用手法（idioms）產生摩擦，甚至導致過度設計。

本報告將全面分析您提出的架構，並針對您所面臨的困境——包括 DTO 的必要性、Repository 的過度設計風險，以及如何優雅地處理如 $request->user() 這類框架原生操作——提出一套精煉且務實的解決方案。

最終的建議藍圖將圍繞以下核心原則構建：

1. **將 DTO 提升為模組間的 API 契約（API Contract）**：在 nwidart/laravel-modules 的模組化架構下，DTO 不僅僅是資料載體，更是確保模組間低耦合與清晰邊界的基石。  
2. **將 Repository 重新定義為 I/O 閘道（I/O Gateway）**：跳脫「僅為 Eloquent 封裝」的狹隘思維，將其職責擴展至管理所有與外部系統的資料互動，包括資料庫、S3、CDN 或第三方 API。  
3. **採用 Service 與 Action 的混合模式處理業務邏輯**：針對複雜的業務流程使用 Service，而對於單一、獨立的操作則採用更輕量的 Action 類別，以避免不必要的程式碼樣板（boilerplate）。  
4. **將 Controller 明確指定為框架邊界的中介者（Framework Boundary Mediator）**：允許 Controller 使用 Laravel 的原生功能（如 Request、Auth Facade），但其核心職責是將這些框架物件「轉譯」為應用程式核心層能夠理解的純粹輸入（如 DTO 或純量值），而非直接傳遞。

此架構旨在為您的團隊提供一個清晰、可擴展且易於維護的開發範本，它既能發揮分層設計的優勢，又能充分利用 Laravel 框架的強大功能，而非與之對抗。

---

## **第一部分：對擬議分層架構的批判性分析**

本部分將解構您所提出的架構，驗證您的疑慮，並根據您的專案限制（特別是模組化特性與僅依賴 HTTP 測試的策略）重新定義每一層的目標。

### **第一節：資料傳輸物件（DTO）作為模組化的基石**

#### **1.1. 簡介：從資料載體到 API 契約**

您對 DTO 的定義——處理資源物件——是正確的，但在此專案的特定脈絡下，其重要性遠不止於此。當使用 nwidart/laravel-modules 建立一個模組化應用程式時 1，DTO 的首要角色是作為模組之間

**穩定、明確且不可變的 API 契約**。這個觀念的轉變，是理解 DTO 必要性的關鍵。

在一個由多個團隊並行開發不同模組的環境中，模組之間的邊界必須清晰且穩固 1。DTO 正是劃定這些邊界的技術實現。它定義了一個模組向外提供資料的「公開介面」，這個介面獨立於其內部的資料庫結構或實現細節。

#### **1.2. 為何 DTO 在您的模組化情境中是不可或缺的**

在大型模組化專案中，強制使用 DTO 並非出於對「純淨程式碼」的偏好，而是出於對架構穩定性的關鍵需求。

* **強制解耦（Enforcing Decoupling）**：設想一個場景，一個核心模組（例如 CoreUser 模組）將其 Eloquent Model User 直接傳遞給一個功能性模組（例如 Payments 模組）。如果 CoreUser 模組因為內部需求變更了 users 資料表的欄位，那麼 Payments 模組很可能會在不知情的情況下崩潰。這種緊密的耦合是模組化架構的頭號敵人。DTO 通過將模組的內部資料結構與其公開的資料契約分離，徹底解決了這個問題 4。  
  Payments 模組只依賴於穩定的 UserData DTO，而非易變的 User Model。  
* **防止洩漏的抽象（Preventing Leaky Abstractions）**：一個 Eloquent Model 不僅僅是資料，它還攜帶著其所有的關聯關係、存取器（accessors）、修改器（mutators）以及持久化方法（如 save(), delete()）。將 Model 傳遞到其他模組，等於洩漏了其所有的實現細節和行為能力。這違反了關注點分離原則。相比之下，DTO 是一個純粹的、沒有行為的資料結構，它確保了消費方模組只接收到它完成工作所必需的資料，不多也不少 7。  
* **為團隊協作定義的 API 契約**：對於在不同模組上工作的開發人員來說，DTO 提供了一個清晰、自我說明的介面。Payments 模組的開發者不需要去研究 CoreUser 模組的資料庫結構，只需查看 UserData DTO 的定義，就能確切知道可以從使用者模組獲得哪些資料。這極大地提高了團隊協作的效率和準確性 2。  
* **安全性與資料完整性**：DTO 允許您明確定義哪些資料應該被暴露，從而避免了如密碼、API token 等敏感欄位的意外洩漏。此外，您選擇的 spatie/laravel-data 套件非常適合將驗證規則和型別轉換邏輯集中在 DTO 內部，確保了資料在應用程式各層之間流動時的一致性與完整性 4。

#### **1.3. DTO 與 Laravel API Resources 的釐清**

一個常見的混淆點是 DTO 與 Laravel 原生的 JsonResource（API Resources）之間的關係。釐清這兩者的職責至關重要。

* **API Resources (JsonResource)** 的職責是為**最終的對外輸出**（例如，一個 JSON API 回應）進行資料格式化。它們的任務是將您應用程式的內部資料結構（可能是 DTO 或 Model）轉換成客戶端（如前端應用或第三方服務）所期望的確切格式 9。它們位於應用程式的最外層，是表示層（Presentation Layer）的一部分。  
* **DTO** 的職責是**應用程式內部各層與各模組之間的資料傳輸**。它們是您應用程式核心的「血液」，而 API Resources 則是應用程式的「皮膚」5。

一個正確的資料流通常是這樣的：  
Repository -> DTO -> Service/Action -> Controller -> API Resource -> JSON Response  
總結來說，您決定採用模組化架構，這本身就構成了強制使用 DTO 的最有力理由。在此情境下，DTO 不再是一個「可有可無」的選項，而是降低模組化單體（modular monolith）主要架構風險——模組間意外耦合——的關鍵工具。

### **第二節：將倉儲模式（Repository Pattern）重新構想為 I/O 閘道**

#### **2.1. 面對「過度設計」：作為 Eloquent 簡單包裝器的缺陷**

您對於 Repository 可能造成過度設計的擔憂是完全合理的。業界普遍認同，如果僅僅是創建一個 Repository 介面和對應的類別，而其方法只是簡單地代理對 Eloquent 的呼叫（例如，UserRepository->find($id) 內部只是執行 User::find($id)），那麼這個抽象層幾乎沒有帶來任何實質價值 12。

這種做法被廣泛批評，因為 Eloquent 本身已經是一個強大的抽象層（ORM）12。在其之上再添加一個功能重複的抽象層，是典型的過度設計。更重要的是，在您的專案中，由於不編寫針對 Repository 的單元測試，傳統上 Repository 模式最大的優勢——方便測試時進行模擬（mocking）——也就不復存在了。因此，如果 Repository 僅僅是 Eloquent 的包裝，那麼在您的專案中它確實是多餘的。

#### **2.2. 務實的重塑：從資料庫抽象到 I/O 閘道**

讓 Repository 模式在您的專案中變得極具價值的關鍵，在於擴展其職責範圍。您自己的定義——Repository：必須負責 I/O 或資料互動，包含：Model (Query), S3, CloudFront——已經為此提供了完美的線索。

我們應該將 Repository 的角色重新定義為一個**I/O 閘道抽象（I/O Gateway Abstraction）**。它的唯一目標是：**為應用程式核心提供一個單一、一致的介面，用以與任何外部資料源或資料槽（data sink）進行通訊。**

這意味著：

* 一個 UserRepository 可能與 MySQL 資料庫互動。  
* 一個 FileRepository 可能與 Amazon S3 互動。  
* 一個 CdnRepository 可能與 Amazon CloudFront 互動。  
* 一個 ThirdPartyCrmRepository 可能與外部的 CRM API 互動。

應用程式的其他部分（如 Services 和 Actions）不需要知道資料是*如何*或*從哪裡*被儲存或讀取的；它們只需要呼叫相應 Repository 上的方法即可 15。這種重新定義為 Repository 模式提供了一個強大的、完全獨立於單元測試的存續理由。它將所有外部 I/O 操作集中化和標準化，這對於一個大型應用程式的長期維護性和一致性來說，是一個巨大的益處。

#### **2.3. Repository 的職責與實現**

基於 I/O 閘道的概念，Repository 的職責應當非常明確：

* **輸入（Input）**：Repository 的方法應該接受純量值（如 ID、搜尋關鍵字）或簡單的參數物件（例如，一個 SearchCriteria DTO）。它們**絕不**應該接受一個 Request 物件，因為這會將其與 HTTP 層耦合。  
* **輸出（Output）**：Repository 的方法必須**總是**返回 DTO（或 DTO 的集合）。Repository 是原始資料（來自 Eloquent、S3 API 回應等）被轉換為應用程式內部穩定契約的邊界。一個 Eloquent Model 絕不應該「逃離」Repository 層。  
* **實現（Implementation）**：  
  * 一個 UserRepository 可能會有一個 find(int $id):?UserData 方法。其內部使用 User::find($id)，但如果找到了使用者，它會呼叫 UserData::fromModel($user) 來創建並返回 DTO。  
  * 一個 ProfileImageRepository 可能會有一個 store(int $userId, UploadedFile $file): ImageUrlData 方法。其內部處理與 S3 的互動，並返回一個包含新圖片 URL 的 DTO。

這種設計將 Repository 從一個可能被過度設計的模式，轉變為您架構中至關重要的元件。它不再是關於能否從 MySQL 切換到 PostgreSQL（這是一個很少發生的事件 13），而是關於如何以一種統一、可維護的方式處理資料庫讀寫、檔案上傳下載以及 CDN 互動。

### **第三節：業務邏輯核心：Service、Action 與純粹性**

#### **3.1. Service 層的角色**

Service 層的角色被確立為處理和編排**複雜業務邏輯**和工作流程的地方。當一個操作涉及多個步驟或需要與多個 Repository 互動時，Service 是理想的選擇 12。

一個典型的例子是 UserRegistrationService。它的 register 方法可能會執行以下一系列操作：

1. 呼叫 UserRepository->create() 來創建使用者記錄。  
2. 呼叫 ProfileRepository->createDefault() 來建立預設的個人資料。  
3. 呼叫 TeamRepository->assignToDefaultTeam() 將使用者加入預設團隊。  
4. 派發一個 WelcomeEmailJob 來發送歡迎郵件。

將這些步驟組合在一個 Service 中，可以確保業務流程的原子性和一致性，並使 Controller 保持簡潔。

#### **3.2. 用 Action 類別解決「純傳遞」問題**

您提出的 MembershipService 僅僅是呼叫 MemberRepository 的例子，是一個典型的「程式碼異味」（code smell），它表明所選的模式（Service）對於該任務來說過於笨重。

為了解決這個問題，我們引入**Action 類別**作為一個更細粒度、單一用途的替代方案 20。Action 是一個簡單的 PHP 類別，通常只有一個公開方法（如

execute 或 __invoke），用於執行一個特定的業務任務。

您所舉的例子可以被重構為一個 UpdateMemberActiveTokenAction 類別。Controller 將直接依賴注入並呼叫這個 Action。

```php
// app/Actions/Members/UpdateMemberActiveTokenAction.php  
namespace AppActionsMembers;

use AppRepositoriesMemberRepository;

class UpdateMemberActiveTokenAction  
{  
    public function __construct(protected MemberRepository $memberRepository) {}

    public function execute(int $memberId, string $token): void  
    {  
        $this->memberRepository->updateMemberActiveToken($memberId, $token);  
    }  
}

// app/Http/Controllers/MemberController.php  
class MemberController extends Controller  
{  
    public function updateToken(Request $request, UpdateMemberActiveTokenAction $action)  
    {  
        //... validation...  
        $action->execute($request->user()->id, $request->input('token'));  
        //... return response...  
    }  
}
```

這種方式消除了無意義的「純傳遞」Service，並創建了一個高內聚、自我說明的業務邏輯單元 22。這為您的團隊提供了一個清晰的選擇：對於簡單、離散的操作，使用 Action；對於複雜、多步驟的編排，使用 Service。這可以防止 Service 類別變得臃腫，充滿了數十個微小的方法。

#### **3.3. 純粹性原則：為何 Service 與 Action 必須與框架無關**

這是一條至關重要的規則。您在提案中提到 Service 可以操作 Session，我們強烈建議不要這樣做。Service 和 Action 構成了您應用程式的業務邏輯核心，它們應該是「純粹」且可重用的。它們不應該對傳遞機制（是 HTTP 請求、CLI 命令還是佇列任務）有任何了解 19。

這意味著它們**絕不能**依賴於框架特定的全域物件，如 Request、Session 或 Auth Facade。所有必要的資料都應該作為純量值或 DTO 通過方法參數傳遞進來 19。

這種純粹性帶來了巨大的好處：

* **可重用性**：一個 CLI 命令可以和 Controller 一樣，呼叫同一個 Action。  
* **可測試性**：雖然您目前不編寫單元測試，但這種設計使得未來若要添加測試變得極其容易，因為不需要模擬整個 HTTP 環境。  
* **關注點分離**：它確保了業務邏輯與應用程式的基礎設施（如 HTTP 層）完全分離，提高了程式碼的可維護性 18。

如果一個 Service 需要知道當前登入的使用者是誰，那麼 Controller 的職責就是從 Auth Facade 獲取使用者 ID，然後將這個 int 型別的 ID 傳遞給 Service 方法，而不是將整個 Auth Facade 或 User Model 注入到 Service 中。

---

## **第二部分：綜合一個平衡且務實的藍圖**

本部分將基於第一部分的精煉分層，構建一個完整、務實的架構藍圖。它為您的團隊提供了清晰的規則和啟發式方法，直接解決了您在框架整合和過度設計方面的兩難困境。

### **第四節：Controller 作為框架邊界的中介者**

#### **4.1. 定義 Controller 的真正角色**

本節提供了您所尋求的「平衡點」。Controller 的首要角色是作為**HTTP 世界與您應用程式核心邏輯之間的中介者**。它是框架基礎設施與您的業務邏輯相遇的邊界層 26。

因此，Controller 與框架提供的物件（如 IlluminateHttpRequest、IlluminateSupportFacadesAuth 和 IlluminateSupportFacadesGate）進行互動，不僅是可以接受的，而且是**被建議的**。這正是它們被設計出來的用途。

#### **4.2. 黃金法則：轉譯，而非傳遞**

維持清晰架構的關鍵規則是：**Controller 消費框架物件，但絕不將它們向下傳遞到應用程式核心（Service、Action、Repository）。**

Controller 的工作是**轉譯（Translate）**。

* 它接收 Request 物件，並將其 payload *轉譯*成一個 DTO。  
* 它使用 Auth Facade 來獲取已驗證使用者的 ID，並將這個純量 int 值傳遞給 Service。  
* 它使用 Gate 來檢查授權，然後再呼叫 Action 28。

這種方法讓您可以充分利用 Laravel 強大的功能（如表單請求驗證、認證系統等）29，同時又不會污染您的核心業務邏輯。

#### **4.3. 解決您的 logout 範例**

您的範例 class AuthController extends Controller { public function logout(Request $request) { $request->user()->currentAccessToken()->delete();... } }，在這個模型下是**完全可以接受且正確的**。

理由如下：這個操作從根本上講，是與當前 HTTP 請求的認證狀態（由 Sanctum 或 Passport 管理）綁定的 30。這是一個框架層面的關注點。

Request 物件正是這個狀態的容器。因此，Controller 是執行此操作的正確位置。試圖將其抽象到 Service 中，將需要笨拙地向下傳遞特定於請求的狀態，這違反了我們之前建立的純粹性原則。

這個觀點解決了「純淨架構」與「Laravel 之道」之間的虛假二分法。解決方案不是在兩者之間擇一，而是為它們定義一個清晰的邊界和一套轉譯規則。Controller 就是這個邊界。這讓您的團隊可以停止與框架對抗，並開始智慧地利用它。

### **第五節：精煉的架構藍圖（附快速參考矩陣）**

本節以清晰、規範的格式呈現最終推薦的架構，供您的團隊遵循。它綜合了前面所有章節的要點。

#### **5.1. 分層定義**

* **Controller**：處理 HTTP 請求和回應。作為框架與應用程式核心之間的中介。將 Request 資料轉譯為 DTO 或純量值。通過 Gates/Policies 處理授權。  
* **Action**：單一用途、可呼叫的類別，執行一個離散的業務操作。  
* **Service**：編排複雜的業務工作流程，通常會呼叫多個 Action 或 Repository。  
* **Repository**：I/O 閘道。管理所有與外部資料源（資料庫、S3、API 等）的通訊。負責將原始資料轉換為 DTO。  
* **DTO (laravel-data)**：用於在各層和模組之間進行資料傳輸的不可變 API 契約。是資料結構形態的唯一真實來源（single source of truth）。  
* **Eloquent Model**：資料庫表格的表示，**僅在 Repository 層內部使用**。可以包含關聯、作用域（scopes）和存取器/修改器，用於在轉換為 DTO 之前進行資料操作。

#### **5.2. 分層職責矩陣**

為了給您的團隊提供一個極具價值的、一目了然的指南，以下表格將所有規則固化為一個可執行的手冊。

**表 1：分層職責矩陣**

| 層級 | 主要職責 | 關鍵輸入 | 關鍵輸出 | 禁止的依賴 | 理由 |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Controller** | 中介 HTTP 與核心邏輯 | Request, Route 參數 | Response, View, JsonResource | Services, Repositories | 指定的邊界，用於與框架特定的部分（如 HTTP 和 Auth）互動 26。它將這些轉譯為核心層的純粹輸入。 |
| **Action/Service** | 執行業務邏輯 | DTOs, 純量值 | DTOs, 純量值, void, Events | Request, Session, Auth facade, Eloquent Models | 應用程式的核心。必須是純粹、可重用且獨立於傳遞機制的，以實現可維護性和可擴展性 19。 |
| **Repository** | 中介所有外部 I/O | 純量值, 搜尋條件 DTOs | DTOs (或 DTOs 集合) | Services, Actions | I/O 閘道。將應用程式核心與資料持久化和檢索的細節（無論是來自資料庫、S3 還是 API）隔離開來 15。 |
| **DTO** | 資料契約與傳輸 | 原始陣列, Request 資料, Eloquent Models | 不可變的 PHP 物件 | 業務邏輯, 持久化邏輯 | 模組與層級之間穩定、明確的 API 契約，對於在模組化單體中實現解耦至關重要 4。 |
| **Model** | 資料庫表格表示 | - | - | Repository 以外的任何層 | Eloquent 是一個強大的 ORM 32，但其 Active Record 的特性會產生耦合。將其限制在 Repository 層可以最大限度地發揮其功能，同時控制其「洩漏」的特性。 |

這個矩陣不僅僅是一個總結，它是一本可執行的規則手冊。對於一個在大型模組化專案上工作的團隊來說，擁有這樣一套清晰、簡潔且有理有據的規則，對於保持一致性至關重要。它在「這段程式碼應該放在哪裡？」這個問題被提出之前就給出了答案，從而減少了認知負擔，最大限度地減少了架構漂移，並使程式碼審查更加高效。

### **第六節：在實踐中解決過度設計：啟發式方法與權宜之計**

#### **6.1. 避免不必要樣板程式碼的實用指南**

本節提供了務實的「何時可以打破規則」的指導，這正是一個好的架構與一個教條式架構的區別所在。我們將引入一套啟發式方法，幫助您的團隊在不過度思考的情況下決定使用哪種模式。

#### **6.2. 架構決策啟發式方法**

以下決策表旨在指導開發人員的日常工作。

**表 2：架構決策啟發式方法**

| 如果任務是... | 推薦的模式是... | 理由與範例 |
| :---- | :---- | :---- |
| 一個簡單、單一且會修改狀態的業務操作。 | **Action 類別** | 高內聚，避免 Service 臃腫。例如：ActivateUserAction(int $userId) 22。 |
| 一個編排多個步驟或元件的複雜流程。 | **Service 類別** | 集中化複雜的工作流程。例如：OnboardingService，它創建使用者、設定個人資料並發送歡迎郵件 18。 |
| 一個簡單的、唯讀的查詢。 | **在 Controller 中直接使用 Eloquent**（僅限簡單情況） | 對於一個簡單的 Post::all()，創建多個層級是過度設計。這是針對瑣碎讀取操作的務實權宜之計 33。 |
| 一個複雜、可重用的資料庫**讀取**查詢。 | **Eloquent 區域作用域 (Local Scope) 或專門的查詢建構器 (Query Builder) 類別** | 利用框架的強大功能，避免為讀取操作編寫 Repository 樣板程式碼。例如：Post::published()->withPopularComments()->get()。結果可以在 Controller 中轉換為 DTO。 13。 |
| 任何**寫入**資料或與**非資料庫** I/O 源（S3、API）互動的操作。 | **Repository 方法** | 強制執行 I/O 閘道抽象，集中化所有狀態變更和外部通訊。這是一條硬性規則 15。 |

#### **6.3. 務實的「權宜之計」：何時可以直接使用 Eloquent**

我們在此明確指出，對於**不被重用**的**簡單唯讀**操作，可以直接在 Controller 中使用 Eloquent 查詢，並在那裡將結果轉換為 DTO 或 JsonResource。

例如：public function index() { $posts = Post::latest()->paginate(); return PostResource::collection($posts); }。這是一個經典的 Laravel 慣用手法。強迫這種簡單操作通過完整的 Repository/Service 堆疊，並不會帶來任何價值，反而會製造摩擦。關鍵在於，這僅適用於**唯讀**操作。所有**寫入**操作都必須通過 Repository/Action 層，以確保業務規則和 I/O 集中化得到遵守。這與在 33 和 34 中發現的務實觀點相符。

## **結論：實現一個可擴展、團隊友好且可維護的架構**

本報告提出的架構藍圖，旨在直接解決您在專案初期遇到的核心困境。它通過一套平衡且務實的原則，為您的團隊提供了一個堅實的基礎。

* **解決了大型模組化專案的結構問題**：通過將 DTO 作為模組間的契約，它為一個由多團隊協作的大型專案提供了必要的結構和解耦，確保了長期的可維護性 1。  
* **避免了過度設計**：通過為簡單任務引入 Action 類別，並為簡單的讀取操作提供務實的權宜之計，它避免了不必要的樣板程式碼和架構的僵化 22。  
* **化解了與框架慣用手法的衝突**：通過明確定義 Controller 作為框架邊界的中介者，它允許您在適當的地方（Controller）充分利用 Laravel 的便利功能（如 $request->user()），而無需犧牲應用程式核心的純粹性 28。

最終，這套架構賦予您的團隊一套清晰的指導方針。它既在其核心層面保持了「純淨」，又在其應用層面體現了「務實」。這是一個能夠讓您和您的團隊在所選框架的基礎上高效建構，而非與之對抗的、可擴展且團隊友好的基礎。

#### **引用的著作**

1. Laravel Modular Architecture: Breaking the Monolith - MuneebDev, 檢索日期：7月 17, 2025， [https://muneebdev.com/laravel-modular-architecture-breaking-the-monolith/](https://muneebdev.com/laravel-modular-architecture-breaking-the-monolith/)  
2. The Laravel Modular Architecture Implementation in Brief - Mohammed Muwanga - Medium, 檢索日期：7月 17, 2025， [https://muwangaxyz.medium.com/the-laravel-modular-architecture-implementation-in-brief-51951a9e9e98](https://muwangaxyz.medium.com/the-laravel-modular-architecture-implementation-in-brief-51951a9e9e98)  
3. Is this too much for a modular monolith system? - Software Engineering Stack Exchange, 檢索日期：7月 17, 2025， [https://softwareengineering.stackexchange.com/questions/457147/is-this-too-much-for-a-modular-monolith-system](https://softwareengineering.stackexchange.com/questions/457147/is-this-too-much-for-a-modular-monolith-system)  
4. DTO (Data Transfer Objects) in PHP (Laravel) | by Mohammad Roshandelpoor | Medium, 檢索日期：7月 17, 2025， [https://medium.com/@mohammad.roshandelpoor/dto-data-transfer-objects-in-laravel-6b391e1c2c29](https://medium.com/@mohammad.roshandelpoor/dto-data-transfer-objects-in-laravel-6b391e1c2c29)  
5. What is the advantage of DTO (over model instances)? : r/laravel - Reddit, 檢索日期：7月 17, 2025， [https://www.reddit.com/r/laravel/comments/1arq55e/what_is_the_advantage_of_dto_over_model_instances/](https://www.reddit.com/r/laravel/comments/1arq55e/what_is_the_advantage_of_dto_over_model_instances/)  
6. java - REST API - DTOs or not? - Stack Overflow, 檢索日期：7月 17, 2025， [https://stackoverflow.com/questions/36174516/rest-api-dtos-or-not](https://stackoverflow.com/questions/36174516/rest-api-dtos-or-not)  
7. Web API DTO Considerations | Blog, 檢索日期：7月 17, 2025， [https://ardalis.com/web-api-dto-considerations/](https://ardalis.com/web-api-dto-considerations/)  
8. Data Transfer Objects: The Contractual Lowdown on DTOs - Gable.ai, 檢索日期：7月 17, 2025， [https://www.gable.ai/blog/data-transfer-objects](https://www.gable.ai/blog/data-transfer-objects)  
9. API Resources: With or Without "data"? - Laravel Daily, 檢索日期：7月 17, 2025， [https://laraveldaily.com/tip/api-resources-with-or-without-data](https://laraveldaily.com/tip/api-resources-with-or-without-data)  
10. DTO vs Resource in Laravel: What's the Difference and When to Use Each - Medium, 檢索日期：7月 17, 2025， [https://medium.com/@prevailexcellent/dto-vs-resource-in-laravel-whats-the-difference-and-when-to-use-each-d9d5270dd1b9](https://medium.com/@prevailexcellent/dto-vs-resource-in-laravel-whats-the-difference-and-when-to-use-each-d9d5270dd1b9)  
11. Eloquent: API Resources - Laravel 12.x - The PHP Framework For Web Artisans, 檢索日期：7月 17, 2025， [https://laravel.com/docs/12.x/eloquent-resources](https://laravel.com/docs/12.x/eloquent-resources)  
12. 大部分人的仓库模式都用错了吗？—— laravel-腾讯云开发者社区 ..., 檢索日期：7月 17, 2025， [https://cloud.tencent.com/developer/article/1491737](https://cloud.tencent.com/developer/article/1491737)  
13. To "Repository Pattern" or Not? - Laracasts, 檢索日期：7月 17, 2025， [https://laracasts.com/discuss/channels/general-discussion/to-repository-pattern-or-not](https://laracasts.com/discuss/channels/general-discussion/to-repository-pattern-or-not)  
14. Is using the repository pattern best practise? - laravel - Reddit, 檢索日期：7月 17, 2025， [https://www.reddit.com/r/laravel/comments/10o5s2l/is_using_the_repository_pattern_best_practise/](https://www.reddit.com/r/laravel/comments/10o5s2l/is_using_the_repository_pattern_best_practise/)  
15. Repository Design Pattern Demystified - SitePoint, 檢索日期：7月 17, 2025， [https://www.sitepoint.com/repository-design-pattern-demystified/](https://www.sitepoint.com/repository-design-pattern-demystified/)  
16. How to Integrate multiple external data sources in Laravel with DTOs - Lucky Media, 檢索日期：7月 17, 2025， [https://www.luckymedia.dev/blog/how-to-integrate-multiple-external-data-sources-in-laravel-with-dtos](https://www.luckymedia.dev/blog/how-to-integrate-multiple-external-data-sources-in-laravel-with-dtos)  
17. how to structure solution to allow for multiple data sources for a repository - Stack Overflow, 檢索日期：7月 17, 2025， [https://stackoverflow.com/questions/42751146/how-to-structure-solution-to-allow-for-multiple-data-sources-for-a-repository](https://stackoverflow.com/questions/42751146/how-to-structure-solution-to-allow-for-multiple-data-sources-for-a-repository)  
18. Service Layer Laravel Tutorial | Examples & Best Practices for Laravel 11 - MuneebDev, 檢索日期：7月 17, 2025， [https://muneebdev.com/service-layer-laravel-tutorial/](https://muneebdev.com/service-layer-laravel-tutorial/)  
19. Understanding Laravel Service Classes: A Comprehensive Guide - Medium, 檢索日期：7月 17, 2025， [https://medium.com/@laravelprotips/understanding-laravel-service-classes-a-comprehensive-guide-1f22310c70bd](https://medium.com/@laravelprotips/understanding-laravel-service-classes-a-comprehensive-guide-1f22310c70bd)  
20. Services VS Actions in Laravel #laravel - YouTube, 檢索日期：7月 17, 2025， [https://www.youtube.com/shorts/Mq7n5q_lcdc](https://www.youtube.com/shorts/Mq7n5q_lcdc)  
21. laravel actions (or service classes) - Reddit, 檢索日期：7月 17, 2025， [https://www.reddit.com/r/laravel/comments/l2ab5u/laravel_actions_or_service_classes/](https://www.reddit.com/r/laravel/comments/l2ab5u/laravel_actions_or_service_classes/)  
22. Save Data: Service or Action? - Laravel Daily, 檢索日期：7月 17, 2025， [https://laraveldaily.com/lesson/laravel-projects-structure/save-data-service-action](https://laraveldaily.com/lesson/laravel-projects-structure/save-data-service-action)  
23. Laravel AaaS - Actions as a Service - Wendell Adriel, 檢索日期：7月 17, 2025， [https://wendelladriel.com/blog/laravel-aaas-actions-as-a-service](https://wendelladriel.com/blog/laravel-aaas-actions-as-a-service)  
24. Laravel Services Or Actions or None - Laracasts, 檢索日期：7月 17, 2025， [https://laracasts.com/discuss/channels/laravel/laravel-services-or-actions-or-none](https://laracasts.com/discuss/channels/laravel/laravel-services-or-actions-or-none)  
25. Clean Code Architecture in Laravel: A Practical Guide - DEV Community, 檢索日期：7月 17, 2025， [https://dev.to/arafatweb/clean-code-architecture-in-laravel-a-practical-guide-ho2](https://dev.to/arafatweb/clean-code-architecture-in-laravel-a-practical-guide-ho2)  
26. Laravel — Discover Application Layers for Testing | by Dmitry Khorev - Level Up Coding, 檢索日期：7月 17, 2025， [https://levelup.gitconnected.com/laravel-discover-application-layers-for-testing-42c9f35ddea6](https://levelup.gitconnected.com/laravel-discover-application-layers-for-testing-42c9f35ddea6)  
27. Everyone who has done MVC long enough would also know that the controller is the... | Hacker News, 檢索日期：7月 17, 2025， [https://news.ycombinator.com/item?id=22358072](https://news.ycombinator.com/item?id=22358072)  
28. Authorization - Laravel 12.x - The PHP Framework For Web Artisans, 檢索日期：7月 17, 2025， [https://laravel.com/docs/12.x/authorization](https://laravel.com/docs/12.x/authorization)  
29. Laravel Authentication: Getting Started and HTTP Auth Tutorial - Frontegg, 檢索日期：7月 17, 2025， [https://frontegg.com/blog/laravel-authentication](https://frontegg.com/blog/laravel-authentication)  
30. Authentication - Laravel 12.x - The PHP Framework For Web Artisans, 檢索日期：7月 17, 2025， [https://laravel.com/docs/12.x/authentication](https://laravel.com/docs/12.x/authentication)  
31. The developer guide to Laravel authentication - Honeybadger.io, 檢索日期：7月 17, 2025， [https://www.honeybadger.io/blog/laravel-authentication/](https://www.honeybadger.io/blog/laravel-authentication/)  
32. Eloquent: Getting Started - Laravel 12.x - The PHP Framework For Web Artisans, 檢索日期：7月 17, 2025， [https://laravel.com/docs/12.x/eloquent](https://laravel.com/docs/12.x/eloquent)  
33. Where to place the queries? - Laracasts, 檢索日期：7月 17, 2025， [https://laracasts.com/discuss/channels/laravel/where-to-place-the-queries](https://laracasts.com/discuss/channels/laravel/where-to-place-the-queries)  
34. Proper way to use Laravel Eloquent | by Mantas Donelavicius - Medium, 檢索日期：7月 17, 2025， [https://medium.com/@mantasd/proper-way-to-use-laravel-eloquent-3ca194e2b766](https://medium.com/@mantasd/proper-way-to-use-laravel-eloquent-3ca194e2b766)