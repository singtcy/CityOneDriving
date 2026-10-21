# HiAce H200 駕駛模擬 — 已驗證步驟

Unity 專案：`C:\Users\Sing\Desktop\AY2526_CityOne`  
Unity **2022.3.62f3**，內建渲染管線。車輛根物件名稱是 `ToyotaH200`。FBX 仍在 `Assets/2012 Toyota HiAce Commuter/`，不要改這個資料夾名稱。

車身規格見 `toyota_hiace_h200_at_unity.md`。下面只記從安裝 Input System 到目前可駕駛、玻璃透明為止，實際有效的做法。

## 1. Input System

- 安裝 `com.unity.inputsystem` **1.14.0**。
- `ProjectSettings/ProjectSettings.asset` 的 `activeInputHandler` 設為 **2**（Both）。場景裡有 EasyRoads3D，只開新 Input System 會關掉舊輸入。
- 改完 Active Input Handling 後要重開 Unity，新後端才會生效。

NWH 用的就是這套 Unity Input System，不是另一個輸入插件。

## 2. 輸入物件

場景裡放一個 `VehicleInput`，只加這兩個元件：

- `InputSystemVehicleInputProvider`，**Mouse Input 關閉**。
- `InputSystemSceneInputProvider`。

每種 Provider 只留一個。

`InputSystemVehicleInputProvider` 要保留這些修正：

- `Mouse.current` 為空時不要讀滑鼠。否則 `Update` 每幀丟 `NullReferenceException`，油門會一直是 0。
- `Keyboard.current` 為空時，用舊輸入讀 **W / S / A / D / Space**。
- `ShiftInto()` 直接用 `KeyCode.R`、`KeyCode.N`、`KeyCode.G`。只靠 Input System 的 `ShiftIntoR1` 回呼時，R 不會進變速箱，畫面上的入檔要求會停在 -999。

## 3. 車輛

用 NWH `VehicleSetupWizard.RunSetup`，車輪類型選 **UnityWheelCollider**。不要加 NWH 預設追蹤攝影機，也不要加入出場點。

先在 Tag Manager 加上 `Vehicle` 和 `Wheel`。車輛上至少要有一個碰撞體，wizard 才會繼續。車身用一個 `BoxCollider`，不要給每塊車殼加 MeshCollider。

四個輪胎：


| 位置      | 物件                |
| ------- | ----------------- |
| 右前（駕駛側） | `TireRing173`     |
| 左前      | `TireRing174`     |
| 右後      | `TireRing173.001` |
| 左後      | `TireRing174.001` |


`WheelCollider` 半徑 **0.40**，寬 **0.32**，`suspensionDistance` **0.5**。`Rims9`、`Rims11` 不能當碰撞體。

設定：

- `Rigidbody.mass = 1900`
- 轉向最大角 **37.8°**
- 變速箱 **Automatic**
- **D/N/R Shift Type = Require Shift Input**
- 離合器 **Automatic**
- `swapInputInReverse = false`。倒檔一樣是 **W** 後退、**S** 煞車。
- `flyingStartEnabled = true`
- 後輪驅動：變速箱輸出到中央差速器，`biasAB = 1`，再接到後差速器和兩個後輪。左右輪用 **local X** 判斷。這台車世界座標 X 約 -410，用世界座標 X 會把四個輪都當成同一側。

`Assets/HiAceStartInDrive.cs` 在 `Start` 設定出力與齒比，在 `LateUpdate` 鎖檔。不要在 `Start` 呼叫 `transmission.ShiftInto`。那時 `vehicleController.input` 還沒建立，會在 `TransmissionComponent.ShiftInto` 丟空參考，而且 NWH 初始化會把檔位清回 N。

齒比與出力（已能正常向前加速）：

- `maxPower = 110`
- 動力曲線在低轉就有力：`(0, 0.45)`、`(0.25, 0.95)`、`(0.6, 1)`、`(1, 0.8)`
- `finalGearRatio = 4.2`
- 檔位比：`-2.4, 0, 2.5, 1.6, 1.15, 0.85`（倒檔、空檔、1 至 4 檔）
- 升檔 2100 rpm，降檔 1100 rpm
- 離合器 `slipTorque = 2200`，`engagementRPM = 850`
- 每個 `WheelCollider.mass = 35`

檔位閘門：

- 預設前進。空檔時油門大於 0.1，掛 1 檔。
- **R** 鎖倒檔，之後放開 R 仍保持 R。
- **N** 鎖空檔。
- **G** 鎖前進。不要用 **D** 當前進檔，D 已是右轉。
- 已在 2、3、4 檔時，不要每幀強制改回 1 檔。

## 4. 操作

先點 **Game** 視窗。


| 按鍵    | 作用                     |
| ----- | ---------------------- |
| W     | 向前。在 N 時會入 1 檔。在 R 時向後 |
| S     | 煞車                     |
| A / D | 左右轉向                   |
| Space | 手煞車                    |
| R     | 倒檔，按一次即鎖住              |
| N     | 空檔                     |
| G     | 前進檔                    |
| E     | 熄火或再發動                 |
| Z / X | 左右方向燈                  |
| J     | 危險警告燈                  |
| L / K | 近光 / 遠光                |


升檔時的短卡聲是換檔切動力。引擎在空檔轉到約 4400 rpm 的連續卡聲是斷油，代表檔位仍是 N，動力沒有進變速箱。

除錯面板是 `Assets/HiAceDebugHud.cs`，顯示在畫面右上角。檔位應離開 N，變速箱輸出不應長期為 0。

## 5. 玻璃

材質：`Assets/HiAceGlass.mat`。

- Shader：Standard
- Rendering Mode：Transparent（`_Mode = 3`，`_ALPHAPREMULTIPLY_ON`）
- 顏色：RGB `(0.12, 0.16, 0.14)`，alpha **0.35**
- Smoothness **0.9**，Metallic **0**
- Render Queue **3000**
- 玻璃 Renderer 關閉 Cast Shadows 和 Receive Shadows
- 不加 Collider

套用物件，車門玻璃留在原來的門下面：

`Meshpart10`、`Meshpart71`、`Meshpart32`、`boot_ok.002`、`boot_ok.003`、`Meshpart86.001`、`Meshpart84.004`、`door_lr_sliding`、`door_lr_sliding.001`

這份材質已套在 `scene2` 的 `ToyotaH200`。Standard 是單面，車內若看不到玻璃，再改雙面。

## 6. 攝影機

`Main Camera` 父級是 `ToyotaH200`。車頭方向是車輛 local **+Z**。目前畫面用的是這個相對位置，單位米：

- **Local Position**：`(0.03, 1.99, 1.44)`
- **Local Rotation**：`(0, 0, 0)`
- **Field of View**：60

攝影機在車身中線附近、離地約 1.99 m、車頭方向 1.44 m。車輛移動時世界座標會跟著變，不要改回世界座標。