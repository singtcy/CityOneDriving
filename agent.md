# HiAce H200 駕駛模擬 — 已驗證步驟

Unity 專案：`C:\Users\Sing\Desktop\AY2526_CityOne`  
Unity **2022.3.62f3**，內建渲染管線。車輛根物件名稱是 `ToyotaH200`。FBX 仍在 `Assets/2012 Toyota HiAce Commuter/`，不要改這個資料夾名稱。

車身規格見 `toyota_hiace_h200_at_unity.md`。下面只記從安裝 Input System 到目前可駕駛、玻璃透明、三面後視鏡可用為止，實際有效的做法。

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

`Assets/HiAceStartInDrive.cs` 在 `Start` 設定出力、齒比和離合器，在 `inputModifyCallback` 鎖 D/N/R，在 `LateUpdate` 入檔。不要在 `Start` 呼叫 `transmission.ShiftInto`。那時 `vehicleController.input` 還沒建立，會在 `TransmissionComponent.ShiftInto` 丟空參考，而且 NWH 初始化會把檔位清回 N。

引擎與齒比：

- `maxPower = 110` kW，怠速 **800**，斷油 **4200** rpm
- 動力曲線：`(0, 0.45)`、`(0.25, 0.95)`、`(0.6, 1)`、`(1, 0.8)`
- `finalGearRatio = 4.2`
- 檔位比：`-2.4, 0, 2.5, 1.6, 1.15, 0.85`（倒檔、空檔、1 至 4 檔）
- 升檔 **1800** rpm，降檔 **1100** rpm
- `variableShiftPoint = false`。升檔用車速反推的不打滑轉速，不看引擎實際轉速。全油門若再開可變升檔點，1 檔會停在約 30 km/h 升不上去。1800 rpm 大約是 1 檔 **26 km/h**。
- 每個 `WheelCollider.mass = 35`

離合器保持 **Automatic**，駕駛員不用踩。NWH 的動力順序是引擎、離合器、變速箱，自動檔沒有另做扭力轉換器。

- `engagementRPM = 850`
- 設好動力曲線後呼叫 `UpdatePeakPowerAndTorque()`，`slipTorque = EstimatedPeakTorque * 1.5`。目前峰值 **1572 Nm**（含增壓係數 1.4），所以限額約 **2358 Nm**。

檔位閘門在 `HiAceStartInDrive`。速度用剛體速度投影到車頭 **+Z**，單位 m/s。停穩門檻 `StopSpeed = 0.5`（約 **1.8 km/h**）。

- 預設前進。空檔時油門大於 0.1，回到前進。
- 只認當下按著的 **R / N / G**。讀完立刻把 NWH 的 `ShiftInto` 設為 **-999**，`ShiftUp` 和 `ShiftDown` 設為 false。R 的 Input System 放開後仍會報 -1；NWH 在空檔看到這個值會立刻入倒檔，不看車速。
- **R** 鎖倒檔，放開後仍保持。車還在向前，或檔位仍是 1 至 4 時，先入空檔。停穩後才由空檔入倒檔。不要從前進檔直接跳倒檔。
- **N** 鎖空檔。
- **G** 鎖前進。不要用 **D**，D 已是右轉。車還在向後，或檔位仍是倒檔時，先入空檔。停穩後才由空檔掛 1 檔。已在 2、3、4 檔時不要拉回 1 檔。
- 倒檔一樣是 **W** 後退、**S** 煞車（`swapInputInReverse = false`）。

## 4. 操作

先點 **Game** 視窗。有手掣接上時用 Xbox 360 手掣，拔掉就回到鍵盤。同一時間只用一種。

| 按鍵    | 作用                     |
| ----- | ---------------------- |
| W     | 向前。在 N 時會入 1 檔。在 R 時向後 |
| S     | 煞車。踩下時點亮煞車燈            |
| A / D | 左右轉向                   |
| Space | 手煞車。不點煞車燈               |
| R     | 倒檔，按一次即鎖住。入倒檔後點亮白色倒車燈  |
| N     | 空檔                     |
| G     | 前進檔                    |
| E     | 熄火或再發動                 |
| Z / X | 左右方向燈                  |
| J     | 危險警告燈                  |
| L / K | 近光 / 遠光                |

