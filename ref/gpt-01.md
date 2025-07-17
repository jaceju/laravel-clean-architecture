# Laravel應用程式架構分析

## 引入 Data Object (DTO) 的必要性

**Data Object（資料傳輸物件，DTO）**是一種將資料結構與資料庫模型解耦的方式。在 Laravel 中使用 DTO（例如透過 Spatie 的 **laravel-data** 套件）有其利弊，需要視專案情況決定其必要性：

* **清晰的資料契約與模組解耦：**在大型或多模組的專案中，引入 DTO 可以讓各層之間的資料傳遞更明確，降低耦合度。DTO 可以作為**模組與模組之間**交換資料的契約，避免直接依賴 Eloquent 模型。即使沒有嚴格實行 DDD，DTO 也能在整個應用內提供統一的資料介面。例如，當有多個支付服務提供商時，我們可以發送 DTO 給不同的整合服務並接收 DTO，確保不同實現都返回相同格式的資料。這對於團隊協作尤其有利 —— 大家對傳遞的資料結構有共識，不容易產生誤解。

* **資料結構的靈活性與安全性：**DTO **不必與資料庫模型一一對應**。它可以從多個模型彙總資料，或選擇模型的部分欄位，甚至加入額外的派生欄位或狀態標記，以符合業務需求。這讓我們在不改動資料庫結構的情況下，調整對外提供的資料格式。例如，可以透過 DTO 隱藏模型中敏感或前端不需要的欄位，僅暴露必要資訊（laravel-data 支援欄位 lazy 加載或 optional 選擇）。這種封裝提高了**資料傳輸的安全性**與穩定性，前端不會意外依賴到不該曝光的欄位。

* **統一請求驗證與回應格式：**使用 Spatie 的 **Laravel Data** 套件，可以讓同一份 DTO 定義同時用於**請求資料的驗證**以及**回應資料的序列化**。也就是說，一個資料類別可以取代 Form Request（表單請求驗證）和 API Resource（資源序列化）兩種角色，從而保證輸入輸出遵循相同的資料結構契約。這其實**簡化**了應用程式結構，而非增加複雜度：只需在 DTO 類中定義欄位和轉換邏輯，就能同時處理好輸入與輸出，減少重複的代碼和維護成本。

* **潛在的未來擴充彈性：**如果未來可能更換資料來源（例如從直接存取資料庫換成呼叫外部 API），使用 DTO 會非常有幫助。因為 Service 層和 Controller **不依賴 Eloquent 模型實例**，只需要 Repository 層將不同來源的資料轉成 DTO 給上層即可。換言之，DTO 在架構上提供了一個抽象層，使我們可以「較容易地替換某些東西的實現，而不會因為對模型的依賴牽一髮動全身」。這對大型團隊協作也是利多，因為不同小組可以各自處理資料來源的細節，透過約定的 DTO 介面進行整合。

**然而，引入 DTO 也有相應的成本與考量：**

* **增加樣板代碼與維護負擔：** DTO 通常需要撰寫大量樣板程式碼來定義屬性、轉換方法（如 `fromModel`）等。如果 DTO 大部分欄位其實和模型一樣，那麼這些重複定義可能被視為樣板程式碼。正如社群討論所指出的：「使用 DTO 確實更符合單一職責、程式碼更乾淨的原則，但你得寫許多樣板程式碼；對於簡單的 CRUD 應用，這可能並不划算」。一旦模型欄位變更，你得同步修改 DTO 和轉換邏輯，增加維護成本。在開發速度與程式碼架構優雅之間需要權衡。

* **Laravel 既有功能的流失：**當我們決定完全以 DTO 作為上下層溝通的資料物件，就意味著**上層不直接接觸 Eloquent 模型**。這雖然達到解耦，但也失去了一些 Laravel 提供的便利功能。例如，Eloquent 模型的延遲載入、關聯處理、資料庫查詢快捷方法等，在 DTO 中無法使用（因為 DTO 通常只是純資料，沒有ORM行為）。正如有開發者提到的，如果在 Repository 中**返回自訂的 DTO 物件**且上層不再使用 Eloquent，「會讓你無法使用 Laravel 中許多重要的功能」。團隊需要自行在 Repository 層處理關聯資料的載入或建立更多方法，以彌補直接使用模型時的那些方便之處。

* **並非所有專案都需要：**在很多情況下，直接使用 Eloquent 模型就能滿足需求，因此額外引入 DTO 可能被視為**過度設計**。一句話總結就是：「大多數情況下，你不需要 DTO。如果你不確定是否需要，那大概就是不需要——因為 DTO 很容易讓你走向過度設計」。通常只有在**需要在不同系統/服務間傳遞資料**時（DTO的原意即如此）才會用到它；比方說，要把模型部分欄位發給外部 API，或反過來把 API 回傳資料封裝成物件時，DTO 很有用。但如果你的應用主要是在同一個 Laravel 專案內部傳遞資料，沒有跨系統的情境，那麼直接傳遞模型實例或使用 Laravel Resource 其實已經足夠，而且更簡潔。

概括來說，**在大型多模組專案中**，若你需要嚴格的分層解耦、考慮長期的可擴充性或者需要跟多種外部系統對接，引入 DTO 能帶來清晰的結構和穩定的資料契約，對團隊協作和代碼健全性都有幫助。反之，**如果專案規模中小且邏輯簡單、團隊主要關注點在於快速交付功能**，那麼直接使用模型可能更實際。特別是在目前大部分測試都是透過 Laravel 的整體 HTTP 測試（集成測試）而非對各層單元測試的情況下，DTO 帶來的好處會比較難以體現。重要的是避免為了模式而模式，當發現 DTO 的方法只是單純把模型欄位搬運過去，或團隊成員經常繞過 DTO 直接用模型（就像你舉的 `$request->user()->currentAccessToken()->delete()` 例子），那就說明 DTO 可能並非嚴格必要。**總結**：評估DTO必要性的關鍵在於**需求複雜度**與**抽象收益**。如果它能實際解決你的痛點，那就是必要且值得的；若只是理論上「感覺架構更漂亮」，實際開發中反而帶來更多負擔，那就是過度設計。

## Repository 層會不會造成過度設計？

**Repository 模式**的目的在於抽象資料存取邏輯，提供一個介面給業務層，不用關心底層使用何種資料庫或來源。理論上，Repository 層可以讓我們更輕鬆地替換資料來源（例如從 MySQL 切換到 PostgreSQL，甚至改成讀取 API 或其他存儲）而不用修改業務程式碼，同時也便於在單元測試中以模擬（mock）的 repository 取代真實資料庫。然而，在 Laravel 生態中使用 Repository 是否屬於**過度設計**，一直有熱烈討論，需要結合實際情況來判斷：

* **Laravel 模型已經提供類似功能：**Laravel 的 Eloquent 模型本身就是 Active Record 模式，封裝了資料庫操作並提供豐富的查詢接口。在許多場合下，Eloquent **已經扮演了「資料存取層」的角色**。有經驗的開發者指出，在 Laravel 裡「模型幾乎就像一個 repository」；換言之，你可以直接在模型上調用 `User::all()`, `User::find($id)` 等方法完成CRUD，而這正是Repository常見要做的事。如果我們再建立一個 Repository 來呼叫這些模型方法，其實是對 Eloquent 的二次封裝，往往並沒有增加新的抽象好處。Laravel **官方中文文檔**甚至直接建議：“**絕不使用 Repository**，因為我們不是在寫 Java 代碼，太多封裝就成了『過度設計』，極大降低了編碼愉悅感，使用 MVC 夠傻夠簡單”。從這可以看出，在 Laravel 環境下過度引入 Repository 層可能徒增複雜。

* **無謂的包裝導致重複代碼：**很多人在 Laravel 中實現 Repository 時，會出現「為實現而實現」的情況：方法內部只是簡單呼叫對應的模型操作，例如 `return $this->model->find($id)` 然後包裝成 `getById($id)`。這種做法其實並沒有真正抽象出更高層次的意義，只是把 Eloquent 的用法轉移到另一個類，導致**樣板代碼**激增。正如有人批評的，如果 Repository 只是把 Eloquent 查詢「包一層」，那麼「一開始就建立倉庫是沒有意義的，它只是 Eloquent 查詢的抽象，而 ORM 的抽象並不等於真正的倉庫模式」。這不僅沒降低耦合，反而可能因為多了一層呼叫關係讓代碼更難追蹤。尤其當 Repository **直接返回 Eloquent 模型**時，上層業務其實還是與 Eloquent 綁定，解耦目的根本沒達成。此時 Repository 層只是徒增檔案與心智負擔，屬於典型的過度設計。

* **完全抽象資料層的代價：**也有開發者嚴格遵循 Repository 模式的理念，讓 Repository **不返回 Eloquent 模型**而是返回自訂的資料物件（例如前述的 DTO）。這種實作確實達到了與 ORM 的解耦，但相對地，**上層無法享受 Eloquent 帶來的便利**。例如，你無法在 Service 裡調用 `$model->relations->load()` 或使用集合(`Collection`)的方法去處理資料，因為拿到的已經是純資料物件而非模型實例。除此之外，如果 Laravel 特有的功能（事件、全域作用域、模型觀察者等）在 Repository 層以下觸發，上層也難以感知或控制。換言之，為了達到資料來源可切換，你勢必要實作一套自己的資料操作邏輯來替代 Laravel 原生的一些特性。這對團隊而言是額外的學習和維護成本。如果你的專案未來**並不真的會切換ORM或資料來源**，那這份代價就白白浪費了。