手掣對應：

| 手掣 | 作用 |
| --- | --- |
| 左搖桿左右 | 轉向，和 **A / D** 同一方向 |
| **RT** | 油門 |
| **LT** | 煞車 |
| **A** | 手煞車 |
| **Y** 或方向上 | 前進檔，同 **G** |
| **B** 或方向下 | 倒檔，同 **R** |
| **X** 或方向左 | 空檔，同 **N** |
| **LB / RB** | 左 / 右方向燈 |
| **Back** | 危險警告燈 |
| **Start** | 熄火或再發動 |
| 左搖桿按下 / 右搖桿按下 | 近光 / 遠光 |

升檔時的短卡聲是換檔切動力。引擎在空檔轉到約 4400 rpm 的連續卡聲是斷油，代表檔位仍是 N，動力沒有進變速箱。

除錯面板是 `Assets/HiAceDebugHud.cs`，顯示在畫面右上角。檔位應離開 N，變速箱輸出不應長期為 0。

### 方向盤

已接在 **SampleScene** 的 `ToyotaH200`。NWH Steering 的 Steering Wheel 是 `steering_dummy`。它沒有網格，只做轉軸。盤心和輻條是子物件 `steering`，皮輪圈是 `steering.001`。兩個都要留，刪掉任何一個都會少一塊。

盤面跟前輪用同一個轉向角，沒有另外的動畫。`maximumSteerAngle` 是 **37.8°**。`steeringWheelTurnRatio = 5`，打滿時盤面再轉約 **189°**。比例用正數，**D** 向右、**A** 向左，和玩家輸入同一方向。負數會轉反。

轉軸是 `steering_dummy` 的 local forward，也就是轉向柱。轉得快慢跟前輪的轉向平滑和每秒轉角上限一樣。

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

## 7. 三面後視鏡

場景是 `scene2`。這台車靠左行、右軚。車輛 local **+Z** 是車頭，**+X** 是右側（駕駛側）。三支鏡頭都在 `CameraGroup` 下面，`CameraGroup` 掛在 `ToyotaH200`，local 是單位變換，所以下面的位置就是車輛 local，單位米。

每支鏡頭畫進自己的 Render Texture，再由 `Assets/Mirrors/` 的 `HiAce/MirrorConvex` 材質顯示。鏡面鏡頭渲染時關掉陰影，而且只在車內 `Main Camera` 是主鏡頭時才渲染。控制腳本是 `ToyotaH200` 上的 `HiAceMirrors`。

| 鏡頭 | Local Position | Local Euler | FOV | 畫面 |
| --- | --- | --- | --- | --- |
| `MirrorCameraInterior` | `(0.03, 1.70, -2.75)` | `(9, 180, 0)` | 52 | `MirrorInterior` 1024×512 |
| `MirrorCameraRight` | `(1.35, 1.70, 2.55)` | `(14, 152, 0)` | 65 | `MirrorRight` 384×256 |
| `MirrorCameraLeft` | `(-1.29, 1.70, 2.56)` | `(14, 208, 0)` | 65 | `MirrorLeft` 512×256 |

Yaw **180** 是朝車尾。右側鏡 yaw **152** 是從正後方向外約 28°，左側鏡 yaw **208** 是向左外約 28°。兩邊都向下 14°。側鏡材質 `_Convex = 0.25`。倒後鏡是平面，`_Convex = 0`。

### 鏡面貼在哪裡

- **右側鏡**：材質 `MirrorRight` 貼在右門 `door_rf` 的 `Meshpart84.006`。
- **左側鏡**：材質 `MirrorLeft` 貼在左門 `door_lf` 的 `Meshpart84.001`。
- **車內倒後鏡**：畫面是 `MirrorInterior` 這塊板，父級是鏡殼 `mmGroup165_default_0.001`，所以會跟著鏡殼。相對整輛車大約在 `(0.05, 2.10, 2.35)`，鏡面約 **0.55 m × 0.13 m**。原來的灰鏡片 `irvm` 要關掉 Renderer，否則會蓋住畫面。`MirrorLeft`、`MirrorRight` 兩塊暫用平面保持關閉。