* **對測試的影響：**從測試角度看，Repository 模式的好處是可以透過介面將資料存取與業務邏輯隔離，以方便替換假的資料來源來進行**單元測試**。但在你的情境下，大部分測試是整合性的 HTTP 測試，而不是對 Service 或 Repository 進行獨立單元測試。這意味著你們並沒有利用到 Repository 模式帶來的可測試性優勢；相反，測試仍然會經過真實的資料庫（或是Memory DB）來運行 Eloquent。因此，為了“可測試”而加的 Repository 對你們實際並沒有幫助（這點從你們沒有撰寫 Repository 的 mock 測試即可看出）。如果一個架構的複雜性提高了，但預期收益（如可測試性）沒有兌現，那麼這個架構就是在降低開發效率，屬於不必要的設計。

**總的來說**：在**大型團隊協作專案**中，保持清晰的分層是好的，但必須確保每一層**確實提供了額外價值**。Service 層通常是有價值的，它承擔了控制器與資料層之間的業務邏輯，讓控制器更簡潔、邏輯更聚焦。你提到有時 Service 方法只是呼叫 Repository 方法，例如 `MembershipService.updateMemberActiveToken()` 只是轉呼 `MemberRepository.updateMemberActiveToken()`。這種情況短期看似乎多此一舉，但可以從兩方面評估：其一，如果這段邏輯未來可能增強（比如更新 token 後還要發事件、寫 log 等），那現在保留 Service 作為擴充點是合理的；其二，如果確定這純粹是簡單委派且不會演變出更多邏輯，那完全可以**直接在其他層呼叫 Repository**，省略中間的空轉。畢竟架構是為瞭解決問題而存在，不是硬性約束。Laravel 社群普遍的看法是：**能不用 Repository 就盡量不用**，避免為模式而模式。如果覺得模型查詢散落各處難以管理，可以考慮使用 **Local Scope/Global Scope** 或在模型中封裝常用查詢邏輯，或者採用**Domain Service**在需要的地方組合模型操作，而不是機械地套一層 Repository。

在你的描述中，已經出現開發人員**繞過 Repository/Service 直接使用模型**的情形（例如在 Controller 直接呼叫 `$request->user()->currentAccessToken()->delete()` 進行登出）。這反映出當前的分層規範可能不完全符合實際開發便利性。與其強制所有操作都走 Repository，不如**區分情境**：**CRUD 等簡單操作**直接用模型或 Query Builder 未嘗不可；**複雜的業務操作**（涉及多個模型或外部服務的流程）才封裝到 Service，必要時在 Service 中調用多個 Repository 或其他服務。Repository 層則可以**有選擇地實施**：針對「資料來源多樣化或查詢複雜度高」的部分提供抽象。例如，檔案存取、第三方 API 資料，就很適合作為 Repository 方法；而單純對單一模型的簡單查詢，使用 Eloquent 本身就能做好。這樣，你的架構既保持了靈活性，又不會讓每一層都淪為走過場。

最後，**過度設計與適度設計的平衡**需要不斷根據專案規模和團隊需求來調整。Laravel 的哲學一直強調**實用至上**：如果某個模式在你的開發中沒有明顯收益，那就沒有引入的必要。以你的情況，大型多模組系統確實需要考慮長遠維護，但也要注意過多抽象帶來的學習成本和心智負擔。你可以和團隊討論並制定統一的方針，例如：**界定何種情況下一定要用 Repository/DTO，何種情況下可以直接使用模型**。只要團隊達成共識並遵守約定，其實不論是否使用 Repository/DTO，都能寫出易於維護的程式碼。記住，架構模式是服務於業務需求的，當發現**「繁文縟節」超過了實際好處**時，就應該考慮簡化。合理取捨後，你的Laravel專案將同時兼具**結構性**與**開發效率**，避免落入過度設計的陷阱。

**參考資料：**

1. Laravel 中 Repository 模式與過度設計的討論
2. 引入 DTO 的利弊與社群觀點
3. Spatie Laravel Data 套件在請求/回應中的應用
4. Laravel 大型專案架構經驗分享


你提到兩個核心問題：

1. **Controller 直接使用 Model 是否合理？有無更好的做法？**
2. **Repository 常成為過度設計，有沒有更實用的替代架構？**

這些其實都是在追求 **「實用解耦」** 的最佳平衡點，以下我會條列幾種**替代思維與架構模式**，協助你在大型 Laravel 專案中找到更務實的做法：

---

## 🔹 一、Controller 可以直接使用 Model 嗎？

### ✅ **何時可以接受 Controller 直接使用 Model？**

* **Request 綁定的關聯 Model（如 `$request->user()`）**

  * 這類屬於 **框架已注入的資料來源**，邏輯簡單、使用頻率高，沒必要再抽一層。
  * 若你用 `UserData::from($request->user())` 包裝，只是為了傳遞 `name`, `email` 到前端，這其實是**過度形式化**。
  * ✅ 建議方式：直接用 `$request->user()`，配合 Laravel Resource（或 `laravel-data` output only DTO）即可。

### ✅ **單純資料操作，不涉及業務邏輯時**

* CRUD 表單儲存、更新時，如果只是存 Model，不涉及流程控制或其他系統互動。
* ✅ 建議方式：

  * Controller → Model（可包一個 Form Request）
  * 不需要經過 Service / Repository

---

## 🔹 二、Repository 模式的替代方案

### 🟡 1. **Query Object 模式（專職查詢）**

* 與 Repository 相比，**Query Object 更單一職責**：專門封裝查詢條件，不處理儲存或刪除等副作用。

* 📌 適合用來處理複雜篩選、關聯查詢或排序，例如：

  ```php
  class ActiveMemberQuery
  {
      public static function get(): Builder
      {
          return Member::where('active', true)->with('group')->orderByDesc('last_login_at');
      }
  }
  ```

* ✅ 優點：

  * 可重用
  * 沒有額外物件抽象
  * 與 Eloquent 相容性佳

* 🚫 缺點：

  * 無法完全抽象來源（例如改用 API 取得就無法用了）

---

### 🟡 2. **Action / Job / UseCase 類別（取代 Service + Repository 的組合）**

* 每個商業邏輯包成「一個操作」類別：

  * 如：`UpdateMemberActiveTokenAction`, `LogoutUserAction`

* 這類行為通常**只對應一件事**，可透過注入的依賴來處理資料。

* 📌 Laravel 中你也可以直接用 `Job` 來實作這種 use case。

  ```php
  class LogoutUserAction
  {
      public function __invoke(User $user): void
      {
          $user->currentAccessToken()->delete();
      }
  }
  ```

* Controller 中直接呼叫：

  ```php
  public function logout(Request $request, LogoutUserAction $logout)
  {
      $logout($request->user());
      return response()->json(['message' => '已登出']);
  }
  ```

* ✅ 優點：

  * 無需抽 Service / Repository 雙層
  * 易測試（單一行為，容易寫 test）
  * 可用 Laravel 的 `dispatch()` 排程執行（同步/非同步皆可）

---

### 🟢 3. **Domain Service 模式（只用於跨模型邏輯）**

* 將多個模型互動、複雜的業務邏輯抽成 Domain Service
* 📌 不處理簡單 CRUD
* 📌 不處理查詢語意（Query 可分離）

例如：

```php
class ActivateSubscriptionService
{
    public function handle(Member $member, Plan $plan): void
    {
        DB::transaction(function () use ($member, $plan) {
            $member->subscribe($plan);
            $member->recordAction('subscription_activated');
        });
    }
}
```

---

## 🔸 統整建議

| 情境                                     | 建議做法                            | 是否需要 DTO         | 是否需要 Repository         |
| -------------------------------------- | ------------------------------- | ---------------- | ----------------------- |
| Controller 使用 `$request->user()` 等常見注入 | ✅ 直接用                           | ❌ 不需             | ❌ 不需                    |
| 單純 CRUD 儲存或查詢                          | ✅ Controller → Model 或 Resource | 🔄 可用 DTO 作為回傳格式 | ❌ 不需                    |
| 複雜查詢（排序、條件篩選）                          | ✅ Query Object                  | ❌                | ❌                       |
| 複雜邏輯（跨模型、多步驟）                          | ✅ Domain Service 或 Action Class | ✅ 輸出可用 DTO       | ✅ 可用自訂 Repository（但視情況） |
| 多資料來源（DB / API）                        | ✅ Repository 層封裝來源              | ✅ 必須解耦 Model     | ✅ 推薦使用                  |

---

## ✳️ 實用原則（供團隊制定 coding style）

1. **小操作 → 用 Model**

   * 像 `$user->currentAccessToken()->delete()` 屬於自然語意操作，避免包一層增加心智負擔

2. **複雜操作 → 用 Action**

   * 用 `XXAction::handle()` 來取代複雜 Service / Repository 組合

3. **可選 DTO，用於傳遞資料或定義資料契約**

   * 輸入用 FormRequest，輸出用 Laravel Resource 或 Spatie Laravel-Data

4. **Repository 僅用於多來源切換或邏輯復用**

   * 例如同一組查詢可能需要支援「從 DB 查」與「從 API 查」，這時才值得包裝

---

如果你想讓團隊更容易統一風格，我可以幫你整理一份簡單的「**Laravel 開發分層準則備忘錄**」或是「樣板結構模板」，你需要的話我可以補上。