鏡面要貼在鏡殼朝駕駛員的那一面，只比外框小一圈。不要把畫面留在放大鏡殼之前的舊座標，否則會浮在鏡殼前面。

### 倒後鏡不要被座椅擋住

倒後鏡頭不要放在擋風玻璃的鏡子上。放在那裡會拍到三張座椅，車後路面被擋住。

鏡頭放在後窗內側 `(0.03, 1.70, -2.75)`，朝車尾、向下 9°。座椅在鏡頭後面，鏡裡是車尾路面、白線和欄杆。前面的鏡面只負責顯示這張畫面。

### 車內倒後鏡左右對調

朝車尾拍出來的左右，和真實倒後鏡相反。只改 `MirrorInterior` 材質：

- `_MainTex` Scale：`(-1, 1)`
- `_MainTex` Offset：`(1, 0)`

對調之後，車右側的欄杆出現在倒後鏡的右側。左右側鏡不要做這個翻轉。

### 倒車時左側鏡

入 **R**（檔位小於 0）時，`HiAceMirrors` 把左側鏡頭再向下 **10°**，用來看到左後輪附近的路緣。回到前進檔就恢復上面的角度。這個休息角度在 `Awake` 記下，不要在 `Start` 之後才改左側鏡頭的旋轉，否則倒車復位會回到錯誤角度。

## 8. 車燈

已接在 **SampleScene** 的 `ToyotaH200`。材質在 `Assets/HiAceLights/`，Standard 自發光，平時關鍵字關閉，NWH 點亮時才開。位置是車輛 local，**+Z** 車頭、**+X** 右側，單位米。

模型沒有把近光和遠光拆開，車頭燈面 `Meshpart73`（左）、`Meshpart8`（右）只做近光。遠光是另外的聚光。方向燈和倒車燈是加在實車位置的小燈面。

| 燈組 | 燈面 | 聚光 |
| --- | --- | --- |
| 近光 `lowBeamLights` | 左 `Meshpart73`、右 `Meshpart8`，暖白 | `HiAceLowBeamSpotL` `(-0.85, 1.05, 3.25)`、`HiAceLowBeamSpotR` `(0.95, 1.05, 3.25)`。30 m，角度 52°，強度 2.2 |
| 遠光 `highBeamLights` | 無獨立燈面。按 **K** 時 NWH 同時打開近光和尾燈 | `HiAceHighBeamSpotL` `(-0.62, 1.05, 3.25)`、`HiAceHighBeamSpotR` `(0.72, 1.05, 3.25)`。65 m，角度 26°，強度 5.5，偏冷白 |
| 尾燈 `tailLights` | 左 `Meshpart36`、右 `Meshpart36.002`，暗紅。跟近光一起亮 | 無 |
| 煞車燈 `brakeLights` | 同上兩塊尾燈改為亮紅，另加車尾門頂 `boot_ok` | 無。只跟煞車踏板 **S**。手煞不點燈 |
| 倒車燈 `reverseLights` | `HiAceRevL` `(-1.08, 0.88, -3.98)`、`HiAceRevR` `(1.12, 0.88, -3.98)`，冷白。檔位小於 0 自動亮，離開倒檔就熄 | 無 |
| 左方向燈 | 車頭 `HiAceIndFL` `(-1.16, 1.08, 3.22)`、左前翼 `red.001`、車尾 `HiAceIndRL` `(-1.08, 1.22, -3.98)` | 無 |
| 右方向燈 | 車頭 `HiAceIndFR` `(1.22, 1.08, 3.22)`、右前翼 `HiAceIndicatorFenderRight`、車尾 `HiAceIndRR` `(1.12, 1.22, -3.98)` | 無 |

**Z** / **X** 切換單邊，約每秒閃 1.5 次，再按同一側熄滅。**J** 左右一起閃，再按一次熄滅。車尾方向燈和倒車燈貼在車尾外殼外側，朝車尾。